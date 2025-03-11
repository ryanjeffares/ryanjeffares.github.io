---
layout: post
title: "You'll Pry C++ From My Cold, Dead Hands"
date: 2025-03-06 16:45:00 -0000
categories: cpp, languages
---
## Introduction

So, C++ has been taking a lot of flak recently. A lot of it honestly comes from script kiddies with skill issues, but there is also a lot of well meaning concern about memory safety and some of the less user-friendly aspects of the language. Even the US government [hates it](https://media.defense.gov/2023/Apr/27/2003210083/-1/-1/0/CSI_SOFTWARE_MEMORY_SAFETY_V1.1.PDF) (has somebody sat in the oval office and tried to explain to Donald Trump what a segfault is?).

C++ is far from perfect - as with any language, there are things that are bad about it - and I can't even keep up with the endless committee drama. These are things that others have covered in excruciating detail, and it's not what I'm talking about here. What I want to talk about are some completely unique features and abilities of C++ that I oh-so miss in any other language, and just make me think that that language's version of it sucks. So long as other languages don't have these features, you'll have to pry C++ from my cold, dead hands.

## 1. `template`s are Better than Generics

C++'s templates are fundamentally different to generics in pretty much any other language (as far as I know - TypeScript types can [run Doom](https://www.youtube.com/watch?v=0mCsluv5FXA), and I've never written TypeScript but maybe they're better?), and it's the differences that make templates just so much better (in my opinion).

As long as the code is valid syntax, you can write basically whatever the hell you like in a template, and it won't be checked until the template is actually *instantiated*. They can also be specialised as well as overloaded, and everything is resolved at compile time with no runtime type erasure. This immediately sets them apart from the likes of C# generics and whatever the hell Java has. Rust's generics are pretty good, but just a little more restrictive.

So, what do these differences allow for? Well, take this for example:

```cpp
template<typename T>
auto add(const T& a, const T& b) -> decltype(a + b) {
    return a + b;
}
```

This generic `add` function, as it is right here with no further bounds or explicit restrictions at the function's definition, is simply not possible in any other mainstream language. They complain that you can't add together two `T`s without knowing more about the type. When we call `add<int>(2, 3)`, it compiles fine and it returns `5`. When we call `add<std::string>("Hello ", "World")`, it compiles fine and returns `"Hello World"` as an `std::string`. If I try to instantiate it with some type that cannot be added like some `struct Foo{};`,  I get the dreaded C++ template compiler error:

```
<source>: In function 'int main()':
<source>:11:22: error: no matching function for call to 'add<Foo>(Foo, Foo)'
   11 |     auto b = add<Foo>(Foo{}, Foo{});
      |              ~~~~~~~~^~~~~~~~~~~~~~
<source>:4:6: note: candidate: 'template<class T> decltype ((a + b)) add(const T&, const T&)'
    4 | auto add(const T& a, const T& b) -> decltype(a + b) {
      |      ^~~
<source>:4:6: note:   template argument deduction/substitution failed:
<source>: In substitution of 'template<class T> decltype ((a + b)) add(const T&, const T&) [with T = Foo]':
<source>:11:22:   required from here
   11 |     auto b = add<Foo>(Foo{}, Foo{});
      |              ~~~~~~~~^~~~~~~~~~~~~~
<source>:4:48: error: no match for 'operator+' (operand types are 'const Foo' and 'const Foo')
    4 | auto add(const T& a, const T& b) -> decltype(a + b) {
      |                                              ~~^~~
```

This one isn't even *that* bad. Since C++20, I can add a concept to make the error message a little more explicit:

```cpp
template<typename T>
concept Addable = requires(const T& a, const T& b) {
    a + b; // require that this is a valid expression - not examing the return type right now
};

template<Addable T>
auto add(const T& a, const T& b) -> decltype(a + b) {
    return a + b;
}
```
```
<source>: In function 'int main()':
<source>:16:22: error: no matching function for call to 'add<Foo>(Foo, Foo)'
   16 |     auto b = add<Foo>(Foo{}, Foo{});
      |              ~~~~~~~~^~~~~~~~~~~~~~
<source>:9:6: note: candidate: 'template<class T>  requires  Addable<T> decltype ((a + b)) add(const T&, const T&)'
    9 | auto add(const T& a, const T& b) -> decltype(a + b) {
      |      ^~~
<source>:9:6: note:   template argument deduction/substitution failed:
<source>:9:6: note: constraints not satisfied
<source>: In substitution of 'template<class T>  requires  Addable<T> decltype ((a + b)) add(const T&, const T&) [with T = Foo]':
<source>:16:22:   required from here
   16 |     auto b = add<Foo>(Foo{}, Foo{});
      |              ~~~~~~~~^~~~~~~~~~~~~~
<source>:4:9:   required for the satisfaction of 'Addable<T>' [with T = Foo]
<source>:4:19:   in requirements with 'const T& a', 'const T& b' [with T = Foo]
<source>:5:7: note: the required expression '(a + b)' is invalid
    5 |     a + b;
      |     ~~^~~
```

If I wanted to ensure that adding two `T`s produces a `T`, I can specify that too:

```cpp
template<typename T>
concept Addable = requires(const T& a, const T& b) {
    { a + b } -> T;
};

template<Addable T>
auto add(const T& a, const T& b) -> decltype(a + b) {
    return a + b;
}

struct Foo {
    auto operator+(const Foo& other) const -> int;
};

// compiler error, because Foo + Foo = int
add<Foo>(Foo{}, Foo{});
```

This is, of course, reasonably possible in some other languages with a similar amount of type bounds:

```cs
T Add<T>(T a, T b) where T: IAdditionOperations<T, T, T>
{
    return a + b;
}
```

```rust
fn add<T: Add<Output = T> + Copy, U>(a: T, b: T) -> T {
    a + b
}
```

We can do any of these if the result type is not `T` too, but before .NET 7 or if `T` is not a numeric type, this would have been awkward (impossible?) in C#. What C++ also brings to the table here is that it's possible to call methods on/with and create instances of an unknown template type very easily.

```cpp
template<typename Foo>
auto use_foo(Foo&& foo) {
    Foo copy = foo;
    Foo zero = 0;
    consume_foo(std::move(foo));
    zero.bar();
}
```

In other languages, this is an error immediately. In C++, if any of these statements are invalid, that error is only reported *when the function is instantiated*. We can use concepts to make sure all of these statements are valid too. Some may scream in horror at the uncertainty of it all, but I think it's quite freeing, and opens up so many more situations in which generic programming can be employed. Personally, I prefer this - I know what kind of type I'm going to instantiate this generic with, so just let me write the damn code and have it just work. If I wanted to write one-or-more trait bounds for every single operation in this function, I'd rather just write an overload.

C# can somewhat achieve this with interfaces and some very explicit type constraints with possible polymorphism, as we saw. In Java, good luck. Rust can do this with trait bounds, and it won't even use polymorphism in that context (most of the time, but it might also use a fat pointer), but what Rust doesn't have is *specialisation* and *Turing-complete metaprogramming*.

Let's compute the Nth Fibonacci number with templates at compile time:

```cpp
template<int I>
struct Fib {
    static constexpr int value = Fib<I - 1>::value + Fib<I - 2>::value;
};

template<>
struct Fib<0> {
    static constexpr int value = 0;
};

template<>
struct Fib<1> {
    static constexpr int value = 1;
};

int main() {
    return Fib<10>::value;
}
```

And the assembly that generates?

```asm
main:
        mov     eax, 55
        ret
```

Specialisation allows the standard library to provide some very simple but extremely useful utilities, like simply checking if two types are the same:

```cpp
template<typename T1, typename T2>
struct is_same {
    static constexpr bool value = false;
};

template<typename T>
struct is_same<T, T> {
    static constexpr bool value = true;
};

template<typename T1, typename T2>
static constexpr bool is_same_v = is_same<T1, T2>::value;

is_same_v<int, int>;  // true
is_same_v<int, bool>; // false
```

Or using different associated types depending on the properties of some other type:

```cpp
template<typename T>
struct iterator {
    using inner_iterator = std::conditional_t<
        std::is_const_v<T>,
        typename std::vector<T>::const_iterator,
        typename std::vector<T>::iterator
    >;
};
```

C#, Java and even Rust require some runtime work for this. SFINAE was *ugly*, but combining these simple type properties with concepts makes for some incredibly versatile templated code with relatively nice syntax and specific compiler errors.

Templates get even better, because C++ templates also have...

## 2. Variadic Template Parameters

Variadic template parameters, a type of [pack](https://en.cppreference.com/w/cpp/language/pack), provide a way to give any amount of parameters to a template, decided at the point of instantiation. They allow the declaration of types like this:

```cpp
template<typename... Types>
struct tuple {};

tuple<> t0;                         // no types
tuple<int> t1;                      // just a single int
tuple<int, float, std::string> t2;  // int, float, and std::string or in Types
```

In other languages, you'll find common types like a tuple to be painstakingly overloaded for each amount of generic types. This also works with a single type:

```cpp
template<int... Ints>
auto do_something_with_ints(Ints... ints); // can be called with any amount of integers
```

Pack expansion (the `...` after the name of a pack) can be thought of as being rewritten by the compiler as just each element in the pack listed out. It interacts with other language features, like `using` statements and inheritance lists. In absence of proper pattern-matching, this allows us to write relatively concise visitor patterns on `std::variant` and other union-like types:

```cpp
template<typename... Lambdas>
struct Visitor : Lambdas... {
    using Lambdas::operator()...;
};

int main() {
    std::variant<int, float, std::string> v;
    const auto visitor = Visitor{
        [] (int i) { std::println("Contained an int {}", i); },
        [] (float f) { std::println("Contained a float {}", f); },
        [] (std::string_view s) { std::println("Contained a string {}", s); }
    };

    std::visit(visitor, v);
}
```

My `Visitor` struct can inherit from any amount of types, which are all function-like objects with a call operator in this case. It can be initialised as such, with the inherited call operators exposed.

An incredible feature that uses packs is the [fold expression](https://en.cppreference.com/w/cpp/language/fold). Fold expressions expand a pack, but put an operator between each element. A simple example:

```cpp
template<typename... Bools>
auto bool all(Bools... bools) -> bool {
    return (bools && ...);
}

// expands to ((true && true) && true) && false
bool b = all(true, true, true, false);
```

We can use this to write some more useful functions, like logging an unknown amount of values before C++23's `print` library:

```cpp
template<typename... Args>
auto print(Args&&... args) -> void {
    ((std::cout << args << ' '), ...) << std::endl;
}

// prints "This is a sentence 0 1 2 3 4 5 true false"
print("This", "is", "a", "sentence", 0, 1, 2, 3, 4, 5, std::boolalpha, true, false);
```

So, back to my little `add` function from the previous section. Say I wanted it to be able to add any number of values together. Easy, I'll just use an array! Java and C# even give some nice syntax for this:

```cs
T Add<T>(params T[] nums) where T: IAdditionOperators<T, T, T> {
    var sum = 0;
    foreach (var n in nums) {
        sum += n;
    }
    return sum;
}
```

Oh wait, this doesn't work, because C# doesn't know how to assign `0` to a `T`. This was not possible at all until C# *11* when they added [static abstract interface properties](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/static-abstracts-in-interfaces):

```cs
T Add<T>(params T[] nums) where T: INumber<T> {
    var sum = T.Zero;
    foreach (var n in nums) {
        sum += n;
    }
    return sum;
}

Add<int>(0, 1, 2, 3, 4, 5);
```

If `T` is not a number, or it's some standard library type we can't extend, we run into the same problem again. We've now also introduced a heap allocation for the array. Don't even get me *started* on the Java version of this. This is the best I could come up with:

```java
<T extends Number> T add(T... nums) {
    double sum = 0.0;
    for (T num : nums) {
        sum += num.doubleValue();
    }

    if (nums.length == 0) {
        if (nums[0] instanceof Integer) {
            return (T) Integer.valueOf((int) sum);
        } else if (nums[0] instanceof Double) {
            return (T) Double.valueOf(sum);
        } else if (nums[0] instanceof Float) {
            return (T) Float.valueOf((float) sum);
        } else if (nums[0] instanceof Long) {
            return (T) Long.valueOf((long) sum);
        }
    }

    return null; // ??? what do i do here
}
```

Rust does well here, we can do something like either of these:

```rust
use std::ops::Add;

fn add<T: Add<Output = T> + Copy + Default>(nums: &[T]) -> T {
    nums.iter().copied().fold(T::default(), Add::add)
}

fn add<T: AddAssign + Copy + Default>(nums: &[T]) -> T {
    let mut sum = T::default();
    for num in nums {
        sum += *num;
    }
    sum
}
```

But that's, like, a *lot* of trait bounds (`Copy + Default + L + Ratio + NoVariadics + NoConstexpr + `). The compiler will probably be able to inline a lot of the function calls, but it's a lot of noise.

In C++, I hear you ask?

```cpp
template<Addable... Args>
auto add(Args&&... nums) -> decltype((nums + ...)) {
    return (nums + ...);
}

add(0, 1, 2, 3, 4, 5);
```

That's it. That's the fold expression coming into play. No array, no heap allocation at the call site, no polymorphism or dynamic memory. This even works with non-compile-time constants and non-numeric types that support addition, like strings and even vectors, without any further annoyance.

Let's look at some less academic examples.

Let's say I have some abstract class `Base`, which has many subclasses that inherit from it. These subclasses all have unique (and possible many, overloaded) constructors that take different parameters. All instances of these classes need to be managed by, for example, a garbage collector or an arena allocator with a free list, or I just generally need to do some extra housekeeping whenever an object like this is created. I want some kind of factory function that can create any instance of any of these subclasses, using any constructor, and do the housekeeping automatically. This is trivial, with variadic template arguments!

```cpp
template<typename T>
concept SubclassOfBase = std::is_base_of_v<Base, T>;

template<SubclassOfBase T, typename... Args>
inline auto create_gc_instance(Args&&... args) -> GcObject<T> {
    T instance{ std::forward<Args>(args)... };
    // housekeeping...
    return GcObject<T>{ std::move(instance) };
}

// imagine these are all subclasses of Base, for some reason
auto person = create_gc_instance<Person>("John Smith", 32, gender::male, job::accountant);
auto dog = create_gc_instance<Dog>("Scooby Doo", 7, breed::great_dane);
auto car = create_gc_instance<Car>(model::lexus, 2025);
```

Taking `Args` by [*universal reference*](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers) and using `std::forward` ensures that if we pass a `const` reference, it remains a `const` reference, if we pass an rvalue reference, it remains an rvalue reference and can be used as a "moved-from" value (more on that later).

Programmers often find themselves in a position where they want to store instances of subclasses of some base class in a collection, or implementers of an interface. In a language that supports polymorphism, this is usually trivial - just make the generic collection of type `Base`, and store the instances as pointers. This happens implicitly in the likes of C# and Java, can be done in Rust with something like a `Box<dyn T>`, and in C++ with `std::unique_ptr<T>` or similar.

But, let's say we're in C++ and we really didn't want to use polymorphism, because we are running code in a performance-critical environment on some very limited embedded device where that isn't allowed. This embedded device doesn't even have a heap, so we know how many elements should be in our array ahead of time and what they should be, they're just not the same type exactly, but could all be described by some interface. Is such a thing possible? Yes!

```cpp
template<typename T>
concept UnaryTypeTrait = requires(T t) {
    typename T::value;
    { T::value } -> std::same_as<bool>;
};

template<template<UnaryTypeTrait> typename TypeTrait, typename... Types>
    requires((TypeTrait<Types>::value && ...))
class VariantArray {
public:    
    template<typename... Args>
    VariantArray(Args&&... args)
        : m_array { Types{std::forward<Args>(args)... }... } {}

    template<typename F>
    auto visit(F&& function) {
        for (auto& instance : m_array) {
            std::visit(function, instance);
        }
    }
    
private:
    std::array<std::variant<Types...>, sizeof...(Types)> m_array;
};
```

This ensures that every type given to the `VariantArray` in `Types` all fulfil the same [unary type trait](https://en.cppreference.com/w/cpp/named_req/UnaryTypeTrait) (a more old-school and restrictive way of doing something like our `Addable` concept from earlier - the static property `value` will be `true`/`false` depending on whether the type fulfils the type traits) by expanding the pack and checking the type trait on each one and combining it into a logical and expression, and then fills the array with a variant for each type, with each variant instance holding a instance of each of the types constructed with `args` in the order we state them in the template argument list. We can also visit each instance in the array with a function passed to the `visit` method. To use it:

```cpp
template<typename T>
concept CanFooConcept = requires(T instance) {    
    instance.foo();
};

template<typename T>
struct CanFoo {
    static constexpr bool value = CanFooConcept<T>;
};

struct Person {
    auto foo() -> void {
        std::println("Person foo!");
    }
};

struct Dog {
    auto foo() -> void {
        std::println("Dog foo!");
    }
};

struct Car {
    auto foo() -> void {
        std::println("Car foo!");
    }
};

int main() {
    VariantArray<CanFoo, Person, Dog, Car> array;
    array.visit([] (auto& el) {
        el.foo();
    });
}
```

This works and prints, in the correct order:

```
Person foo!
Dog foo!
Car foo!
```

Instead of making these types polymorphic, we can write a unary type trait to match our concept and ensure that all of the classes adhere to it. This means calling `foo()` on them in the lambda in the call to `visit()` is totally fine. The compiler can even *inline* this - this is the [assembly generated](https://godbolt.org/z/9cE3xca5c) by Clang with O3:

```asm
main:
        push    rbx
        mov     rbx, qword ptr [rip + stdout@GOTPCREL]
        mov     rdi, qword ptr [rbx]
        lea     rdx, [rip + .L.str.1]
        mov     esi, 11
        call    void std::println<>(_IO_FILE*, std::basic_format_string<char>)
        mov     rdi, qword ptr [rbx]
        lea     rdx, [rip + .L.str.32]
        mov     esi, 8
        call    void std::println<>(_IO_FILE*, std::basic_format_string<char>)
        mov     rdi, qword ptr [rbx]
        lea     rdx, [rip + .L.str.33]
        mov     esi, 8
        call    void std::println<>(_IO_FILE*, std::basic_format_string<char>)
        xor     eax, eax
        pop     rbx
        ret
```

The `VariantArray` class, `std::variant`, our structs - they don't even exist. It's just three calls to `std::println()`. Outside of the printing, we're not even using the heap.

Concepts, like Rust's traits, allow us to bring the type guarantees of interfaces to compile time checks with no polymorphism required. But, they allow both further freedom and further control - we can ensure that a bunch of types all have some method `foo()`, but not all the methods need even return the same type. In the same concept, we can ensure that the type is default constructible, constructible from some other type, that some free function `bar()` is callable with an rvalue reference to the type and some other parameters and is `noexcept` and returns a function that is `noexcept`, and that the type has an associated `typedef` that is `const`:

```cpp
template<typename T, typename U, typename... BarArgs>
concept HasFooAndAllTheOtherStuffISaid = requires(T value) {
    value.foo();
    typename T::some_typedef;
    std::is_const<T::some_typedef>;
} && std::is_default_constructible_v<T>
  && std::constructible_from<T, U>
  && std::is_nothrow_invocable_v<decltype(bar), T&&, BarArgs...>
  && std::is_nothrow_invocable_v<std::invoke_result_t<decltype(bar), T&&, BarArgs...>>;
```

This power, combined with variadic templates and the additional freedoms of templates, makes for some real spice.

## 3. Value Categories

Every expression in C++ has, as well as its type, its [value category](https://en.cppreference.com/w/cpp/language/value_category). This is a pretty complicated aspect of the language, and not one that the average programmer needs to know in excruciating detail, but it opens up some pretty neat possibilities.

You see, it's quite simple - every value is either a glvalue, an xvalue, or a prvalue. An xvalue is a certain kind of glvalue. An lvalue is a glvalue that is not an xvalue, and an rvalue is either a prvalue or an xvalue.

The details here are exhausting, but fortunately, the vast majority of the time you only care (or more specifically, people only talk about) if something is an lvalue or an rvalue. You can think of the L and R as standing for *locatable* and *readable*. An lvalue is *locatable*, i.e. it has a place in memory, or references something that does - a variable, a function, a reference to a variable, an expression that produces a reference to a variable, etc. An rvalue is *readable*, but not necessarily locatable - a value literal, the return value of a function that returns a value before it's assigned anywhere, a moved-from value. This distinction allows us to be very particular about how our data is used:

```cpp
// modify a string, and let the caller observe the modifications  - this needs to be an lvalue reference
auto modify_string(std::string& s);

// reads from a string, but doesn't take ownership or modify. lvalues or rvalues can bind to a const reference
auto read_string(const std::string& s)

// take ownership of the string - this is the default behaviour of rust. Can only be called with a rvalue reference, or "moved-from" string
auto consume_string(std::string&& s);

// take a string by value - if called with an lvalue, it gets copied. If called with an rvalue, a new string is move-constructed, or takes ownership for all intents and purposes
auto copy_string(std::string s);
```

But the *really* fine level of control that we get, is that we can specify different member function overloads to only be callable on a certain value category.

Here's a stripped back version of `std::optional` and it's value getter methods:

```cpp
struct nullopt_t {};

template<typename T>
class optional {
public:
    template<typename U>
    optional(U&& value)
        : m_value{ std::forward<U>(value) }
        , m_has_value{ true } {}

    optional()
        : m_null{}
        , m_has_value{ false } {}

    auto value() & -> T& {
        assert(m_has_value);
        return m_value;
    }
	
    auto value() const& -> const T& {
        assert(m_has_value);
        return m_value;
    }

    auto value() && -> T&& {
        assert(m_has_value);
        return std::move(m_value);
    }

    auto value() const && -> const T&& {
        assert(m_has_value);
        return std::move(m_value);
    }

private:
    union {
        T m_value;
        nullopt_t m_null;
    };
    bool m_has_value;
};
```

What the `&`, `const &`, `&&`, and `const &&` after the function name but before the trailing return type mean is that each of those methods are only callable on non-`const` lvalue references, `const` lvalue references, non-`const` rvalue references, and `const` rvalue references, respectively. If I have an `std::optional<std::string>` as a variable and I just call `.value()` on it, I get a regular reference because the variable is an lvalue. But, if that `std::optional` was an rvalue, I would get an rvalue reference - i.e., ownership gets transferred.

```cpp
struct Foo {
    Foo() = default;

    Foo(const Foo&) {
        std::println("Copied");
    }

    Foo(Foo&&) {
        std::println("Moved");
    }
};

auto make_optional() -> std::optional<Foo> {
    return std::make_optional<Foo>();
}

int main() {
    std::optional<Foo> o1;
    o1.emplace();

    [[maybe_unused]] auto copy = o1.value();

    std::println("===");

    [[maybe_unused]] auto moved = make_optional().value();
}
```

The output of this program is:

```
Copied
===
Moved
```

The first one is copied, as we would expect. We requested a copy, and our lvalue is left alone. The second one, however, is moved, and we get only a single move construction. By calling `.value()` on an `std::optional` that *is* an rvalue, it transfers the ownership of the held value straight out.
## 4. `std::initializer_list` and Implicit Conversions

So, this is a controversial one, but it's short.

To be clear, when I say "implicit conversions", what I mean is single-argument constructors of types that are allowed to be implicitly callable for convenience, like with `std::optional`:

```cpp
auto do_stuff() -> std::optional<std::string> {
    std::string s;
    // do some stuff to build out s
    return s;
}
```

The `std::optional` can be *implicitly* constructed from the string, and this is a case where named return value optimisation (NRVO, an element of [copy elision](https://en.cppreference.com/w/cpp/language/copy_elision)) can kick in too.

I do think that constructors should be marked `explicit` where possible to avoid subtle bugs and overload ambiguity when this kind of convenience isn't required. I also *hate* that without warnings turned up, so many implicit conversions between built-in types are possible (nobody wants to `malloc(-1)` - just ask [Sony](https://www.youtube.com/watch?v=rWCvk4KZuV4)). I always work with `-Wall -Wextra -Wpedantic -Wconversion -Werror`, as should everybody, and if that was the default all along, C++'s name wouldn't be as tarnished.

`std::initializer_list` is also potentially problematic. I don't like how it looks like an STL collection but actually only works through compiler magic, I don't like how it's elements are `const` so [can't be moved-from](https://tristanbrindle.com/posts/beware-copies-initializer-list), I don't like how it plays weirdly with constructor overload resolution when using braced initialisers and produces ugly error messages.

*But*, the combination of `std::initializer_list` and implicit single-argument constructor calls can make for incredibly expressive code. Take a look at this example of constructing a JSON object from the [nlohmann JSON library](https://github.com/nlohmann/json/tree/develop):

```cpp
json j{
    {"pi", 3.141},
    {"happy", true},
    {"name", "Niels"},
    {"nothing", nullptr},
    {"answer", {
        {"everything", 42}
    }},
    {"list", {1, 0, 2}},
    {"object", {
        {"currency", "USD"},
        {"value", 42.99}
    }}
};
```

That's *C++*, not JavaScript, and it's not even hard to implement. This works because the constructors for their JSON objects take initializer lists of other JSON objects, all of which can be constructed implicitly from values. In other languages, this would be littered with type names and dots and colons and `new`s.
## 5. `constexpr`

I'll keep this one brief too, because it's well known, and to be honest I don't have the knowledge to properly compare this to other languages' compile-time abilities, like Zig's `comptime` which I believe is very good. But, it's worth mentioning because it's cool!

First of all, our Fibonacci calculation from earlier can be rewritten with normal person code, but still executed at compile-time with `constexpr`:

```cpp
constexpr int fib(int n) {
    if (n < 2) {
        return n;
    }

    return fib(n - 1) + fib(n - 2);
}

int main() {
    constexpr int f = fib(10);
    return f;
}
```

Even in debug mode, this basically just compiles to moving 55 into `eax` and returning, because we are forcing this function to run at compile time. This of course works with much more complex programs as well, even including strings and collections. What if I was doing some Leetcode easies and I wanted to find the length of the longest word in a sentence?

```cpp
using namespace std::string_view_literals;

constexpr auto longest_word(std::string_view sentence) noexcept {
    return std::ranges::max(
        std::views::split(sentence, " "sv)
        | std::views::transform([] (const auto& split) {
            return split.size();
        })
    );
}

int main() {
    constexpr auto longest_length = longest_word("This is a sentence"sv);
    return static_cast<int>(longest_length);
}
```

Again, this literally compiles to:

```asm
main:
        mov     eax, 8
        ret
```

Beats 100%?

Something quite cool about `constexpr` is that undefined behaviour *cannot* happen during compile-time execution, so running code as `constexpr` can uncover UB-related bugs:

```cpp
constexpr auto sum_n(int start, int n) -> int {
    auto iota = std::views::iota(start, start + n);
    return std::accumulate(iota.begin(), iota.end(), 0);
}

int main() {
    constexpr int sum = sum_n(std::numeric_limits<int>::max(), 10);
    return sum;
}
```

Since signed integer overflow is undefined behaviour, this reports the appropriate error:

```
<source>:13:47: note: value 2147483657 is outside the range of representable values of type 'int'
   13 |     auto iota = std::views::iota(start, start + n);
      |                                               ^
<source>:18:25: note: in call to 'sum_n(2147483647, 10)'
   18 |     constexpr int sum = sum_n(std::numeric_limits<int>::max(), 10);
      |  
```
## Conclusion

This is not meant to convince anybody to rewrite their Rust project in C++, or to be an armed response to Bjarne's [call to action](https://www.msn.com/en-us/public-safety-and-emergencies/general/c-creator-calls-for-help-to-defend-programming-language-from-serious-attacks/ar-AA1A60Mj) to defend the language. This is just a display of some of the cool features of C++ that, as far as I know, don't really exist elsewhere, and that, even if they aren't useful in most situations, make me go *wow, that's __cool__*.

Yes, the memory safety is an issue. Quite a lot of the concerns here can be alleviated with `-Wall -Wextra -Wpedantic -Wconversion -Werror` and some extra `W`s, using LLVM's sanitisers, placing restrictions on yourself like not using raw pointers, etc., but an opt-in approach is never going to fully work. Yes, the refusal to break ABIs is annoying. Yes, the tooling kind of sucks (to be clear, all the different tools (IDEs, compilers, toolchains, package managers etc.) are very good *in and of themselves*, it's just that there's so many and everything's different on different platforms) (CMake is really good actually, you just have a skill issue). Header files suck ass and nobody has really implemented modules yet.

I wish safety features and `const`-correctness were on by default. I wish we had pattern matching. I wish we had Rust's enums. I wish we had *one* package manager that everybody used that just ~~fucking~~ worked. I wish we had a standard and reference implementation with a development cycle like C#/.NET's.

Other languages are completely appropriate, and even preferrable, to use in many situations, but I'll always come back to C++ for my own projects. Because I'll miss these features that make me say *__cool__*.
