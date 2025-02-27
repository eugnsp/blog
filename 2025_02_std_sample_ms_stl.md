Unexpected `std::sample` regression in MS STL
=============================================

Recently, I've been working on migrating part of our C++ codebase from VS2017 to VS2019. Regression results hit us in an unexpected way, and I'd like to share some details, which turned out to be quite interesting. The culprit was the `<algorithm>` header.

It is well known that certain functions and classes related to random numbers are somewhat underspecified in the standard. For example, `std::mt19937` pseudo random number generator produces the same sequence of random numbers on any platform when initialized with the same seed. However, `std::uniform_int_distribution` may yeild different results even across different versions of the same compiler.

While there might be a good reason for this implementation flexibility, it makes the standard library impractical for applications requiring strict reproducibility. This issue extends beyond random distributions to other functions - specifically, to `std::sample`. And that can have some nasty consequences.

Consider the following simple piece of code:

```cpp
std::vector<int> vec_src(1000), vec_dst(10);
std::iota(vec_src.begin(), vec_src.end(), 0);

std::mt19937 gen(0);
std::sample(vec_src.begin(), vec_src.end(), vec_dst.begin(), vec_dst.size(), gen);
std::shuffle(vec_dst.begin(), vec_dst.end(), gen);
```

This code produces different values in `vec_dst` when compiled with VS2017 and VS2019. Interestingly, if you reseed `gen` between `std::sample()` and `std::shuffle()` calls, results will be consistent across both versions. Why is that?

Investigating root cause led me to the standard library souce code, where I found something surprising: not only was `std::sample` implementation changed, but the new version makes redundant calls to the random number generator - it requests random values only to discard them. According to a [commit](https://github.com/microsoft/STL/commit/99241dce6dc36713931c568d09667e5598614b1c) comment, this change was introduced during `std::ranges` implementation. Whether the implementors overlooked the issue or simply didn't care is unclear.

To explain what changed, let me show how `std::sample` is implemented in MS STL. The standard doesn't mandate a specific algorithm, but for forward iterators MS STL uses selection sampling. (Most likely, other implementations do the same.)

To fix variable names, let's state the problem: we need to select `Count` elements from the range `[First, Last)` and store them in another range starting at `Dest`, ensuring that each element is selected with equal probability. Now, let's take a look the the [implementation](https://github.com/microsoft/STL/blob/vs-2019-16.7/stl/inc/algorithm#L2773-L2787) in VS2019 16.7 (slightly simplified and reformatted for clarity):
```cpp
template<class PopIt, class SampleIt, class RngFn>
SampleIt Sample_selection(
    PopIt First, PopIt Last, std::ptrdiff_t PopSize,
    SampleIt Dest, std::ptrdiff_t Count, RngFn& RngFunc)
{
    for (; Count > 0 && First != Last; ++First, --PopSize)
    {
        if (RngFunc(PopSize) < Count)
        {
            --Count;
            *Dest = *First;
            ++Dest;
        }
    }
    return Dest;
}
```

Here, `PopSize` is initiallly equal to the size of the range `[First, Last)` and `RngFunc(X)` is wrapper around a random number generator that returns an integer in the range `[0, X)` with uniform probability. This algorithm is known as ["Algorithm S"](https://rosettacode.org/wiki/Knuth%27s_algorithm_S) in Knuth's "The Art of Computer Programming" (see Vol. 2, section 3.4.2). A good description can also be found in Bastian Rieck's [blog post](https://bastian.rieck.me/blog/2017/selection_sampling/).

The following condition is always satified: `Count <= PopSize`. If `Count` ever reaches `PopSize`, then `RngFunc(PopSize) < Count` will always evaluate to `true`, ensuring that all remaining elements are selected. The loop terminates when `Count` reaches zero - that is, when we've selected enough elements.

Now, let's compare this with the [updated implementation](https://github.com/microsoft/STL/blob/vs-2019-16.8/stl/inc/algorithm#L5263-L5277) from VS2019 16.8:
```cpp
template <class PopIt, class SampleIt, class RngFn>
SampleIt Sample_selection(
    PopIt First, std::ptrdiff_t PopSize,
    SampleIt Dest, std::ptrdiff_t Count, RngFn& RngFunc)
{
    for (; PopSize > 0; ++First, --PopSize)
    {
        if (RngFunc(PopSize) < Count)
        {
            --Count;
            *Dest = *First;
            ++Dest;
        }
    }
    return Dest;
}
```

This code is almost the same but the loop termination condition has changed: `PopSize > 0` instead of stopping when `Count` reaches zero. And here is the problem: The new condition is too weak. `Count` can reach zero _before_ `PopSize` does.

As a result, the loop continues making redundant calls to `RngFunc()`. Since `Count` is already zero, the condition `RngFunc(PopSize) < 0` will never be `true`, but `RngFunc()` is still invoked, unnecessarily consuming random numbers. This alters the state of the random number generator, affecting all subsequent uses of it. This explains the behavior observed at the start of this discussion.

Finally, let's answer the following question: How many redundant calls to `RngFunc()` are made? It is given by the following expression: `(PopSize - Count) / (Count + 1)`. Derivation is omitted for brevity.

For example, if the source range size is `PopSize = 1000` and we want to select `Count = 3` elements, there will be around 250 redundant calls, and for `Count = 100` there will be around 9. Note that each call to `RngFunc()` may translate into multiple invokations of random number generator's `operator()`, depending on the generator's result type size.
