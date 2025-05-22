Projections
===========

The C++20's [ranges library](https://en.cppreference.com/w/cpp/ranges) introduced a feature that is very useful in many contexts - projections. The idea is simple: rather than operating directly on the elements of a range, you operate on the results of applying a given function object to those elements.

Consider the following structure:
```cpp
struct entry {
    int id;
    std::string name;
};
```

Suppose you want to find an entry with a specific ID in a `std::vector` of `entry` objects. In C++17, you would typically write:
```cpp
std::vector<entry> entries;
auto pos = std::find_if(entries.begin(), entries.end(), [=](const entry& e) { return e.id == given_id; });
```

This is readable and clear, but somewhat verbose. With C++20, you can simplify it to:
```cpp
auto pos = std::ranges::find(entries, given_id, &entry::id);
```

To me, this version just looks better than the previous one that uses a lambda. Projections are applied using [`std::invoke`](https://en.cppreference.com/w/cpp/utility/functional/invoke), and its flexibility extends to projections as well. For instance, member function pointers can be used as projections.

If you're writing your own generic algorithms that operate on ranges, it's worth supporting projections. While ranges are a C++20 feature, the concept of projections can be easily implemented in earlier standards. Adding support for projections is not difficult and provides additional flexibility for users of your code.

As a concrete example, below is a simple function for formatting a range (C++17, Boost). We have a similar one in our codebase and it has turned out to be useful in a number of places thanks to projections.
```cpp
template<class F, class R, class S, class... Ps>
std::string format_and_join(const F& format, const R& range, const S& separator, Ps... projs) {
    const auto fmt = [&](const auto& el) {
        return (boost::format(format) % ... % std::invoke(projs, el)).str();
    };
    return boost::algorithm::join(range | boost::adaptors::transformed(fmt), separator);
}

std::vector<entry> entries{ {1, "A"}, {2, "B"}, {3, "C"} };
std::cout << format_and_join("%02d-%s", entries, " ~ ", &entry::id, &entry::name);  // prints: "01-A ~ 02-B ~ 03-C"
```
