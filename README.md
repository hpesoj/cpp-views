# `indirect` and `optional_indirect`: single object indirects for C++

`indirect` and `optional_indirect` are types used to indirectly reference another object without implying ownership. `indirect<T>` is a mandatory indirect of an object of arbitrary type `T`, while `optional_indirect<T>` is an optional indirect of an object of arbitrary type `T`. Both `indirect` and `optional_indirect` have reference-like initialization semantics and pointer-like indirection semantics.

## Contents

* [Motivation](#motivation)
* [Quick Example](#example)
* [Tutorial](#tutorial)
* [Design Rationale](#rationale)
* [FAQ](#faq)

## Motivation

Modern C++ guidelines recommend using high-level abstractions such as [`std::vector`](http://en.cppreference.com/w/cpp/container/vector), [`std::array`](http://en.cppreference.com/w/cpp/container/array), `std::array_indirect` _(not yet standardized)_, [`std::string`](http://en.cppreference.com/w/cpp/string/basic_string), [`std::string_indirect`](http://en.cppreference.com/w/cpp/string/basic_string_indirect), [`std::unique_ptr`](http://en.cppreference.com/w/cpp/memory/unique_ptr), [`std::shared_ptr`](http://en.cppreference.com/w/cpp/memory/shared_ptr) and [`std::optional`](http://en.cppreference.com/w/cpp/utility/optional) instead of raw pointers wherever possible. However, there is one major use of raw pointers that currently lacks a corresponding standardized high-level type: non-owning references to single objects. This is the gap filled by `indirect` and `optional_indirect`:

|          | Owned                                      | Non-Owned                      |
|----------|--------------------------------------------|--------------------------------|
| Single   | `unique_ptr` `shared_ptr` `optional`       | `indirect` `optional_indirect`         |
| Array    | `array` `vector` `unique_ptr` `shared_ptr` | `array_indirect`                   |
| String   | `string`                                   | `string_indirect`                  |
| Iterator | —                                          | _assorted_                     |

An [existing proposal](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4282.pdf) for `unique_ptr`-esque "dumb" pointer type [`observer_ptr`](http://en.cppreference.com/w/cpp/experimental/observer_ptr) aims to address the "non-owned single" use case, but `indirect` and `optional_indirect`, rather than being based on the owning smart pointer types that were designed to fill the "owned single" gap, are designed specifically for their intended purpose. The result is an API that:

* Improves code correctness
* More clearly documents intent
* Has more natural usage syntax

## <a name="example"></a>Quick Example

Take an example where a pointer might be used to represent an optional, non-owning relationship between a person and their animal companion (you can't, like, _own_ an animal, man):

```c++
struct person {
    animal* pet = {};
};

animal fluffy;
person bob;
bob.pet = &fluffy;
```

An `optional_indirect` can be used in place of the pointer:

```c++
struct person {
    optional_indirect<animal> pet;
};

animal fluffy;
person bob;
bob.pet = fluffy; // note: initialized from `animal&` instead of `animal*`
```

The `optional_indirect` can be tested for "empty" status as if it were a `bool`, and set to "empty" by assigning `{}` just like a pointer or [`std::optional`](http://en.cppreference.com/w/cpp/utility/optional):

```c++
assert(bob.pet);
bob.pet = {};
assert(!bob.pet);
```

A mandatory relationship can be expressed by using `indirect` instead of `optional_indirect`:

```c++
struct person {
    indirect<animal> pet;
};

animal fluffy;
person bob = { fluffy }; // note: initialization required on construction
```

Like pointers, both `indirect` and `optional_indirect` use the indirection operators (`*` and `->`) to access the referenced object:

```c++
bob.pet->sleep();
```

And `indirect` also implicitly converts to `T&`:

```c++
animal dolly = bob.pet; // note: no need to "dereference"
```

Both `indirect` and `optional_indirect` can be copied freely, and `indirect` implicitly converts to `optional_indirect`:

```c++
indirect<animal> pet = bob.pet;
optional_indirect<animal> possible_pet = pet;
```

## <a name="tutorial"></a>Tutorial

In this demonstration, we will create a type that can be used to create a tree of connected nodes. Nodes will only keep non-owning references to other nodes. The client is responsible for the lifetime of each node. Each node will have an optional parent and zero or more children. For simplicity, we will not attempt to prevent cycles from being formed.

With only pointers and references in our arsenal, we might start with something like this:

```c++
class node {
private:
    node* parent;

public:
    void set_parent(node* new_parent) {
        parent = new_parent;
    }

    node* get_parent() const {
        return parent;
    }
};
```

We make `parent` a pointer because it is natural to use the null pointer state to represent the lack of a parent. However, we already have a bug in our code: we forgot to initialize `parent`! In addition, the meaning of `node*` is not 100% clear. We want it to mean "_non-owning_, _optional_ reference to a _single_ `node`". However, pointers have other potential uses:

* Pointers can be used in place of references, where they are assumed to _not_ be null; a violation of this condition is a programming error and may result in undefined behaviour.
* Pointers can be used to represent arrays of objects; indeed, pointers define the subscript operator (e.g. 'p[i]') for indexing into arrays; however, using an index greater than zero with a pointer that references a single object is undefined behaviour.
* Pointers can be used to represent [random-access iterators](http://en.cppreference.com/w/cpp/concept/RandomAccessIterator); indeed, pointers define the arithmetic operators (e.g. 'p++') used to move random-access iterators; however, moving and then dereferencing a pointer that references a single object is undefined behaviour.
* A dynamically created/allocated object, array or system resource referenced by a pointer may be implicitly owned by one or more holders of a copy of the pointer; indeed, pointers implicitly convert to `void*` so they can be used in expressions such as `delete p` or `free(p)`; failure to correctly destroy/deallocate such an object, array or resource may result in either a resource leak or undefined behaviour.

With all this potential undefined behaviour, it is important that it is obvious to the reader what `node*` _represents_. We _could_ provide supplementary documentation, but if we replace `node*` with `optional_indirect<node>`, we can convey our intentions automatically using the type system. An `optional_indirect<T>` _is_ a _non-owning_, _optional_ reference to a _single_ object of type `T`: an optional indirect of `T`. With `optional_indirect<T>`, we also get compile-time assurances that we didn't with `T*`:

* `optional_indirect<T>` is always default initialized (to its empty state)
* `optional_indirect<T>` does not define the subscript operator (it isn't an array)
* `optional_indirect<T>` does not define arithmetic operations (it isn't an iterator)
* `optional_indirect<T>` does not implicitly convert to `void*` (it doesn't "own" anything)

So let's replace `node*` with `optional_indirect<node>`:

```c++
class node {
private:
    optional_indirect<node> parent;

public:
    void set_parent(optional_indirect<node> new_parent) {
        parent = new_parent;
    }

    optional_indirect<node> get_parent() const {
        return parent;
    }
};
```

Note that with `optional_indirect`, the calling syntax changes slightly. `optional_indirect<T>` is no longer implicitly constructible from `T*`, since this conversion may not always be conceptually or even semantically correct since pointers can represent many things. Instead, `optional_indirect<T>` is implicitly constructible from `T&`; this means that we `set_parent` and test the result of a call to `get_parent` _without_ taking the address of the `node`s:

```c++
node a, b;
b.set_parent(a)
assert(b.get_parent() == a);
```

This is arguably a more natural syntax; indeed, references, not pointers, are the most natural reference types in C++.

We use `{}` to specify the empty state and conversion to `bool` to test for the empty state, just as with a pointer or [`std::optional`](http://en.cppreference.com/w/cpp/utility/optional):

```c++
b.set_parent({})
assert(!b.get_parent());
```

Now, it would be nice if `node` also kept track of its children so that we can navigate both up _and_ down the tree. Using only references and pointers, we might initially consider storing a `std::vector` of references:

```c++
    std::vector<node&> children;
```

Alas, this will not compile. References have irregular copy behaviour: unlike with pointers, copy assignment applies directly to the referenced object, not to the reference itself. This behaviour precludes storage of references in STL containers like `std::vector`; instead, we are forced to store pointers:

```c++
private:
    …
    std::vector<node*> children;

public:
    …

    std::size_t get_child_count() const {
        return children.size();
    }

    node& get_child(std::size_t index) const {
        return *children[index];
    }

private:
    void add_child(node& child) {
        children.push_back(&child);
    }

    void remove_child(node& child) {
        children.erase(std::find(children.begin(), children.end(), &child));
    }
```

Now, we could replace `node*` with `optional_indirect<node>`, but this is misleading since children are _not_ optional: `node*` should never be null and `optional_indirect<node>` should never be empty. So instead of `node&` or `node*`, we can use `indirect<node>`, the mandatory counterpart to `optional_indirect<node>`. In addition to all the benefits that `optional_indirect` provides, `indirect`  _must_ always reference an object: it has no empty state. A `indirect<T>` _is_ a _non-owning_, _mandatory_ reference to a _single_ object of type `T`: a indirect of `T`. And it can be stored in STL containers, just like `optional_indirect<T>` or `T*`.

Let's replace all instances of `node&` and `node*` with `indirect<node>`, and add the calls to `add_child` and `remove_child` to `set_parent`:

```c++
private:
    …
    std::vector<indirect<node>> children;

public:
    void set_parent(optional_indirect<node> new_parent) {
        if (parent) parent->remove_child(*this);
        parent = new_parent;
        if (parent) parent->add_child(*this);
    }

    …

    indirect<node> get_child(std::size_t index) const {
        return children[index];
    }

private:
    void add_child(indirect<node> child) {
        children.push_back(child);
    }

    void remove_child(indirect<node> child) {
        children.erase(std::find(children.begin(), children.end(), child));
    }
};
```

We now have compile-time guarantees that we didn't have previously. Note that the syntax and semantics of `indirect<T>` are notably different from those of `T&`. For example, when using `auto` type deduction, copying a `T&` copies the referenced object by default, while copying a `T*` copies the pointer by default:

```c++
auto aa = b.get_parent(); // `decltype(aa)` is `node*`
auto& bb = a.get_child(0); // `decltype(bb)` is `node&`
auto cc = a.get_child(0); // `decltype(cc)` is `node`
```

And `T&` and `T*` use different syntax to access the referenced object:

```c++
bb.set_parent({}); // `T&` uses `.`
aa->set_parent(&bb); // `T*` uses `->`
```

Conversely, both `indirect<T>` and `optional_indirect<T>` have pointer-like copying and indirection semantics:

```c++
auto aa = b.get_parent(); // `decltype(aa)` is `optional_indirect<node>`
auto bb = a.get_child(0); // `decltype(bb)` is `indirect<node>`

bb->set_parent({}); // `indirect<T>` uses `->`
aa->set_parent(bb); // `optional_indirect<T>` uses `->`
```

`indirect<node>` _can_ implicitly convert to `node&` if the type is specified, though indirection syntax must still be used to access the referenced object if conversion isn't triggered:

```c++
node& bb = a.get_child(0);
a.get_child(0)->set_parent({});
```

## <a name="rationale"></a>Design Rationale

### <a name="rationale-construction-from"></a>Construction from and conversion to `T&` and `T*`

Both `indirect` and `optional_indirect` are constructible from and convertible to `T&` and `T*`, but only construction of `indirect<T>` and `optional_indirect<T>` from `T&` and conversion from `indirect<T>` to `T&` are _implicit_. The idea is that conversions should be implicit if they are functionally equivalent and conversion is safe virtually all of the time, while they should be explicit if the types are not always functionally equivalent or conversion is safe only some of the time ("safe" here means not invoking undefined behaviour _and_ being `nothrow`). `T&` should always represent a valid "indirect" of an object in a well-formed C++ program, so implicit construction of `indirect<T>` and `optional_indirect<T>` from `T&` is correct; conversion from `optional_indirect<T>` to `T&` is not safe as the `optional_indirect<T>` may be empty, so conversion must be explicitly specified. Conversely, `T*` may or may not be a valid reference to an object; for example:

* `T*` may have ownership semantics
* `T*` may represent an array
* `T*` may be an iterator (maybe even a "past-the-end" iterator)

In none of these cases would it be correct to allow either `indirect<T>` or `optional_indirect<T>` to be implicitly constructed from  or converted to `T*`. It is up to the programmer to explicitly specify these conversions when they are sure it is correct to do so.

It's worth noting that if conversion from `optional_indirect<T>` to `T*` were implicit, then array-like operations such as `v[i]`, iterator-like operations such as `v++`, and ownership operations such as `delete v` would automatically be enabled. This could be considered reflective of the fact that implicit conversion implies functional equivalence.

Also note that there are no overloads of `make_indirect` or `make_optional_indirect` that take a pointer, as they are merely intended to facilitate automatic type deduction, and are conceptually similar to implicit constructors.

### <a name="rationale-construction-from-pointer-to-indirect"></a>Construction of `indirect` from `T*` or `optional_indirect`

Explicit construction of `indirect` from a pointer or an `optional_indirect` _is_ supported and throws a `std::invalid_argument` if the pointer is null or the `optional_indirect` is empty. It could be argued that such a feature is not strictly required as part of a minimal and efficient API (the user could implement equivalent behaviour just as efficiently as a non-member function) and encourages programming errors as the user may not realize that constructing a `indirect` from a pointer is not `nothrow`. However, the explicit nature of the conversion means that the user is unlikely to do it by accident:

```c++
    foo(indirect<int>(p));
```

Thus, even though such functionality isn't strictly necessary, it is a convenient way to safely (i.e. without invoking undefined behaviour) convert a pointer or `optional_indirect` to a `indirect`. It also seems appropriate to support such operations given the inclusion of the [throwing accessor function](#rationale-named-member-functions) `value` in `optional_indirect`, a function which is also strictly not required and is inspired by the design of `std::optional`.

### <a name="rationale-construction-from-rvalue"></a>Construction from `T&&`

Initialization of `indirect` and `optional_indirect` from rvalues is allowed. This means that it is possible to pass temporary objects to functions taking `indirect<T const>` or `optional_indirect<T const>`:

```c++
void foo(indirect<int const> i) { … }

int i = 42;

foo(i); // okay
foo(42); // also okay
```

However, unlike a reference-to-const, a `indirect<T const>` does not extend the lifetime of the temporary from which it is constructed. Thus, while this is safe:

```c++
int const& i = 42; // lifetime of temporary extended by `i`
std::cout << i; // a-okay
```

This is most definitely not safe:

```c++
indirect<int const> i = 42; // temporary destroyed after assignment
std::cout << *i; // undefined behaviour!
```

Some might consider this grounds to disable construction from `T&&` as in [`std::reference_wrapper`](http://en.cppreference.com/w/cpp/utility/functional/reference_wrapper). However, `std::string_indirect` sets a precedent by allowing such behaviour:

```c++
std::string_indirect v = std::string("hello, world");
std::cout < v; // undefined behaviour
```

Indeed, prohibiting passing of temporaries as "indirect" arguments would be extremely limiting. The user must simply be aware that lifetime extension does not occur with such types.

### <a name="rationale-assignment"></a>Assignment operators and the `v = {}` idiom

Neither `indirect` nor `optional_indirect` explicitly define any assignment operators. This enables automatic support of the `v = {}` idiom for resetting an object to its default state:

```c++
optional_indirect<int> v1 = i;
v1 = {}; // `v1` is now empty

indirect<int const> v2 = i;
v2 = {}; // compile error (`indirect` has no default state)
```

If assignment operators were explicitly implemented, extra measures would need to be taken to ensure that `v = {}` compiled for `optional_indirect` and didn't compile for `indirect`. Since both types are very lightweight, the compiler should be able to easily elide the additional copy, so there should be no performance penalty for this design choice.

### <a name="rationale-move"></a>Move behaviour

Neither `indirect` not `optional_indirect` define distinct move behaviour: moving is equivalent to copying. It could be argued that a moved-from `optional_indirect` should become empty, but in reality we have no way of knowing how client code wants a moved-from `optional_indirect` to behave, and such behaviour would incur a potentially unnecessary run-time cost. In addition, a moved-from `indirect` cannot be empty, so having different behaviour for `optional_indirect` would be potentially surprising. An additional advantage of keeping the compiler-generated move operations is that `indirect` and `optional_indirect` can be [trivially copyable](http://en.cppreference.com/w/cpp/concept/TriviallyCopyable) (they can be copied using [`std::memcpy`](http://en.cppreference.com/w/cpp/string/byte/memcpy), a performance optimization that some library components employ).

### Mutability

It has been suggested that the term "view" as used in the standard library implies immutability of the referenced data (i.e. read-only). Currently, the only "view" type in the standard is [`std::basic_string_view`](http://en.cppreference.com/w/cpp/string/basic_string_view), which is indeed a read-only view of a string of characters. This, however, is a design choice specific to the `string_view` use case, as can be seen by the rationale for immutability given in the [`string_view` proposal](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3762.html):

* It's more common to pass strings by reference/pointer-to-const, so the default (e.g. the `string_view` type alias) should be immutable.
  * While it is arguably more common to see `T const&` than `T&`, there are no "convenience" type aliases for `view` and `optional_view`, so there is no corresponding need to decide a default behaviour.
* You wouldn't be able to change the length of a mutable `string_view` which limits its usefulness.
  * This is a string/array-specific problem not shared by `view` and `optional_view`.
* Developers using existing `string_view`-like classes have found no use case for a mutable `string_view`.
  * There are clear use cases for mutable `view` and `optional_view`.
* The template design would be complicated unnecessarily to support an uncommon use case.
  * The design of `view` and `optional_view` is not significantly complicated by supporting the mutable use case, which is not nearly as uncommon as for `string_view`.

In addition, two recent proposals for `std::array_view` ([here](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4512.html) and [here](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0122r0.pdf)) both support the mutable use case. This makes sense, because arrays are commonly fixed in size (e.g. the length of a `std::array` is fixed at compile time) yet the individual elements are mutable, while a mutable string is generally expected to be able to change in length. Indeed, the use case for a fixed-size, mutable `string_view` is probably covered by `array_view<char>`.

In summary, `std::string_view` is read-only for reasons specific to the "string" use case, and there are several proposals for a mutable `std::array_view`. There is no reason to think that "view" implies immutability.

### <a name="rationale-get"></a>Compatibility with `std::experimental::propagate_const`

It has been decided that `indirect` and `optional_indirect` should not try to "propagate constness" to the referenced object; for example, `indirect<T>::operator*() const` should return `T&` not `T const&` (and there should not exist a non-const overload). Instead, `indirect` and `optional_indirect` are designed to be compatible with the proposed [`std::experimental::propagate_const`](http://en.cppreference.com/w/cpp/experimental/propagate_const) interface adapter. Unfortunately, `propagate_const` has been designed to work with "pointer-like" types, and `indirect` and `optional_indirect` are not "pointer-like" in the conventional sense.

First, `propagate_const` requires compatible types to implement a member function named `get` which returns the object's underlying raw pointer. While `get` may be a great name for such a function in a smart pointer, it is terribly undescriptive for a indirect type. It would be better if conversion to pointer were left to the explicit conversion operator.

Secondly, `propagate_const` requires compatible types to be convertible to `bool`. While this is no problem for `optional_indirect`, it really makes no sense for `indirect` to convert to bool. It would be better if this weren't a requirement of `propagate_const` at all.

Thirdly, `propagate_const` works well with raw pointers because it implicitly converts to `pointer` and `const_pointer`. However, `propagate_const<T>` doesn't implicitly convert to `T` or the const version of `T` for an arbitrary `T`. This could be supported if `propagate_const` had some way of knowing what the const version of `T` was for an arbitrary `T`.

And finally, `propagate_const` currently only supports _implicit_ conversion to `pointer` and `const_pointer`, while `indirect` and `optional_indirect` variously implement implicit _and_ explicit conversion to both `pointer` and `const_pointer` _as well as_ `reference` and `const_reference`.

Fortunately, `propagate_const` has not yet been standardized, so there we have a chance to fix things. In order to allow `propagate_const` to work with all types that have indirection semantics, the following changes are suggested:

* Add a `std::get_pointer` free function that obtains the underlying pointer for various standard library types. `propagate_const` should rely on this function rather than a `get` member function. `propagate_const<T>::get` should be enabled only if an appropriate `T::get` exists. Note that Boost provides a [similar function](www.boost.org/doc/libs/release/boost/get_pointer.hpp) already.
  * `get_pointer(T* p)` will return `p`
  * `get_pointer(unique_ptr<T> const& p)` and `get_pointer(shared_ptr<T> const& p)` will return `p.get()`
  * `get_pointer(indirect<T> const& v)` and `get_pointer(optional_indirect<T> const& v)` will return `static_cast<T*>(v)`
  * `get_pointer(propagate_const<T>& pc)` and `get_pointer(propagate_const<T> const& pc)` will return `get_pointer(pc.t)`, where `pc.t` is the underlying object of type `T`
* Enable `propagate_const<T>::operator bool` _only_ if `T::operator bool` exists.
* Require compatible types to provide a member type `const_type` (for example, `indirect<T>::const_type` would be `indirect<T const>`). This information can then be used to implement implicit conversion to `T` and `T::const_type`.
* Conditionally define either implicit or explicit conversion from `propagate_const<T>` to `pointer` and `const_pointer` and/or `reference` and `const_reference`, depending on whether the corresponding conversions exist for `T`.
* Only define `propagate_const<T>::element_type` if `T::element_type` exists. Similarly, it might be useful to define `propagate_const<T>::value_type` if `T::value_type` exists (as it does for `indirect` and `optional_indirect`), alongside other common type aliases.

A sample implementation of a version of `propagate_const` with these changes can be found [here](https://github.com/hpesoj/cpp-indirects/blob/master/tests/propagate_const.hpp).

### <a name="rationale-optional"></a> Use of `optional_indirect<T>` rather than `std::optional<indirect<T>>`

It could be argued that the role of an optional "indirect" should be played by `optional<indirect<T>>`. After all, there is no `std::optional_string_indirect` to act as a higher-level version of a `char const*` which represents an optional string; in fact, it could be argued that `char const*` should never be conceptually never be null, which is why types such as `std::optional_string_indirect` don't exist. Surely the same applies to a `T*` that represents a reference?

In practice, it is common to use `T*` as an optional reference and `T&` as a mandatory reference, while a `char const*` that represents a string is more commonly expected to not be null. In addition, the double indirection syntax required when using `optional<indirect<T>>` is very clunky:

```c++
foo f;
auto v = make_optional(make_indirect(f));
(*v)->bar = 42;
```

Also, `optional<indirect<T>>` generally occupies _twice_ the memory of `optional_indirect<T>`, which may put people off of using such a simple type.

### <a name="rationale-nullopt"></a>Reuse of `std::nullopt` and `std::bad_optional_access`

Given that `optional_indirect<T>` is conceptually similar to `std::optional<indirect<T>>`, it makes sense to reuse features standardized with `std::optional`.

The current implementation provides its own `nullopt_t`, `nullopt` and `bad_optional_access`, since implementations of `std::optional` are not yet widely available.

### <a name="rationale-named-member-functions"></a>Supplementary accessors (`value` and `value_or`)

The optional value type [`std::optional`](http://en.cppreference.com/w/cpp/utility/optional) provides the throwing accessor function `value` and the convenience accessor function `value_or`. Neither of these functions are necessarily part of a minimal and efficient API (they could be just as efficiently implemented as non-member functions), but presumably they were felt to be useful enough to include anyway. `optional_indirect` implements corresponding functions which work in roughly the same way. The only difference is that `optional` has rvalue overloads that move the contained value out of the `optional` (e.g. `t = std::move(op).value()` will move the value out of `op` and into `t`). `optional_indirect` does not own the value it references, so it does not assist in such operations. Thus, `value` always returns an lvalue reference and `value_or` always performs a copy.

### <a name="rationale-naming"></a>Naming

The names `indirect` and `optional_indirect` are based on conventions already set by the C++ standard library, namely:

* `std::string_indirect` – a non-owning "indirect" of a string
* `std::optional` – an optional value type

The term "indirect" seems to be used to refer to an object that allows the whole or part of another object or data structure to be examined without controlling the lifetime of said object or data structure (much like a reference). This seems like an accurate description `indirect` and `optional_indirect`.

The terms "reference" and "pointer" are inappropriate since they are overly general and already have established meaning in the C++ lexicon. The term "indirect" seems more appropriate as it describes the function of the feature rather than how it is likely to be implemented.

The term "observer" could potentially be used (as in `std::experimental::observer_ptr`); there is no arguing that it is descriptive of what the classes do. However, the term "indirect" has already been enshrined in the C++ standard, and there is also the concern that there will be confusion with the unrelated ["observer pattern"](https://en.wikipedia.org/wiki/Observer_pattern), a software design pattern that is widely used in C++ codebases. There is also the minor benefit that "indirect" is four characters less to type than "observer".

The term "optional" has an obvious meaning. Indeed, an `optional_indirect<T>` is pretty much functionally equivalent to an `optional<indirect<T>>`; the former is simply a bit nicer to use.

One possible question is, should the "optional" type be the default? In other words, should `optional_indirect` be named `indirect` and `indirect` be named something like `mandatory_indirect` or `compulsory_indirect`? Aside from the precident set by `std::optional` (which I believe is pretty strong by itself), I think the answer is "no": `indirect` is inherently a simpler type; there is no need to account for the additional "empty" state which, like null pointers, is a major source of potential programming errors. The mandatory type should be the default choice, and the optional type should be opted-into. The naming of the types should reflect this.

## <a name="faq"></a>FAQ

### <a name="faq-good-enough"></a>Aren't `T*` and `T&` good enough?

The suggestion has been made that raw pointers work just fine as optional indirect types, and there is no need to introduce new types when we already have long-established types that are well-suited to this role.

Some argue that once all other uses of pointers have been covered by other high-level types (e.g. `std::unique_ptr`, `std::string`, `std::string_indirect`, `std::array`, `std::array_indirect`, …) the only use case that will be left for raw pointers will be as optional indirect types, so it will be safe to assume that wherever you see `T*` it means "an optional indirect of `T`". This is not true for a number of reasons:

* This is only a convention; people are not obliged to use `T*` only to mean "an optional indirect of `T`".
* Plenty of existing code doesn't follow this convention; even if everyone followed convention from this point forward, you still cannot make the assumption that `T*` always means "an optional indirect of `T`".
* Low-level code (new and existing) necessarily uses `T*` to mean all sorts of things; even if all new and old code followed the convention where appropriate, there would still be code where `T*` does not mean "an optional indirect of `T`", so you _still_ cannot make that assumption.

Conversely, wherever you see `optional_indirect<T>` in some code, you _know_ that it means "an optional indirect of `T`" (unless the author of the code was being perverse). Explicitly documenting intent is generally better than letting readers of your code make assumptions.

Aside from the case for documenting intent, it can be argued that pointers and references are not optimally designed for use as indirect types. A well-designed type is _efficient_ and has an API that is _minimal_ yet _expressive_ and _hard to use incorrectly_. Pointers as indirect types may be efficient and do have a somewhat expressive API, but their API is not minimal and is very easy to use incorrectly. Case in point, this is syntactically correct but semantically incorrect code:

```c++
int i = 0;
int* p = &i;
p += 1; // did I mean `*p += 1`?
*p = 42; // undefined behaviour
```

On the other hand, references as indirect types are efficient and have a minimal API that is generally hard to use incorrectly. However, the reference API is not as expressive as it could be, since references cannot be reassigned after construction to reference a different object:

```c++
int i = 0, j = 42;
int& r = i;
r = j; // modifies `i`; does not make `r` reference `j`
```

As a bonus, `indirect<T>` is compatible with `std::experimental::propagate_const`, while `T&` is not.

`indirect` and `optional_indirect` are _as_ efficient as pointers and references, and have been _purpose-designed_ as indirect types to be minimal, expressive and hard to use incorrectly.

### <a name="faq-operator-dot"></a>Why does `indirect` use pointer-like syntax when it can't be null? Typing `*` or `->` is a hassle. Shouldn't you wait until some form of `operator.` overloading has been standardized?

Even with some form of `operator.` overloading, `indirect` would have the same design.

The various `operator.` "overloading" proposals put forward possible C++ language features that could allow one type to "inherit" the API of another type _without_ using traditional inheritance. Such mechanisms _could_ be used to implement `indirect` such that instead of writing `(*v).bar` (or `v->bar`), where `v` is of type `indirect<foo>` and `foo` has a member `bar`, we would instead write `v.bar`. At first glance, this makes sense: since a `indirect` cannot be null, why not model it after a reference instead of a pointer?

Alas, there is no such thing as a free lunch. The question is, what should `indirect::operator=(T& t)` do? There are two possibilities: either the `indirect` is reassigned to reference the object referenced by `t`, or the `indirect` indirectly assigns the object referenced by `t` to the object the `indirect` references.

If our design specifies the former, then we have potentially surprising asymmetrical behaviour:

```c++
f.bar = g.bar; // modifies `f`
f = g; // modifies `f`

v.bar = g.bar; // modifies `f`
v = g; // modifies `v`
```

If our design specifies the latter, then we have _different_ potentially surprising asymmetrical behaviour:

```c++
f = g; // modifies `f`
f = u; // modifies `f`

v = g; // modifies `f`
v = u; // modifies `v`
```

In both cases, there is an "exceptional" case that the programmer has to remember. In addition, these are not behaviours that they are already likely to be familiar with: `indirect` would have copy/assignment behaviour not exactly like a reference, and not exactly like a pointer, but like a hybrid of the two. What's more, `indirect` would then behave differently from `optional_indirect`! The chance of programming error with either design is very high. `indirect` avoids these problems by borrowing the indirection and copy behaviour used by pointers and pointer-like types:

```c++
f.bar = g.bar; // modifies `f`
f = g; // modifies `f`
f = u; // error: no conversion from `indirect<foo>` to `foo`
f = *u; // modifies `f`

v.bar = g.bar; // error: `indirect<foo>` has no member `bar`
v = g; // modifies `v`
v = u; // modifies `v`
v = *u; // modifies `v`

(*v).bar = g.bar; // modifies `f`
(*v) = g; // modifies `f`
(*v) = u; // error: no conversion from `indirect<foo>` to `foo`
(*v) = *u; // modifies `f`
```

This behaviour is both symmetrical _and_ familiar to experienced programmers: both `indirect` and `optional_indirect` are initialized like a reference, and have copy and indirection behaviour like a pointer. This is predictable behaviour that is easy to remember.

Of course, `operator.` "overloading" does have its uses, but this is not one of them.

### <a name="faq-reference-wrapper"></a>Isn't `indirect` the same as `std::reference_wrapper`?

No. They are very different. In short, `indirect` has pointer-like indirection semantics, whereas `std::reference_wrapper` has reference-like direct-access semantics.

The key difference between `indirect` and `reference_wrapper` is in assignment and comparison. While both types behave the same on construction:

```c++
int i = 42;
reference_wrapper<int> r = i; // `r` indirectly references `i`
indirect<int> v = i; // `v` indirectly references `i`
```

They have different behaviour on assignment:

```c++
int j = 21;
r = j; // `r` still indirectly references `i`; `i` and `j` are now equal
v = j; // `v` now indirectly references `j`; `i` and `j` are not equal
```

And on comparison:

```c++
if (r == i) { … } // true if the object `r` indirectly references is equal to `i`
if (v == i) { … } // true if v indirectly references `i`
```

In other words, in this case `reference_wrapper` is designed to behave like a reference, while `indirect` is designed to behave more like a pointer. The difference between a reference and a `reference_wrapper` is that copy assigning the latter will rebind it to indirectly reference whatever the other `reference_wrapper` referenced.

```c++
r = std::ref(j); // `r` now indirectly references `j`
v = make_indirect(i); // `v` now indirectly references `i`
```

This slightly modified behaviour allows `reference_wrapper`, among other things, to be stored in an STL container. `indirect` can also be stored in an STL container, but just as a stand-alone `indirect` behaves very differently than a stand-alone `reference_wrapper`, so too do STL containers of these types. Take, for example, the `remove_child` member function from the tutorial:

```c++
private:
    …
    std::vector<indirect<node>> children;

public:
    …
    void remove_child(indirect<node> child) {
        children.erase(std::find(children.begin(), children.end(), child));
    }
```

A call to `remove_child` will erase from the container any `indirect` which indirectly references the same `node` as `child`. Now say we were to use `reference_wrapper` instead of `indirect`:

```c++
private:
    …
    vector<reference_wrapper<node>> children;

public:
    …
    void remove_child(node& child) {
        children.erase(std::find(children.begin(), children.end(), child));
    }
```

If `operator==` isn't defined for `node`, then this code won't compile. If we _do_ define `operator==` for `node`, then this code will compile but won't have the same effect as before. A call to `remove_child` will now erase from the container any `indirect` which indirectly references a `node` which _is equal_ to the `node` indirectly referenced by `child`.

In short, operations on `indirect` tend to modify or inspect the `indirect` itself (like a pointer), while operations on `reference_wrapper` tend to modify or inspect the indirectly referenced object (like a reference). There are times when `indirect` is appropriate and times where `reference_wrapper` is what you need, but they are certainly not interchangeable.

### <a name="faq-not-null"></a>Isn't `indirect` the same as `gsl::not_null`?

No. The aims of `indirect` and `not_null` are different. In short, `indirect<T>` is a indirect of a single object of type `T`, while `not_null<T*>` is simply a `T*` that should not be null.

According to the [current documentation](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#i12-declare-a-pointer-that-must-not-be-null-as-not_null) in the C++ Core Guidelines, the purpose of `not_null` is simply to indicate to the compiler and the reader that a pointer should not be null. The current design of `not_null` allows optional run-time checks to be enabled via a pre-processor switch, but the C++ Core Guidelines suggest enforcement via a static analysis tool. The examples given include use with C-style strings (`char const*`), something that would never be a use case for `indirect`.

One possible source of confusion is the fact that the [current implementation](https://github.com/Microsoft/GSL/blob/master/gsl/gsl) of `not_null` explicitly `delete`s a number of pointer arithmetic operations. At first, it may seem like an indication that `not_null` should not be used to store pointers that represent arrays or iterators. However, one of the authors of `not_null` has [clarified](https://github.com/Microsoft/GSL/issues/417) that these restrictions simply aim to encourage the use of the complementary `span` facility (the GSL version of an `array_indirect`) instead of manually iterating over ranges or indexing into arrays represented by pointers.

So `indirect` and `not_null` are not the same at all. They perform complementary functions and could be used side-by-side in the same codebase, no problem.

### <a name="faq-observer-ptr"></a>Isn't `optional_indirect` the same as `std::experimental::observer_ptr`?

Both `optional_indirect` and [`std::experimental::observer_ptr`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4282.pdf) have essentially the same purpose (high-level indirect/observer types that document intent) but slightly different designs. While the design of `observer_ptr` is based on the existing standard library smart pointer types, `optional_indirect` is specifically designed to be a indirect type. The fact that pointers are already _okay_ as indirect types means that there isn't a great deal of difference between `optional_indirect` and `observer_ptr`; however, there are a couple of things worth noting.

First, `observer_ptr` has to be explicitly constructed from a pointer:

```c++
foo f, g;
observer_ptr<foo> p{&f};
p = make_observer(&g);
assert(p == make_observer(&g));
p->bar();
p = {};
```

On the other hand, while `optional_indirect` too allows explicit construction from a pointer, it also allows _implicit_ construction from a _reference_. This tends to mean using `optional_indirect` feels slightly more natural to use and results in slightly less verbose code than `observer_ptr`:

```c++
foo f, g;
optional_indirect<foo> v = f;
v = g;
assert(v == g);
v->bar();
v = {};
```

Second, `observer_ptr` lacks a "not null" counterpart. While the case for a so-called "vocabulary" type to replace references is weaker than that for non-owning pointers (`T&` pretty much exclusively means "indirect of `T`"), the irregular copying behaviour of references makes the case for a complementary "not null" non-owning reference type fairly strong.

There are a number of other differences between `optional_indirect` and `observer_ptr`, but they are less significant:

* `observer_ptr` has the "ownership" operations `reset` and `release`, presumably because it its API is modelled on the owning smart pointer types.
* `optional_indirect` has "safe" construction from `T*` which throws if called with a null `T*`.
* `optional_indirect` has the "safe" accessor function `value` which throws if called on an empty `optional_indirect`. This part of the API is modelled after `std::optional`.
* `optional_indirect` has cast operations `static_indirect_cast`, `dynamic_indirect_cast` and `const_indirect_cast`.

The question is, is there room for both `optional_indirect` (and `indirect`) _and_ `observer_ptr` in the C++ programmer's toolkit? If a separate case can be made for an "owning smart pointer"-like non-owning "smart" pointer, then perhaps there is. Otherwise, maybe we have two different designs for two competing types trying to solve the same problem. The case for `optional_indirect` and `indirect` has been laid out here, but the best design for such a type is of course up for discussion.

### I use `T&` to represent permenant relationships and `T*` for relationships that can be changed. Don't I lose this functionality with `indirect` and `optional_indirect`?

The `const` mechanism is the natural way to model permenancy in C++. `indirect` and `optional_indirect` provide a natural way to model mandatory and optional relationships. Combining these gives greater flexibility than `T&` or `T*` can alone:

* `indirect<T>` – a mandatory, changeable relationship
* `indirect<T> const` – a mandatory, permanent relationship (equivalent to `T&`)
* `optional_indirect<T>` – an optional, changeable relationship (equivalent to `T*`)
* `optional indirect<T> const` – an optional, permanent relationship

### Why would you ever use `indirect<T>` instead of `T&` in an API?

Using `optional_indirect<T>` in an API has a number of clear advantages over `T*`, including:

* Documentation of intent
* Omission of inappropriate operations
* Natural initialization syntax

The advantages of `indirect<T>` over `T&` in an API, on the other hand, are not as clear cut. However, they are not always interchangeable.

When used as a function parameter type, `indirect<T const>` and `T const&` have subtly different behaviour:

```c++
void foo(int const& v) { … }
void bar(indirect<int const> v) { … }

foo(42);
foo({}); // same as `foo(int{})`, i.e. `foo(0)`

bar(42);
bar({}); // error: cannot convert from `{}` to `indirect<int const>`
```

The minor advantage of using `indirect` here is that client code won't break if you switch to `optional_indirect` (with which `{}` represents the "empty" state) at some point in the future. Other than this, there is little difference between the two. When used as a function return type, `indirect<T>` and `T&` are more obviously different:

```c++
int& foo() { … }

int i = 42;
foo() = i; // assigns 42 to whatever `foo` returned a reference to
auto x = foo(); // `decltype(x)` is `int`
```

```c++
indirect<int> foo() { … }

int i = 42;
foo() = i; // modifies the temporary `indirect` returned by `foo` to reference `i`
auto x = foo(); // `decltype(x)` is `indirect<int>`
```

`indirect<T>` has pointer-like indirection semantics, while `T&` has reference-like direct-access semantics. Thus, you would not want to use `indirect` when you wished to provide _direct_ access to an object, like when implementing `operator[]` for an array type. You _may_ want to use `indirect` if you wish to provide a pointer-like indirect handle to an object; in this case you may also wish to return a `indirect<T> const` to prevent the caller from accidentally assigning to the temporary:

```c++
indirect<int> const foo() { … }

int i = 42;
foo() = i; // error: cannot call `operator=` on `indirect<int> const`
```

The exact philosophical difference between `T&` and `indirect<T>` is up for discussion.
