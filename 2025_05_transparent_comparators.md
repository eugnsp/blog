Transparent comparators
=======================

C++14 introduced a useful feature called transparent comparators. By replacing a non-transparent comparator with a transparent one, you can often make your code potentially more efficient. Unfortunately, even after a decade, this feature remains relatively unknown, so let me briefly describe one typical use case.

Consider the following code:
```cpp
std::set<std::string> set;
auto contains = [&](std::string_view sv) { return set.contains(sv); };
```

The intention here is clear: check whether the set contains a given string. Unfortunately, this code won't compile - you'll be forced to construct a `std::string` from the `std::string_view`:
```cpp
auto contains = [&](std::string_view sv) { return set.contains(std::string(sv)); };
```

This conversion is redundant and may involve a heap allocation, impacting performance. But why is it necessary? The issue lies in the default comparator, which is `std::less<Key>`. In only provides
```cpp
constexpr bool operator(const Key&, const Key&) const;
```
Since you can't implicitly construct `std::string` from `std::string_view` (for good reason!), an explicit conversion is required.

This is where C++14's transparent comparators help. The standard library now provides a specialization `std::less<void>`, which you can also write as `std::less<>` (`void` is defaulted). The comparator's `operator()` becomes a function template:
```cpp
template<class T, class U>
constexpr bool operator()(T&& lhs, U&& rhs) const;  // simplified
```

Additionally, `std::set::contains()` has a generic overload that participates in overload resolution if the comparator is transparent:
```cpp
template<class K>
bool contains(const K&);
```

With this, you can write:
```cpp
std::set<std::string, std::less<>> set;
auto contains = [&](std::string_view sv) { return set.contains(sv); };
```

This version compiles just fine and the call to `.contains()` with a `std::string_view` now works without requiring a temporary `std::string`. Similar benefits apply to other member functions like `find()` and `count()`, as well as other containers.

An even more interesting case arises when `std::string_view` is replaced with a C-style string `const char*`. Since `std::string` can be implicitly constructed from a C-string, both versions of `std::set` (with and without a transparent comparator) will compile and work. However, with a transparent comparator no temporary `std::string` is constructed.

Here are some simple microbenchmark results for calling `contains()` with a C-string of `N` random characters on a `std::set` containing 1000 strings of `N` random characters each (compiled with GCC 14.2, -O2):
```none
-----------------------------------------------------------------------------------------
Benchmark                                               Time             CPU   Iterations
-----------------------------------------------------------------------------------------
f<std::set<std::string>>/4                        35.0 ns         35.0 ns     20066464
f<std::set<std::string>>/8                        38.1 ns         38.1 ns     18331535
f<std::set<std::string>>/64                       47.7 ns         47.7 ns     14698163
f<std::set<std::string>>/512                      51.1 ns         51.1 ns     14173181
f<std::set<std::string>>/4096                     95.0 ns         95.0 ns      7034922
f<std::set<std::string>>/16384                     206 ns          206 ns      3415660
f<std::set<std::string, std::less<>>>/4           28.8 ns         28.8 ns     24241323
f<std::set<std::string, std::less<>>>/8           31.7 ns         31.7 ns     22249004
f<std::set<std::string, std::less<>>>/64          34.3 ns         34.3 ns     21223841
f<std::set<std::string, std::less<>>>/512         32.1 ns         32.1 ns     22007889
f<std::set<std::string, std::less<>>>/4096        55.4 ns         55.4 ns     14197551
f<std::set<std::string, std::less<>>>/16384        102 ns          102 ns      6931108
```

Lastly, it's important to note that `std::set<std::string>` and `std::set<std::string, std::less<>>` are distinct types. This may cause issues in existing codebases. For example, when an existing function expects a `std::set<std::string>` but you try to pass a `std::set<std::string, std::less<>>`.
