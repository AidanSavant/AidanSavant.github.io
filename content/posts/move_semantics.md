+++
title = "Move semantics in C++" 
date  = "2025-05-22"
tags  = ["C++", "Move semantics"]
+++

Small introduction to move semantics in C++
<!--more-->

# Introduction
C++ is a systems language with an affinity for performance, and it does a pretty damn good job of achieving this. 
However, there was a grey area before C++11 that was hard for compiler developers to address **efficiently**. 
I'm referring to the construction of temporary objects causing unnecessary copying of, possibly, 
large [objects](https://eel.is/c++draft/intro.object).
(NOTE: C++ objects, not OO objects. Completely different things) 

C++ compilers later implemented a few ways to address this problem, namely through
compiler optimizations such as [RVO](https://sigcpp.github.io/2020/06/08/return-value-optimization) 
and [copy elision](https://en.cppreference.com/w/cpp/language/copy_elision).

However, this only addressed a subset of scenarios where a copy could be avoided. 
Move semantics was the solution implemented that addressed these set of scenarios.

So... what exactly are [move semantics](https://www.cprogramming.com/c++11/rvalue-references-and-move-semantics-in-c++11.html)?
Move semantics are a set of optimization techniques implemented in C++11 to efficiently *move* around objects 
without the unnecessary need of copying them. This is done through transferring **ownership** of an object.
Furthermore, if something takes ownership of an object, then the state of that object outside of what owns it, 
is **invalid**.

# How?
C++ implemented the notion of moving objects through adapting what's known as [value categories](https://en.cppreference.com/w/cpp/language/value_category).
Value categories are a property of expressions that are notate notation some information. 
about said expression. In their simplest form, there are 2 **main** categories: [lvalues](https://en.cppreference.com/w/cpp/language/value_category#lvalue) and [rvalues](https://en.cppreference.com/w/cpp/language/value_category#rvalue) 
(and subsequent reference forms of such categories). -- Technically, there are more categories that
are more or less broad such as: glvalues, prvalues, and xvalues.

C++14 introduced **rvalue references** (and [xvalues](https://en.cppreference.com/w/cpp/language/value_category#xvalue),
but we'll go over that later) value category as a way to implement move semantics. Let's define what ``lvalues`` and ``rvalues`` are.

## Lvalues
Generally, **lvalues** are **identifiable** (can be addressed in memory and non-temporary) expressions. Archaically,
they're typically referred to as what you find on the "left-hand side" of an assignment.

Let's look at some examples:
```cpp
// 'x' is an lvalue, it has an identity (addressible in memory and exists outside of the evaluation context)
int x = 15;

// 'xs' AND xs[0] are both lvalues, both have an identity.
int xs[] = {1, 2, 3};
xs[0] += 1;
```

A slightly more confusing example would be:
```c++
static int x = 5;
int& f() { return x; }

// f() is an lvalue
f() = 15;

// "x" is now 15
```

## Rvalues
Generally, **rvalues** are expressions that **aren't lvalues**. Therefore, ``rvalues`` are simply value that have no 
``identity`` (``not addressible`` in memory and are temporary`` relative to the evaluation context). Subsequently, 
they are also archaically referred to as what's found on the "right-hand side" of an assignment.

Let's look at some examples:
```cpp
// 15 is an rvalue, it's temporary and aren't identifiable
int x = 15;

// This further proves that "15" is a temporary, because we cannot assign it as if it had an identity.
15 = x; // ERROR
```

Let's look at a little tricky example:
```cpp
int x = 15;
int y = x; // what is the value category of 'x'?
```
``x`` is an ``lvalue``, but we put it in place of an ``rvalue``??? How does that work? ``x`` 
went under a process called [lvalue-to-rvalue conversion](https://eel.is/c++draft/conv.lval). 
This is the process of implicitly replacing an ``lvalue`` with an ``rvalue``.

It's the same thing for something like this:
```cpp
int x = f(); // implicitly convert lvalue "f()" to it's rvalue return value 
```

## Rvalue references
With that out of the way, let's talk about 
[rvalue references](https://learn.microsoft.com/en-us/cpp/cpp/rvalue-reference-declarator-amp-amp?view=msvc-170). 
Just like lvalue references, rvalue reference some rvalue. Rvalue references are denoted with a double ``&&``.

Let's look at an example:
```cpp
void f(int&& x) {}

f(5); // OK, binds to a temporary rvalue ("5" in this case)
    
int x = 5;
f(x); // ERROR, tries to bind to a non-temporary lvalue ("x" in this case)
```

As you can see ``f`` can only accept temporary values because of the ``&&``. However, there's another way to accept a 
temporary value e.g:
```cpp
void f(const int& x) {}
```

this will also accept temporaries AND non-temporary values. The reason why this is okay is because C++ will extend the
lifetime of the temporary to last for the duration of the function. It's considered 
[the most important const](https://herbsutter.com/2008/01/01/gotw-88-a-candidate-for-the-most-important-const/) by 
Andrei Alexandrescu, author of "Modern C++ Design" and "C++ Coding Standards".


So, how is this used to transfer ownership of objects? We use them in a special kind of constructor 
and assignment operator known as the "move constructor" and "move assignment operator", respectively.

Let's look at an example:
```cpp
class A {
public:
    A() = default;

    // Accepts a temporary "A" value
    A(A&&) { std::cout << "A(A&&) moved constructor called"; }
};

// Calls A(A&&) (only when you specify -fno-elide-constructors otherwise C++ will elide the temporary)
A a{A()}; // prints: "A(A&&) move constructor called" but only with that compiler flag
```
So this will only call the move ctor on the temporary if we explicitly tell the compiler to not
[elide](https://en.cppreference.com/w/cpp/language/copy_elision) the temporary with an in-place 
construction. 

# std::move
C++ gives us a nice utility to convert any reference into what seems like an rvalue reference using 
[std::move](https://en.cppreference.com/w/cpp/utility/move).Except that's **wrong** :), 
it doesn't convert into an rvalue reference, but rather an [xvalue](https://en.cppreference.com/w/cpp/language/value_category#xvalue).


An ``xvalue`` is a value that is identifiable and movable. Let's look at what our code would look like 
with ``std::move``:
```cpp
class A {
public:
    A() = default;
    A(A&&) { std::cout << "A(A&&) moved constructor called"; }
};

A a = A();
A b = std::move(b); // std::move(A()) also works 
```
So the code now calls ``std::move(b)`` and converts ``b`` into a ``A&&``. ``std::move`` is essentially sugar syntax for a 
``static_cast``.

Another place we use the ``&&`` is for move assignments to reassign already existing objects without creating
temporaries. An example with our previous class would be:
```cpp
class A {
public:
    A() = default;
    A& operator=(A&&) { std::cout << "operator=(A&&) called"; return *this; }
};

A a = A();
A b = A();

a = std::move(b); // prints: "operator=(A&&) called"
```

## Practical Example
Let's write a very small ``std::vector`` that implements move semantics. I'll also introduce you to using
[std::exchange](https://en.cppreference.com/w/cpp/utility/exchange), a very useful function for writing clean move 
constructors and move assignment operators.

```cpp
template<typename T>
class MyVector {
public:
    using ValuePtr = T*;


    /* === Basic ctors === */
    MyVector() = default;
    explicit MyVector(const std::size_t size) :
        m_cap(size),
        m_size(size),
        m_storage(new T[size])
    {
        std::fill(begin(), end(), T())
    }

    /* === Insertion function === */
    void push(T value) {
        if(m_size <= m_cap) {
            // call resize function, omitted for simplicity
        }

        data[m_size++] = value;
    }


    /* === Move semantics === */
    
    // Move constructor
    MyVector(MyVector&& other)
    noexcept(std::is_nothrow_move_constructible_v<decltype(m_storage)>) : // Just in case we change the storage type
        m_cap(std::exchange(other.m_cap, 0)),
        m_len(std::exchange(other.m_len, 0)),
        m_storage(std::exchange(other.m_storage, nullptr))
    {}

    // Move assigmment
    MyVector& operator=(MyVector&& other)
    noexcept(std::is_nothrow_move_constructible_v<decltype(m_storage)>) {
        if(this == other) {
            return *this; // Self-assignment
        }

        delete[] data;

        // Take ownership of "other"'s resources
        m_cap     = std::exchange(other.m_cap, 0);
        m_size    = std::exchange(other.m_size, 0);
        m_storage = std::exchange(other.m_storage, nullptr);
    }
 
    // Not really important, just useful
    ValuePtr begin() const { return m_storage; }
    ValuePtr end()   const { return m_storage + m_size; }
    
private:
    T *m_storage;
    std::size_t m_size, m_cap;
};

MyVector expensive_task(const std::size_t N) {
    MyVector vec(N);

    for(std::size_t i = 0; i < N; ++i) {
        vec.push(n);
    }

    return vec; // move constructor called, will be cheap whenever N is large
}
```
As you can see from our custom vector implementation, whenever we return ``vec`` there's no need to
create a copy of the return that you'd have if you didn't have a move constructor.

## Forwarding references
In the previous section we learned we use the ``&&`` to notate that we want to do a "move" operation. 
Could we then write generic functions that can accept any rvalue reference?

Something like this?
```cpp
template<typename T>
void f(T&&) {}

f(std::move(some_object)); // OK

int a = 15;
f(a); // NOT OK? MAYBE?
```
Not exactly... that ``f(a)`` is actually valid. ``T`` is what's known as a [forwarding references / universal references](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers)
(love you scott meyers). It's a reference that can either bind to an lvalue or rvalue depending on the arguments we
give it. However, it has this one cave:

Imagine we take the same function but instead we call another function that has two overloads: an lvalue
reference overload and an rvalue (const T&) overload:
```cpp
template<typename T>
void g(T& x) {}

template<typename T>
void g(const T& x) {}

template<typename T>
void f(T&& x) { g(x); }

f(15);
```
This code doesn't do what you expect it to. You expect ``f`` to call ``g(const T&)``, but it actually
calls ``g(T&)``. This is due to something called [reference collapsing](https://www.ibm.com/docs/en/xl-c-and-cpp-aix/16.1.0?topic=operators-reference-collapsing-c11).

## Reference collapsing
Reference collapsing is a set of rules that sets when we combine reference types it'll follow these forms:
```
T&& + T&  => T&
T&  + T&& => T&
T&& + T&  => T&
T&& + T&& => T&&
```
The purpose of these rules is to implement [perfect forwarding](https://www.justsoftwaresolutions.co.uk/cplusplus/rvalue_references_and_perfect_forwarding.html).

## Perfect forwarding
Perfect forwarding is the process of maintaining an expressions value category between function calls.

This is done through another utility function called [std::forward](https://en.cppreference.com/w/cpp/utility/forward).
To properly maintain the value category between function calls to resolve the correct overloads, 
we have to change the body of ``f`` to:
```cpp
template<typename T>
void f(T&& x) { g(std::forward<T>(x)); }
```
(NOTE: ``std::forward`` is also some syntactical sugar for a ``static_cast``, much like ``std::move`` is)

This updated ``f`` function now properly calls the correct overloads.

# Revisions
Somewhat still a WIP, some stuff is probably incorrect or spelled incorrectly. Please email me if you find anything 
/ think I should add stuff.

