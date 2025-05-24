Iterating a tuple
=================

Suppose we have an [`std::tuple<A, B, C...>`](https://en.cppreference.com/w/cpp/utility/tuple) and a function object that can be invoked on each element of the tuple. For example:
```cpp
const auto print = [](const auto& value) { std::cout << value << ' '; };
std::tuple<int, double, std::string> tuple(1, 2.0, "three");
```

We want to implement a generic function that iterates over the tuple elements:
```cpp
template<class Tuple, class Fn>
void tuple_for_each(const Tuple& tuple, Fn fn) {
    fn(std::get<0>(tuple));
    fn(std::get<1>(tuple));
    ...
}

tuple_for_each(tuple, print);  // prints "1 2 three"
```

How can we implement this without relying on third-party libraries like `Boost` and before a function like `std::tuple_for_each` becomes part of the standard? Let's explore several different approaches. Each of them can be useful depending on specific variations of the problem.

Solution 1
----------
We can use a straightforward recursion over the index:
```cpp
template<std::size_t I, class Tuple, class Fn>
void tuple_for_each_impl(const Tuple& tuple, Fn fn) {
    fn(std::get<I>(tuple));

    constexpr std::size_t N = std::tuple_size_v<Tuple>;
    if constexpr (I + 1 < N)
        tuple_for_each_impl<I + 1>(tuple, fn);
}

template<class Tuple, class Fn>
void tuple_for_each(const Tuple& tuple, Fn fn) {
    tuple_for_each_impl<0>(tuple, fn);
}
```

This code requires C++17 due to [`if constexpr`](https://en.cppreference.com/w/cpp/language/if#Constexpr_if), but it can be easily rewritten for C++11 if you're restricted to that standard.

Solution 2
----------
We can use [`std::index_sequence`](https://en.cppreference.com/w/cpp/utility/integer_sequence) to unpack the indices indices:
```cpp
template<std::size_t... Is, class Tuple, class Fn>
void tuple_for_each_impl(std::index_sequence<Is...>, const Tuple& tuple, Fn fn) {
    (fn(std::get<Is>(tuple)), ...);
}

template<class Tuple, class Fn>
void tuple_for_each(const Tuple& tuple, Fn fn) {
    constexpr std::size_t N = std::tuple_size_v<Tuple>;
    tuple_for_each_impl(std::make_index_sequence<N>(), tuple, fn);
}
```

Here, `std::make_index_sequence<N>()` creates an (empty) object of type `std::index_sequence<0, 1, ..., N - 1>`, and the [fold expression](https://en.cppreference.com/w/cpp/language/fold) calls `fn` for each index, starting from `0`.

This version can easily be adapted to C++14 by replacing the fold expression with an array initialization trick:
```cpp
int dummy[] = {0, (fn(std::get<Is>(tuple)), 0)...};
(void)dummy;  // to silence unused variable warning
```

Solution 3
----------
A slight variation of the previous solution becomes possible in C++20:
```cpp
template<class Tuple, class Fn>
void tuple_for_each(const Tuple& tuple, Fn fn) {
    constexpr std::size_t N = std::tuple_size_v<Tuple>;
    [&]<std::size_t... Is>(std::index_sequence<Is...>){
        (fn(std::get<Is>(tuple)), ...);
    }(std::make_index_sequence<N>());
}
```

Here, we use an immediately invoked variadic lambda with an explicit template parameter list (introduced in C++20) to generate the indices.

Solution 4
----------
And finally, my favourite solution, which uses a combination of [`std::apply()`](https://en.cppreference.com/w/cpp/utility/apply), a variadic lambda, and a fold expression:
```cpp
template<class Tuple, class Fn>
void tuple_for_each(const Tuple& tuple, Fn fn) {
    const auto for_each = [&](const auto&... args) { (fn(args), ...); };
    std::apply(for_each, tuple);
}
```

In this version, `std::apply(func, tuple)` unpacks the tuple as arguments to `for_each`, i.e., calls `for_each(std::get<0>(tuple), std::get<1>(tuple), ...)`, and the fold expression turns that into a sequence of `fn` calls.
