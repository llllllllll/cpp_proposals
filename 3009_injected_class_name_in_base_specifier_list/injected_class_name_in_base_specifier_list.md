---
title: "Injected class name in the base specifier list"
document: P3009R0
date: 2023-10-07
audience: EWG
author:
    - name: Joe Jevnik
      email: <joejev@gmail.com>
toc: false
---

# Abstract

Inside of a class template body, the _class-name_ of the class template refers to the current template instantiation; this is referred to as the _injected-class-name_.
The _injected-class-name_ is not present until the class body begins, which is after the opening brace of the class definition.
The consequence of this is that the _injected-class-name_ is not available in the _base-specifier-list_, which means it is not accessible when using the curiously recurring template pattern (CRTP).
While [@P0847R7] has removed the need for using CRTP in many places, there still exist cases that must be expressed using this pattern.
This paper proposes allowing the use of the _injected-class-name_ in the _base-specifier-list_ to simplify code using CRTP.
This paper also aims to reduce complexity in the language by simplifying the interpretation of the _class-name_ and the rules around the _injected-class-name_.

# Introduction

Using the _injected-class-name_ when implementing class templates is not only easier to read, it is also less bug prone.
Many complicated container types use multiple template arguments, often with rarely changed default arguments.
For example, a mapping type have may some `Key` type and a `Value` type in addition to `Hash`, `KeyEqual`, and `Allocator`.
The last three arguments are often defaulted, as it is for the standard library's `std::unordered_map`, which means people can easily forget that they exist.
Using the _injected-class-name_ to refer to the current instantiation makes it impossible to forget to include all of these extra template arguments.
When the _injected-class-name_ is not used, it is possible to write code that works for common cases, but produces difficult to understand compiler errors for some template instantiations.
For example, consider the following template class:

```c++
template <typename Derived>
struct CRTPBase {
    // ...
};

template <typename Key,
          typename Value,
          typename KeyEqual = std::equal<Key>,
          typename Hash = std::hash<Key>
          typename Allocator = std::allocator<std::pair<Key const, Value>>>
class Container : CRTPBase<Container<Key, Value KeyEqual, Hash>> {
    // ...
};
```

In this case, we likely wanted to pass the current instantiation of `Container` as the template argument to `CRTPBase`; however, we accidentally forgot the `Allocator` argument.
`Allocator` will likely often use the default type, which means that most of the time this code will actually behave as intended.
The bug will only be revealed when a non-default allocator is used, and it will likely yield a complicated error message.

# Proposal

This paper proposes introducing the _injected-class-name_ in the _base-specifier-list_.

## Behavior Changes

Unfortunately, while the proposal itself is concise, it can alter the behavior of existing code.

## Unambiguous to ambiguous

The following code is currently unambiguous according to the rules of C++: the base type will always be `WasTemplate`:

```c++
#include <type_traits>

struct WasType {};
struct WasTemplate {};

template <typename Type>
auto foo() -> WasType;  // overload 1

template <template <typename...> class Template>
auto foo() -> WasTemplate;  // overload 2

template <typename Type>
struct CurrentlyUnambiguousBase : decltype(foo<CurrentlyUnambiguousBase>()) {};

static_assert(std::is_base_of_v<WasTemplate, CurrentlyUnambiguousBase<void>>);
```

In this example, we have a function template `foo` which is overloaded once for a type template argument, and once for a template template argument.
In the _base-specifier-list_ the _class-name_ `CurrentlyUnambiguousBase` will always refer to the class template and never refer to an instantiation of the class template.
Because the name unambiguously refers to the class template, the compiler chooses overload 2 for `foo`.

Inside the class body, the _class-name_ can refer to the current template instantiation being defined, which means that `CurrentlyUnambiguousBase` may be the type `CurrentlyUnambiguousBase<Type>` or the class template `CurrentlyUnambiguousBase`.
This means that moving the exact same expression from the _base-specifier-list_ inside the class body results in an ambiguous overload.

```c++
template <typename Type>
struct CurrentlyUnambiguousBase : decltype(foo<CurrentlyUnambiguousBase>()) {
    // Error: call to overloaded `foo` is ambiguous between overloads 1 and 2
    using InsideBody = decltype(foo<CurrentlyUnambiguousBase>());
};
```

If the _injected-class-name_ was introduced in the _base-specifier-list_, then this previously unambiguous code would become an error and force the user to disambiguate.

## Same code with different behavior.

Changing well formed code into ill-formed code is not ideal, but users will mostly be able to mechanically change the few occurrences and move on.
Changing the meaning of code in such a way that both the old and the new are well formed with a different result is much more serious.

Consider the following code:

```c++
#include <type_traits>

struct WasType {};
struct WasTemplate {};

template <typename Type>
auto bar(int) -> WasType;  // overload 1

template <template <typename...> class>
auto bar(long) -> WasTemplate;  // overload 2

template <typename Type>
struct DifferentBehavior : decltype(bar<DifferentBehavior>(0)) {
    using InsideBody = decltype(bar<DifferentBehavior>(0));
};

static_assert(std::is_base_of_v<WasTemplate, DifferentBehavior<void>>);
static_assert(std::is_same_v<WasType, DifferentBehavior<void>::InsideBody>);
```

This code uses an overloaded function template: `bar`.
`bar` has two overloads: one which has a type template parameter with an `int` parameter and a second which takes a template template parameter and a `long` parameter.
In the _base-specifier-list_, where `DifferentBehavior` unambiguously refers to the class template, `bar<DifferentBehavior>(0)` only considers overload 2 and allows an implicit conversion from `0` (which is of type `int`) to `long`.
Inside the class body where `DifferentBehavior` can refer to either the class template or the current instantiation depending on the context, both overloads 1 and 2 are considered.
Overload 1 takes exactly an `int`, which makes the function an exact match.
If the _injected-class-name_ was introduced in the _base-specifier-list_, then this code would begin to fail the first `static_assert`, and instead the base class would become `WasType`.

## Proposed Wording

In [basic.scope.pdecl]{.sref}/8:

The locus of an _injected-class-name_ declaration ([class.pre]{.sref}) is immediately following the [opening brace]{.rm}[_class-head-name_]{.add} of the class definition.

# Prior Work

[@CWG432] was reported in 2003 to ask about this exact feature.
At the time the author believed that not introducing the _injected-class-name_ in the _base-specifier-list_ was accidental, and should be corrected with a defect report.
This defect report did not go anywhere as far as I can tell.
Given that this feature would change the meaning of existing code, I believe it is more appropriate to treat this as a feature with its own paper.

# Conclusion

Allowing the _injected-class-name_ in the _base-specifier-list_ will make existing template metaprogramming more concise, easier to read, and less bug-prone.
While some code may change behavior, it is the author's opinion that the code that will change behavior is already ambiguous to readers, even if it is not ambiguous in the standard.
The code that changes from working to not-working will only require the author to clarify their intent.
The code that changes meaning is, in the author's opinion, contrived and unclear in intent.
The fact that the code already does something different if written one line later inside the body of the class means that most readers of the code will be unaware that the result is different.
Picking one consistent interpretation for this expression will make it easier for people to understand and build a mental model for the rules of the language.

# Acknowledgments

At C++Now 2023 at the first language feature in a week session someone shouted this idea out as a good proposal.
Unfortunately I did not see who it was but thank you to the anonymous C++Now attendee who proposed working on this idea.
Thank you to Barry Revzin and Steven Watanabe who helped find code that would change behavior given this change.
