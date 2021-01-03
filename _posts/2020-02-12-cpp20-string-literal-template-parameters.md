---
title: Passing String Literals as Template Parameters in C++20
date: 2020-02-12 16:45:04 -0500
categories: [Programming Languages]
tags: [cpp, cpp20]
seo:
  date_modified: 2020-05-20 13:10:43 -0400
---

In a [recent post](/posts/cpp20-class-as-non-type-template-param/), I described a C++20 feature that allows class literals to be used as non-type template parameters. The original feature proposal mentions how this could [enable constant expression strings to be passed as template parameters as well (see P0732R2, section 3.2)](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0732r2.pdf).

This would work by wrapping the constant expression string in a *structural* class literal, which would store its characters in a fixed-length array (e.g. `char[N]`).

The following sample demonstrates this by introducing a structural class literal wrapper type
`StringLiteral`, which is constructed from a char array.

Note that the syntax `Print<"literal">` is a syntactic sugar. The string is not being passed *directly* to `Print`, but rather undergoes an implicit conversion to our class literal `StringLiteral`. Note that this conversion depends on C++17's added support of template parameter deduction using constructor arguments (the value of `N` can be deduced from constructor argument `str`).

```c++
#include <iostream>
#include <algorithm>

/**
 * Literal class type that wraps a constant expression string.
 *
 * Uses implicit conversion to allow templates to *seemingly* accept constant strings.
 */
template<size_t N>
struct StringLiteral {
    constexpr StringLiteral(const char (&str)[N]) {
        std::copy_n(str, N, value);
    }
    
    char value[N];
};

template<StringLiteral lit>
void Print() {
    // The size of the string is available as a constant expression.
    constexpr auto size = sizeof(lit.value);

    // and so is the string's content.
    constexpr auto contents = lit.value;

    std::cout << "Size: " << size << ", Contents: " << contents << std::endl;
}

int main()
{
    Print<"literal string">(); // Prints "Size: 15, Contents: literal string"
}
```
