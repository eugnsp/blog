Beware qualified names in macros
================================

Some time ago I stumbled upon a problem in our code base that I'd like to share. It's fairly trivial, but still worth mentioning - these issues can catch your fellow programmers much later, so it pays to be cautious. Below is a simplified version of the original code.

Suppose someone decides to write their own assertion macro:

```cpp
// in the global namespace

namespace detail {
    void handle_assertion(const char*, const char*, int);
}

#define MY_ASSERT(Condition)                                            \
    do {                                                                \
        if (!(Condition))                                               \
            detail::handle_assertion(#Condition, __FILE__, __LINE__);   \
    } while (false)
```

Can you spot the potential problem?

Macros are just text substitution, and you never know what context they'll be expanded in. In my case, I did this:

```cpp
namespace foo::detail {
    void bar(int i) {
        MY_ASSERT(i > 0);
    }
}
```

Boom! It fails to compile. After macro expansion, you end up with `detail::handle_assertion()` inside `bar()`, but since there's no `handle_assertion` in `foo::detail`, the compiler reports: `'handle_assertion' is not a member of 'foo::detail'`. According to the qualified name lookup rules, it won't look in the `::detail` namespace after no name was found in `foo::detail`.

The fix is to qualify the name with the global scope resolution operator:
```cpp
#define MY_ASSERT(Condition)                                            \
    do {                                                                \
        if (!(Condition))                                               \
            ::detail::handle_assertion(#Condition, __FILE__, __LINE__); \
    } while (false)
```

Now the compiler correctly finds `::detail::handle_assertion`. The real headache here is that the macro lives in some library code I can't easily change.

Some people say macros are evil. They're not when used carefully, but they do require extra responsibility to define them correctly.
