---
title: Literal Classes as Non-type Template Parameters in C++20
date: 2020-02-12 09:13:10 -0500
categories: [Programming Languages]
tags: [cpp, cpp20]
seo:
  date_modified: 2020-05-20 13:10:43 -0400
---

With the introduction of C++20, it's now possible to provide a literal class type (i.e. `constexpr` class) instance as a template parameter.

> As a refresher, a non-type template parameter is a template parameter that does not name a type, but rather, a constant ***value*** (e.g. `template<int value>`).

## Literal Class Example

```c++
#include <iostream>

struct NullOptT {} NullOpt;

/**
 * Literal class type.
 *
 * Represents an optionally provided `int`.
 */
struct OptionalInt {
    constexpr OptionalInt(NullOptT) {}
    constexpr OptionalInt(int value): has_value(true), value(value) {}
            
    const bool has_value = false;
    const uint32_t value {};
};

/**
 * Prints whether or not a value was provided for "maybe" WITHOUT branching :)
 */
template<OptionalInt maybe>
void Print() {
    if constexpr(maybe.has_value) {
        std::cout << "Value is: " << maybe.value << std::endl;
    } else {
        std::cout << "No value." << std::endl;
    }
}

// Note: implicit conversions are at play!
int main()
{
    Print<123>();     // Prints "Value is: 123"
    Print<NullOpt>(); // Prints "No value."
}
```

In above example, the `Print` function accepts a non-type template parameter of type `OptionalInt`.

Prior to C++20, non-type template parameters were restricted to:

> - lvalue reference type (to object or to function);
> - an integral type;
> - a pointer type (to object or to function);
> - a pointer to member type (to member object or to member function);
> - an enumeration type;
> - std::nullptr_t (since c++11)

Now (C++20), this has been extended with support for:
> - an floating-point type;
> - a literal class type with the following properties: 
>   - all base classes and non-static data members are public and non-mutable and
>   - the types of all bases classes and non-static data members are structural types or (possibly multi-dimensional) array thereof.
>
> From <https://en.cppreference.com/w/cpp/language/template_parameters>

A type which satisfies one of the above definitions is considered to be a ***structural*** type.

From the above example, `OptionalInt` satisfies the definition for allowable literal class types. This custom class type allows the `Print` function to accept an *optional* integer as a template parameter. Since this value is provided as a template parameter, the implementation of `Print` can use it in constant expressions (e.g. as the condition to an `if constexpr` to conditionally compile *only* the relevant branch).

Note that `std::optional` is ***not*** considered to be a structural type, and thus cannot be used with non-type template parameters. More on this below.

# Limitations Explained
As noted above, the following limitations apply to literal class types used as non-type template parameters.

## All base classes and non-static data members are **public** and **non-mutable**.
This means that these class types can *not* have private data fields. They *can* however have private functions.

> Author commentary:
>
> I've so far been unable to find a definitive explanation for why all data members must be public. However, my guess would be that this is due to all template parameter objects of the same type and value sharing the same underlying static storage duration object (see Storage Duration below). It may otherwise be misleading to allow a literal class used as a NTTP to define private state when that private state would *necessarily* be used to determine deep member-wise equality on template instantiation. Please leave a [comment on Reddit](https://www.reddit.com/r/cpp/comments/f2s4ut/literal_classes_as_nontype_template_parameters_in/) if you have a better answer and I'll update the post, with attribution.

Regarding immutability, this means that data members cannot be marked with the [`mutable` keyword](https://en.cppreference.com/w/cpp/language/cv). Thanks to [zygoloid](https://www.reddit.com/user/zygoloid) on Reddit for this distinction.

```c++
class Literal {
public:
    constexpr Literal() {}

private:
    const int private_data {}; // Fails to compile!

    constexpr int PrivateFunc() const { return 123; } // This is allowed.
}
```

## The types of all bases classes and non-static data members are **structural types** or array thereof.
This means that literal class types must be comprised of data members that are also allowable under the same restrictions for non-type template parameters. That is to say they must *also* be structural types.

# Storage Duration
Class literal template parameter objects have *static* [storage duration](https://en.cppreference.com/w/cpp/language/storage_duration), meaning only one instance of the object exists in the program.

Further, template parameter objects of the same type with the same value **will always be the same object.**

> An identifier that names a non-type template parameter of class type T denotes a static storage duration object of type const T, called a template parameter object, whose value is that of the corresponding template argument after it has been converted to the type of the template parameter. **All such template parameters in the program of the same type with the same value denote the same template parameter object.**
> 
> From <https://en.cppreference.com/w/cpp/language/template_parameters>

# Compiler Compatibility
The examples provided here have been tested using GCC 9.2.0 (support was added in GCC 9).

Clang's C++2a implementation does not yet support class types as non-type template parameters. For reference, see the [current implementation status of C++2a features in Clang](https://clang.llvm.org/cxx_status.html#cxx20).

# Relevant Proposals
This feature was originally proposed in [P0732R2](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0732r2.pdf) and was refined by [P1907R1](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1907r1.html).
