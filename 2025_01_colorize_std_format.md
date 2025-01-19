Colorizing output with C++20’s `<format>` library
=================================================

One of the many exciting features introduced in C++20 is the new formatting library, [`<format>`](https://en.cppreference.com/w/cpp/header/format). While some good overviews of this library are available, here I want to briefly discuss two features I’ve been playing with recently.

The first feature is compile-time validation of format specification strings. For instance, the following code will fail to compile: `std::format("{} {} {}", 1, 2.0);`. This validation is enabled by another new core language feature: the [`consteval`](https://en.cppreference.com/w/cpp/language/consteval) specifier.

The first parameter of the [`std::format()`](https://en.cppreference.com/w/cpp/utility/format/format) function is not just a string but a [`std::format_string<Args...>`](https://en.cppreference.com/w/cpp/utility/format/basic_format_string), where `Args...` represents the types of the arguments to be formatted (`Args... = int, double` in the example above):
```cpp
template<class... Args>
std::string format(std::format_string<Args...> fmt, Args&&... args);
```

Simplifying slightly, `std::format_string` wraps a format specification string view and performs validation in its `consteval` constructor, throwing a compile-time error if that string doesn’t match the `Args...` pack. You can retrieve the underlying `string_view` by calling the `get()` member function. Naturally, the format specification string must itself be a compile-time value - validation at compile time is impossible for a runtime string.

The second interesting feature is the ability to define custom formatters for user-defined types. While experimenting with `<format>`, I attempted to use (or perhaps abuse - I’m still undecided) this feature to add colors using [ANSI escape sequences](https://en.wikipedia.org/wiki/ANSI_escape_code#Colors). To achieve this, we need to define a simple wrapper and [specialize](https://quuxplusone.github.io/blog/2021/10/27/dont-reopen-namespace-std/) the [`std::formatter`](https://en.cppreference.com/w/cpp/utility/format/formatter) struct for it:

```cpp
template<class Storage>
struct colorize_wrapper {
    std::string_view fmt;
    Storage args_storage;
};

template<class... Args>
auto colorize(std::format_string<Args...> fmt, Args&&... args) {
    return colorize_wrapper{fmt.get(), std::make_format_args(args...)};
}

template<class Storage>
struct std::formatter<colorize_wrapper<Storage>> {
public:
    constexpr auto parse(std::format_parse_context& ctx) {
        auto const last = std::find(ctx.begin(), ctx.end(), '}');
        my_color_code = name_to_color_code(std::string_view(ctx.begin(), last));

        return last;
    }

    auto format(colorize_wrapper<Storage> const& clr, std::format_context& ctx) const {
        auto out = std::format_to(ctx.out(), "\033[{}m", my_color_code);
        out = std::vformat_to(out, clr.fmt, clr.args_storage);
        return std::format_to(out, "\033[0m");
    }

private:
    int my_color_code;
};
```

Our custom formatter needs to implement two member functions: `parse()` and `format()`. The `parse()` function is responsible for handling the format specifier (i.e., the text inside `{}`), while the `format()` function performs the actual text formatting. The interface for using this wrapper looks like this:

```cpp
std::println("{1:green} {0:blue}", colorize("{:3d}", 10), colorize("{:3d}", 20));
```

The necessity to repeat the curly braces may feel a bit artificial, but it allows for including colorized characters without explicitly formatting them:

```cpp
std::println("{:red}", colorize("{}:{}", "text", 10)); // ":" is also displayed in red
```

That said, this is just a toy example, and the API could likely be significantly improved.

The only remaining piece is the `name_to_color_code()` function, which must be `constexpr` - just like the `parse()` member function in our formatter - because the format specification is checked at compile time!

```cpp
constexpr int name_to_color_code(std::string_view clr_name) {
    struct color { int code; std::string_view name; };
    constexpr color colors[] = {
        {30, "black"  }, {31, "red"    },
        {32, "green"  }, {33, "yellow" },
        {34, "blue"   }, {35, "magenta"},
        {36, "cyan"   }, {37, "white"  }
    };

    auto const pos = std::ranges::find(colors, clr_name, &color::name);
    if (pos != std::ranges::end(colors))
        return pos->code;
    else
        throw std::format_error("bad color name");
}
```

Note another very nice feature of the C++20’s [`<ranges>`](https://en.cppreference.com/w/cpp/ranges) library: a "projection" (`&color::name`) passed as the last argument to `std::ranges::find()` - no lambdas are needed in such trivial cases.

Here’s a reference to the complete code example: https://godbolt.org/z/qGzEex3q3

Finally, let me emphasize that caution is needed when using [`std::format_args`](https://en.cppreference.com/w/cpp/utility/format/basic_format_args) and [`std::make_format_args()`](https://en.cppreference.com/w/cpp/utility/format/make_format_args). An attempt to store `std::format_args` inside `colorize_wrapper`,

```cpp
struct colorize_wrapper {
    std::string_view fmt;
    std::format_args args;
};

template<class... Args>
auto colorize(std::format_string<Args...> fmt, Args&&... args) {
    return colorize_wrapper{fmt.get(), std::make_format_args(args...)};
}
```

will result in a dangling reference to storage created by `std::make_format_args(args...)`. This is because `std::format_args` has reference semantics and does not own the storage it refers to.
