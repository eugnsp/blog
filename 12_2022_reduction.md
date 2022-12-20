On the solution of one functional problem
=========================================

Definitions
-----------

For the sake of this post let's assume the following narrow definitions: a _function_ is a map from scalars into scalars, a _functional_ is a map from functions into scalars. Below are trivial examples of functions and functionals, which I'll use below for illustration purposes:
```cpp
// Function:
double sq(double x) {
    return x * x;
}

// Functionals:
auto at = [](auto fn) {
    return fn(10);
};

auto integrator = [](auto fn) {
    double sum = 0;
    const int n = 10;
    for (int i = 0; i < n; ++i)
        sum += fn(i);
    return sum / n;
};
```

The first functional, `at`, returns the value of a function at the fixed point `10`. The second one, `integrator`, computes an approximation of an integral over a finite range `[0, 10]`,
$$\int_0^{10} dx \ f(x) \approx \sum_{i = 0}^{9} f(i).$$

An application of a these functionals to the function `sq` gives the following results: `at(sq) = 10`, `integrator(sq) == 385`.

Generalization
--------------

We can generalize these definitions to `n`-ary functions in a straightforward way. Say, we have a binary function $f(x, y)$. We can fix the first argument $x$ and obtain a parameterized unary function $f_x(y) = f(x, y)$. If we apply a functional to that function, we get a scalar that depends on $x$, i.e. a unary function:

```cpp
template<class Fnl, class BinaryFn>
auto apply_1(Fnl fnl, BinaryFn fn) {
    return [=](double x) {  // Be careful: Don't capture fnl and fn by reference!
        return fnl([=](double y) {
            return fn(x, y); 
        });
    };
}
```

For example, for a function
```
auto fn = [](double x, double y) { return std::pow(y, x); };
```
`apply_1(at, fn)` will return a function object that computes a function `std::pow(10, x)` and `apply_1(integrator, fn)` will return a function object that approximates $$\int_0^{10} dy \ y^x = \frac{10^{x + 1}}{x + 1}.$$

Obviously, in the general case the order of `x` and `y` does matter. For example, if we fix `y` in the definition of `apply_1`, for `apply_1(integrator, fn)` we'll get $$\int_0^{10} dx \ y^x = \frac{y^{10} - 1}{\ln(y)},$$ which is a totally different function. Belowe I'll assume that the first argument is the one that is fixed.

Now, we can apply a functional the secod time to get a scalar:
```cpp
template<class Fnl, class BinaryFn>
double apply_2(Fnl fnl, BinaryFn fn) {
   return fnl(apply_1(fnl, fn));
}
```

For `integrator`, `apply_2(integrator, fn)` approximates a double integral $$\int_0^{10} dx \ \int_0^{10} dy \ y^x.$$

Generalization is obvious: by applying a functional to an `n`-ary function `m` times we obtain an `(n-m)`-ary function. In particular, if we apply it `n` times, we get a scalar. 

Problem statement
-----------------

Now the problem statement: write a generic function that takes an `n`-ary function and a functional and applies it `n` times. Note that we want a generic function: the value of `n` should be deduced.

The signature of such a generic function might look like this:
```cpp
template<class Fnl, class Fn>
double apply(Fnl fnl, Fn fn);
```

For example: `apply(integrator, [](double x, double y, double z){ return x * y * z; })` should return an approximate value for a triple integral $$\int_0^{10} dx \ \int_0^{10} dy \ \int_0^{10} dz \ x y z.$$

Solution
--------

How could we implement such a generic function? From the examples given above, one possible solution that comes to mind is recursion: chomp one argument at a time using a lambda expression, apply a functional and recurse until a unary function is finally passed as an argument.

With C++14 generic variadic lambdas, the recursion itself looks rather straightforward:
```cpp
template<class Fnl, class Fn>
int apply(Fnl fnl, Fn fn) {
    if constexpr (/* Fn is a unary function */)
        return fnl(fn);
    else
        return fnl([=](double x) {
            return apply(fnl, [=](auto... xs) {
                return fn(x, xs...);
            });
        });
}
```

How could we detect a unary function? The standard library has a special type trait to do it: `std::is_invocable_v<Fn, double>` returns `true` if `Fn` can be invoked on a single `double` argument.

Let's try it:
```cpp
template<class Fnl, class Fn>
int apply(Fnl fnl, Fn fn) {
    if constexpr (std::is_invocable_v<Fn, double>)
        return fnl(fn);
    else
        return fnl([=](double x) {
            return apply(fnl, [=](auto... xs) {
                return fn(x, xs...);
            });
        });
}
```

This implementation works with unary and binary functions, but breaks for `n >= 3` with the following error (clang): 
```none
error: no matching function for call to object of type 'const (lambda at ...)'
return fn(x, xs...);
         ^~
note: in instantiation of variable template specialization 'std::is_invocable_v<(lambda at ...), double>' requested here
    if constexpr (std::is_invocable_v<Fn, double>)
                  ^
```

For some reason, `std::is_invocable_v<Fn, double>` doesn't work on a generic variadic lambda. To understand why, consider the following example:
```cpp
double fn(double x);

auto lambda1 = [](double x)   { return fn(x); };
auto lambda2 = [](auto... xs) { return fn(xs...); };
```

Loosely speaking, both lambdas do the same thing: they forward an argument to a unary function `fn(x)`. For the idea presented above to work properly, both lambdas should be invocable only on one `double`. Let's check if this is the case:
```cpp
std::cout << std::is_invocable_v<decltype(lambda1), double>;
// prints 1 as expected
std::cout << std::is_invocable_v<decltype(lambda1), double, double>;
// prints 0 as expected

std::cout << std::is_invocable_v<decltype(lambda2), double>;
// prints 1 as expected
std::cout << std::is_invocable_v<decltype(lambda2), double, double>;
// doesn't compile - why?
```

The last line doesn't compile for the following reason: `std::is_invocable` type trait examines only the function signature - if an error happens in the function body, it's a hard error. But a function with the signature `return_type(auto...)` can be called with any number of arguments!

This is similar to hard/soft errors [SFINAE](https://github.com/eugnsp/library/blob/master/cpp/templates.md#sfinae). Consider the following two (simplified) implementations of `std::enable_if`:
```cpp
template<bool B>
struct enable_if_1 {};

template<>
struct enable_if_1<true> {
    using type = void;
};

template<bool B>
struct enable_if_2 {
    using type = typename enable_if_1<B>::type;
};
```

The first implementation is what you'll most likely see in a standard library source code. The second one looks similar: if `B` is `true`, it defines a member type `type`, if `B` is `false` a substitution error will be generated. However, if you try to use the second implementation, you'll get the following compilation error:
```none
error: no type named 'type' in 'struct enable_if_1<false>'
using type = typename enable_if_1<B>::type;
      ^~~~
```

SFINAE works only in what is called an [_immediate context_](https://stackoverflow.com/q/15260685): a substitution failure inside `enable_if_2` _is_ an error.

To fix `lambda2`, we should encode invocability of `fn` into lambda's signature. A trivial way to do it is to put (unevaluated) `fn` call into the explicit return type specification:
```cpp
auto lambda2 = [](auto... xs) -> decltype(fn(xs...)) { return fn(xs...); };

std::cout << std::is_invocable_v<decltype(lambda2), double>;
// prints 1 as expected
std::cout << std::is_invocable_v<decltype(lambda2), double, double>;
// prints 0 as expected
```

Combining everything together, we get the final solution:
```cpp
template<class Fnl, class Fn>
int apply(Fnl fnl, Fn fn) {
    if constexpr (std::is_invocable_v<Fn, double>)
        return fnl(fn);
    else
        return fnl([=](double x) {
            return apply(fnl, [=](auto... xs) -> decltype(fn(x, xs...)) {
                return fn(x, xs...);
            });
        });
}
```

In conclusion, let me thank StackOverflow user *dkaramit* for their [question](https://stackoverflow.com/q/71905195) that led to writing this post.
