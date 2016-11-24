# A proposal to add non-owning indirection class templates `indirect` and `optional_indirect`

_Joseph Thomson \<joseph.thomson@gmail.com\>_

## Introduction

`indirect<T>` and `optional_indirect<T>` are high-level indirection types that reference objects of type `T` without implying ownership. An `indirect<T>` is an indirect reference to an object of type `T`, while an `optional_indirect<T>` is an _optional_ indirect reference to an object of type `T`. Both `indirect` and `optional_indirect` have reference-like construction semantics and pointer-like assignment and indirection semantics.

## Table of contents

* [Motivation and scope](#motivation)
* [Impact on the standard](#impact)
* [Design decisions](#design)
  * [Conversion from `T&`](#design/conversion/from-lvalue-ref)
    * [Conversion from `T&&`](#design/conversion/from-rvalue-ref)
  * [Conversion to `T&`](#design/conversion/to-lvalue-ref)
  * [Conversion from `T*`](#design/conversion/from-ptr)
  * [Conversion to `T*`](#design/conversion/to-ptr)
  * [Assignment operators and the `{}` idiom](#design/assignment)
  * [Optional API functions](#design/optional)
  * [Comparison operations](#design/comparison)
  * [Move behaviour](#design/move)
  * [Relationships and immutability](#design/immutability)
  * [Const-propagation](#design/const-propagation)
  * [The `make` free functions](#design/make-functions)
  * [The `cast` free functions](#design/cast-functions)
  * [The `get_pointer` free functions](#design/get_pointer-functions)
  * [Compatibility with `propagate_const`](#design/propagate-const)
  * [Use of inheritance by delegation ("`operator.` overloading")](#design/operator-dot)
  * [Why not `T*`?](#design/why-not-ptr)
  * [Why not `T&`?](#design/why-not-ref)
  * [Why not `optional<T&>`?](#design/why-not-optional-ref)
  * [Why not `optional<indirect<T>>`?](#design/why-not-optional-indirect)
  * [Why not `observer_ptr<T>`?](#design/why-not-observer_ptr)
  * [Why not `reference_wrapper<T>`?](#design/why-not-reference_wrapper)
  * [Why not `not_null<T*>`?](#design/why-not-not_null)
  * [Naming](#design/naming)
* [Technical specifications](#technincal)
* [Acknowledgements](#acknoledgements)
* [References](#references)

## <a name="motivation"></a>Motivation and scope

Modern C++ [guidelines](https://github.com/isocpp/CppCoreGuidelines) recommend against using pointers to manage dynamically allocated resources, arrays, strings and optional values, instead encouraging the use of higher-level abstractions such as [`std::unique_ptr`](http://en.cppreference.com/w/cpp/memory/unique_ptr), [`std::shared_ptr`](http://en.cppreference.com/w/cpp/memory/shared_ptr), [`std::vector`](http://en.cppreference.com/w/cpp/container/vector), [`std::array`](http://en.cppreference.com/w/cpp/container/array), [`std::array_view`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n3851.pdf) _(not yet standardized)_, [`std::string`](http://en.cppreference.com/w/cpp/string/basic_string), [`std::string_view`](http://en.cppreference.com/w/cpp/string/basic_string_view) and [`std::optional`](http://en.cppreference.com/w/cpp/utility/optional). These types provide:

* Compile-time safety
* Run-time safety
* Documentation of intent
* Modern, easy-to-use interfaces
* Resource management

However, no such high-level types are provided by the standard to act as non-owning references to other objects; thus, pointers are still widely used in modern C++ code for this purpose, despite presenting many of the same disadvantages that led to their replacement in other use cases. This is the role played by `indirect` and `optional_indirect`:

|          | Owned                                      | Non-Owned                      |
|----------|--------------------------------------------|--------------------------------|
| Single   | `unique_ptr` `shared_ptr` `optional`       | `indirect` `optional_indirect` |
| Array    | `array` `vector` `unique_ptr` `shared_ptr` | `array_view`                   |
| String   | `string`                                   | `string_view`                  |
| Iterator | —                                          | _assorted_                     |

### Current demand

A number of existing and proposed high-level types attempt to address similar problems, but none of them adequately fit the use cases for which `indirect` and `optional_indirect` are designed. Their existence, however, shows that demand exists for solutions to the problems addressed by `indirect` and `optional_indirect`. Discussions of these other types can be found in the [design section](#design) of this proposal:

* [Why not `optional<T&>`?](#design/why-not-optional-ref)
* [Why not `reference_wrapper<T>`?](#design/why-not-reference_wrapper)
* [Why not `observer_ptr<T>`?](#design/why-not-observer_ptr)
* [Why not `not_null<T*>`?](#design/why-not-not_null)

## <a name="impact"></a>Impact on the standard

This proposal adds a single header, `<indirect>` containing both `indirect` and `optional_indirect`, which depends on the `<optional>` header for `nullopt_t`, `nullopt` and `bad_optional_access`. The `<optional>` header and all other library components will be unaffected.

The proposal could be modified to separate the definitions of `indirect` and `optional_indirect` into two headers, `<indirect>` and `<optional_indirect>` respectively, which would remove the dependency of `<indirect>` on `<optional>`. Alternatively, the `optional_indirect` class could instead be added to the `<optional>` header.

The proposed additions can be implemented using C++11 library and language features only.

## <a name="design"></a>Design decisions

When design decisions are discussed here with reference to `indirect` only, assume that the discussion applies equally to `optional_indirect` unless otherwise stated. 

### <a name="design/conversions"></a>Type conversions

When deciding which type conversions to enable for `indirect`, and whether specific conversions should be _explicit_ or _implicit_, we considered both the logical correctness of a potential conversion (i.e. whether it makes sense) and its potential impact on the correctness of client code (i.e. whether it is safe). We consider a conversion to be _logically correct_ if the object before the conversion represents the _same kind of thing_ as the object after the conversion. Logical correctness should ideally be enforced by the type system, and explicit conversions often represent some violation of this enforcement; therefore, a conversion should be explicit unless it is nearly always logically correct. We consider a conversion to be _unsafe_ if it has behaviour which, if not actively accounted for, could result in programming errors in client code. Implicit conversions can easily happen without the programmer realizing, so it is imperative that they be safe.

In the case of `indirect`, we have to consider type conversions to and from the two fundamental C++ reference types, references (`T&`) and pointers (`T*`).

`indirect<T>` _always_ represents a non-owning reference to an object of type `T`, as does `T&`. Since they always represent the same kind of thing, both conversion from `T&` to `indirect<T>` and conversion from `indirect<T>` to `T&` are always logically correct. In addition, these conversions are perfectly safe, since every state of `indirect<T>` maps exactly onto every state of `T&`, and vice-versa. There are no exceptional cases for the programmer to consider. This means that implicit conversion to and from `T&` _should_ be considered.

Conversely, while `T*` _may_ represent a non-owning reference to an object of type `T`, it can also represent a number of other things:

* `T*` could represent an array
* `T*` could represent an iterator
* `T*` could represent a dynamically allocated resource to be managed

Case in point, none of the following conversions would make logical sense:

```c++
int arr[] = { 1, 2, 3 };

indirect<int> a = arr; // this wouldn't work with a `std::array`
indirect<int> b = end(arr); // doesn't point to a valid object
indirect<int> c = new int(); // resource leak
```

In addition, `T*` has an additional state that does not map onto any valid state of `indirect<T>`: the null pointer state. Therefore, when converting from a `T*` to `indirect<T>`, we can reasonably do one of two things when faced with a null pointer:

1. Throw an exception
2. Specify undefined behaviour

Both of these behaviours require the programmer to actively account for the exceptional case (by either catching the exception or by preemptively checking for a null pointer), so implicit conversion from `T*` to `indirect<T>` is out of the question. And while conversion from `indirect<T>` to `T*` _is_ safe, it is still not logically correct, so it shouldn't be implicit either; the same logic applies to conversion from `T*` to `optional_indirect<T>` which while safe (the null pointer state can map onto the empty state), is still not logically correct.

In summary, we consider enabling a particular conversion (explicit _or_ implicit) if it is likely to be logically correct _some_ of the time. We consider making a conversion implicit if it is likely to be logically correct in the _vast majority_ of cases _and_ is reasonably safe. This means that implicit conversion to and from `T&` should be considered, while conversion to and from `T*` should only be explicit. Of course, there are other things to take into account when deciding whether or not to actually enable a conversion. We discuss the specifics of particular cases in the following sections.

#### <a name="design/conversion/from-lvalue-ref"></a>Conversion from `T&`

`T&` is implicitly convertible to `indirect<T>`, so the following is valid:

```c++
int i = {};
int j = {};
indirect<int> ii = i; // `ii` references `i`
ii = j; // `ii` now references `j`
```

As detailed earlier, conversion from `T&` to `indirect<T>` is both logically correct and safe. However, given that `indirect<T>` in many ways behaves like `T*`, and `T&` does not implicitly convert to `T*`, this behaviour may raise a few eyebrows.

One concern is that allowing conversion from `T&` and then providing access via the indirection operators (`*` and `->`) is asymmetrical and unlike anything currently in the standard library, and that we should avoid this combination because it goes against what users have come to expect from a pointer-like type. However, such behaviour is already present in the standard library type `optional`:

```c++
int i = {};
optional<int> oi = i;
int j = *oi;
```

If users can get used to using "pointer-like" indirection operators in a value-like type, then we propose that they can get used to "reference-like" construction from `T&` in a pointer-like type.

Another concern is that "rebinding" on reassignment is surprising behaviour; if `indirect<T>` is initiailized like a reference, users might expect assignment from `T&` to assign to the underlying object like a reference. However, rebinding on assignment is already the behaviour exhibited by `reference_wrapper`:

```c++
int i = {};
int j = {};
reference_wrapper<int> ri = i; // `ri` references `i`
ri = j; // `ri` now references `j`
```

This behaviour is arguably _more_ surprising with `reference_wrapper` given its otherwise reference-like semantics. It could be argued that a pointer-like type such as `indirect` would be expected to rebind on assignment, while a reference-like type such as `reference_wrapper` could reasonably be expected to assign to the underlying object. Thus, we feel confident that users will be comfortable with the rebinding semantics of `indirect`.

##### <a name="design/conversion/from-rvalue-ref"></a>Conversion from `T&&`

While conversion from `T&` to `indirect<T>` is always logically correct, it is not necessarily semantically correct in the wider context. Consider this:

```c++
indirect<int const> a = 42; // equivalent to `indirect<int const> a = int{42}`
```

When the constructor of `a` returns, the temporary `int{42}` is destroyed and `a` is left _dangling_, pointing to a non-existent object. This is not a problem with `T const&` because of a C++ rule which means that the lifetime of a temporary object may be extended by binding to a const lvalue reference:

```c++
int const& a = 42; // temporary `int{42}` destroyed when `a` goes out of scope
```

Obviously, dangling references are undesirable, as they can lead to undefined behaviour. One option would be to explicitly disable construction from `T&&`; however, this would preclude passing temporary objects to functions taking `indirect<T const>` parameters:

```c++
auto f = [](indirect<foo const> x) { … }
f(make_foo()); // error: constructor `indirect<foo const>(foo&&)` is deleted 
```

This might not seem a big deal, since you could just use `T const&` instead and use `indirect<T const>` internal to your function if need be, but its impact on the usability of `optional_indirect<T const>` is far greater:

```c++
auto f = [](optional_indirect<foo const> x) { … }
foo tmp = make_foo();
f(optional_indirect<foo const>(tmp));
```

Not only must a temporary object be explicitly created, the `optional_indirect` must also be explicitly constructed, since `T*` is [not implicitly convertible](#design/conversion/from-ptr) to `optional_indirect<T>`. Compare this to the syntax required if `f` instead takes a pointer:

```c++
auto f = [](foo const* x) { … }
foo tmp = make_foo();
f(&tmp);
```

Then compare this again to the syntax required if `optional_indirect<T>` did not disable construction from `T&&`:

```c++
auto f = [](optional_indirect<foo const> x) { … }
f(make_foo());
```

If we disable construction from `T&&`, then `T*` becomes easier to use than `optional_indirect<T>`. If we do not disable construction from `T&&`, then `optional_indirect<T>` becomes easier to use than `T*`. Since use of `optional_indirect<T>` encourages more logically correct code than does use of `T*`, we want to _encourage_ instead of discourage use of `indirect`. Therefore, in balance, we have decided against disabling construction from `T&&`. Users will just have to be aware that lifetime extension of temporary objects does not occur with `indirect<T const>` as it does with `T const&`. We also note that the designers of `basic_string_view` have not disabled construction from rvalue references; though we cannot be sure of their reasoning, it is likely to have been along similar lines.

#### <a name="design/conversion/from-ptr"></a>Conversion from `T*`

`indirect<T>` explicitly constructs from `T*`, throwing an `invalid_argument` exception if passed a null pointer:

```c++
int* i = get_some_pointer();

try {
    auto ii = indirect<int>(i);
} catch (invalid_argument& ex) {
    …
}
```

Since the conversion from `T*` to `indirect<T>` is not always logically correct and cannot be implemented safely, we decided that the conversion must be explicitly specified.

When considering how to handle the null pointer case, we had two choices: throw an exception or specify undefined behaviour. Throwing an exception is preferable from a safety perspective because even uncaught exceptions can be handled by a `terminate_handler`, allowing the program to shut down or recover from the error in a controlled manner; with undefined behaviour, however, the most you can do is cross your fingers and hope for the best. The only time undefined behaviour is preferable to an exception is when it results in performance gains; this is especially important in low-level library components that make up the foundations of your program (and in the case of the C++ standard library, everyone else's programs as well). In this case, if performance is a concern, and you are _sure_ that the pointer will not be null, the user can dereference the pointer instead of explicitly constructing the `indirect`:

```c++
int* i = get_some_pointer();
indirect<int> ii = *i; // you'd better be sure `i` is not null!
```

As a side note, if the source of the "not null" pointer is some legacy part of your interface that cannot be or hasn't yet been rewritten to use `indirect<T>` instead of `T*`, then you might benefit from using something like the [Guideline Support Library](https://github.com/Microsoft/GSL)'s `not_null` adapter, which provides additional run-time and compile-time assurances that a pointer is not null, while maintaining a more traditional pointer-like interface than `indirect`:

```c++
not_null<int*> i = get_some_pointer();
indirect<int> ii = *i; // `i` shouldn't be null
```

Another alternative to throwing an exception is to not support construction from `T*` at all; this way, the user is responsible for checking whether a pointer is null, as they would be when converting `T*` to `T&`. We decided against this for a number of reasons:

* No one is being forced to use the throwing constructor; the "default" operation (dereferencing `T*` then conversion from `T&` to `indirect<T>`) still has zero run-time overhead.
* Undefined behaviour is bad; if providing an alternative to pointer dereferencing which throws instead of invoking undefined behaviour results in safer client code, then this is a net win.
* Conversion of `T*` to `optional_indirect<T>` would be a huge hassle if an explicit constructor weren't provided (`auto oi = i ? *i : optional_indirect<T>()`); this would be especially annoying given that the conversion is perfectly safe; if we provide conversion from `T*` for `optional_indirect<T>` then we feels as through it should also be provided for `indirect<T>`.

#### <a name="design/make-functions"></a>The `make` functions

The `make_indirect` free function is a factory function which takes a single `T&` and returns an `indirect<T>`. Its main purpose is to automatically deduce the template parameter of the constructed `indirect`:

```c++
int i = {};
auto ii = make_indirect(i); // `decltype(ii)` is `indirect<int>`
```

Conceptually, `make_indirect` is similar to an implicit conversion, which is why there is no overload of `make_indirect` that takes a `T*`. If we were to provide such an overload, it the logical correctness and safety of `make_indirect` would be lost:

```c++
auto ix = make_indirect(x); // this may or may not throw, depending on `decltype(ix)`
```

We _could_ provide a function to allow automatic template parameter deduction from `T*`, but it would have to be named something that conveyed its unsafe nature (something like `cast_to_indirect`). Such a function is not included in this proposal.

Even though `indirect<T>` implicitly converts to `optional_indirect<T>`, the corresponding `make_optional_indirect` function is provided for convenience.

Incidentally, the `observer_ptr` proposal specifies that the factory function `make_observer` take a `T*`. We believe this to be a mistake; `make_observer` should instead take a `T&` (though we don't recommend adding implicit conversion from `T&` given its "smart pointer"-esque design). After all, `make_unique` and `make_shared` do not take a `T*`.

#### <a name="design/conversion/to-lvalue-ref"></a>Conversion to `T&`

#### <a name="design/conversion/to-ptr"></a>Conversion to `T*`

### <a name="design/assignment"></a>Assignment operators and the `{}` idiom

### <a name="design/optional"></a>Optional API functions

### <a name="design/comparison"></a>Comparison operations

### <a name="design/move"></a>Move behaviour

### <a name="design/immutability"></a>Relationships and immutability

### <a name="design/const-propagation"></a>Const-propagation

### <a name="design/cast-functions"></a>The `cast` functions

### <a name="design/get_pointer-functions"></a>The `get_pointer` free functions

### <a name="design/propagate-const"></a>Compatibility with `propagate_const`

### <a name="design/operator-dot"></a>Use of inheritance by delegation ("`operator.` overloading")

### <a name="design/why-not-ptr"></a>Why not `T*`?

### <a name="design/why-not-ref"></a>Why not `T&`?

### <a name="design/why-not-optional-ref"></a>Why not `optional<T&>`?

### <a name="design/why-not-optional-indirect"></a>Why not `optional<indirect<T>>`?

### <a name="design/why-not-observer_ptr"></a>Why not `observer_ptr<T>`?

### <a name="design/why-not-reference_wrapper"></a>Why not `reference_wrapper<T>`?

### <a name="design/why-not-not_null"></a>Why not `not_null<T*>`?

### <a name="design/naming"></a>Naming

## <a name="technincal"></a>Technical specifications

## <a name="acknoledgements"></a>Acknowledgements

## <a name="references"></a>References