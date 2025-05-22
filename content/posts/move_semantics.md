+++
title = "Move semantics in C++" 
date  = "2025-05-22"
tags  = ["C++", "Move semantics"]
+++

Small introduction to move semantics in C++. WIP.
<!--more-->

# Introduction
C++ is a systems language designed for performance, and it does a pretty damn good 
job of achieving this. However, there was a grey area before C++11 that was hard for 
compiler developers to address **efficiently**. I'm referring to the construction of 
temporary objects causing ``unnecessary`` copying of, possibly, ``large`` objects. 

C++ compilers later implemented a few ways to address this problem, namely through
compiler optimizations such as ``RVO`` (return value optimization) and ``copy elision``.
However, this only addressed a subset of scenarios where a copy could be avoided. 
Move semantics was the solution implemented that addressed these set of scenarios.

So... what exactly are move semantics? Move semantics are a set of optimization 
techniques implemented in C++11 to ``efficiently`` *move* around objects without 
the ``unnecessary`` need of copying them. This is done through transferring **ownership** of an object.
Furthermore, if something takes ownership of an object, then the state of that object outside of what owns it, is ``invalid``.

# How?
C++ implemented the notion of moving objects through adapting what's known as [value categories](https://en.cppreference.com/w/cpp/language/value_category).
Value categories are a property of expressions that are notate notation some information. 
about said expression. In their simplest form, there are 2 **main** categories: lvalues and rvalues 
(and subsequent reference forms of such categories). -- Technically, there are more categories that
are more or less broad, but most of which can be encapsulated by ``lvalues`` and ``rvalues``.

C++14 introduced the ``rvalue reference`` (and ``xvalues``, we'll go over that later) value category as a way to implement 
move semantics. Let's define what ``lvalues`` and ``rvalues`` are.

## Lvalues
Generally, ``lvalues`` are ``identifiable`` (can be addressed in memory and non-temporary) expressions. Archaically,
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
Generally, ``rvalues``` are expressions that **aren't** lvalues. Therefore, ``rvalues``` are simply value that have no ``identity`` 
(``not addressible``` in memory and are ``temporary`` relative to the evaluation context). Subsequently, they are archaically
referred to as what's found on the "right-hand side" of an assignment.

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
``x`` is an ``lvalue``, but we put it in place of an ``rvalue``??? How does that work? ``x`` went under a process called
``lvalue-to-rvalue conversion``. This is the process of implicitly replacing an ``lvalue`` with an associated ``rvalue``.

It's the same thing for something like this:
```cpp
int x = f(); // implicitly convert lvalue "f()" to it's rvalue return value 
```

## Rvalue references


