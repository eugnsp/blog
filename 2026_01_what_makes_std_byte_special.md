What makes `std::byte` special
==============================

Every C++ programmer is familiar with the strict aliasing rule. You cannot access an object through a pointer or reference to an unrelated type even if that type has the same size and alignment. According to the standard, if you want to access the raw bytes of an object, you can only do so using `char`, `unsigned char`, or `std::byte` types. This is stated in the paragraph [[basic.lval/11]](https://eel.is/c++draft/basic.lval#11) of the standard.

`std::byte` was added to this list when it was introduced in C++17. I asked myself: how does a compiler (say, GCC) know that `std::byte` is special? One could imagine that fundamental character types are treated in a special way, but `std::byte`, even inside the `std` namespace, is [defined](https://github.com/gcc-mirror/gcc/blob/releases/gcc-15/libstdc%2B%2B-v3/include/c_global/cstddef#L75) as a simple scoped enumeration type:

```cpp
namespace std {
    enum class byte : unsigned char {};
}
```

No compiler-specific attributes, no visible magic. So how does it work?

Digging into the GCC source code, in the file `decl.cc` we find a function called [`start_enum`](https://github.com/gcc-mirror/gcc/blob/releases/gcc-15/gcc/cp/decl.cc#L17574). According to the comment, it "begins compiling the definition of an enumeration type". In the body of that function, there is a short, [explicit check](https://github.com/gcc-mirror/gcc/blob/releases/gcc-15/gcc/cp/decl.cc#L17683-L17686):

```cpp
/* std::byte aliases anything.  */
if (TYPE_CONTEXT(enumtype) == std_node && !strcmp("byte", TYPE_NAME_STRING(enumtype)))
    TYPE_ALIAS_SET(enumtype) = 0;
```

The logic is simple: if the enum is declared in the `std` namespace and its name is "byte", it is allowed to alias any other type. Clang does something very similar.

One final note. Unlike in C, `signed char` is formally not included in the list of those alias-anything types. However, both GCC and Clang treat that type in the same way as other character types. Clang has a comment with a neat reasoning:
>For now, the risk of exploiting this detail in C++ seems likely to outweigh the benefit.

Don't forget the strict aliasing rule. It can bite you when you least expect it, even if your code seems to work fine today.
