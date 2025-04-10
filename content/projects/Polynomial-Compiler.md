+++
title = "Polynomial Compiler"
date  = "2025-04-02"
tags  = ["CS theory", "Compilers", "Algorithms", "Low-level"]
description = "Basic polynomial expression compiler (supports logical expressions)"
+++

<!-- Compiler for expressions to x86-64 written in Java. WIP. -->

# Introduction

Simple compiler for polynomial expressions and logical expressions.
* (1 + 5) * 2 / 12
* ((1 + 5) + 2) + ~(5 + 15)
* (1 & 2 | 10 + 5 * 2) + ~(1 | 0 & 5)
* ... etc

it also supports error diagnostics at the lexer and parser phases, so it can detect
ill-formed expressions, e.g:
* (1 + 5
* ++ 5
* 1 +
* )(1
* ... etc

and invalid characters, e.g:
* 1^
* \1
* a = 5 (soon to be supported :P)

# Lexical analysis
...

# Syntax analysis
...

# Code generation
...

# TODO
* Add support for compiling files      | ❌
* Add Support for better phase errors  | ❌
* Add Support for variable bindings    | ❌
* Add Support for function definitions | ❌
* Add Support for LLVM bindings for optimization passes | ❌

