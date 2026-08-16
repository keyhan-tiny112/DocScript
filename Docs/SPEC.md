---
Version: 0.2
Status: Stable Draft
---
# DocScript Language Specification

This document defines the formal syntax and semantic rules of the DocScript documentation language.

---

## 1. Introduction

### 1.1 What DocScript Is

DocScript is a **pseudocode-style documentation language**. Its goal is to describe the structure, behavior, input/output, and error conditions of a software system using syntax that visually resembles code.

### 1.2 Design Principles

- DocScript **is not an executable language.** No compiler or interpreter is required for it.
- DocScript is **programming-language independent**; it can be used to document code written in any language.
- Each line or block is considered an independent **statement**, unless it is combined using nesting notation (section 7).
- This document defines the syntax using pseudo-EBNF notation. Implementing an official parser or Validator is optional; section 12 provides guidance for implementing such a tool.

---

## 2. Core Objects

DocScript has three core object types: function, class, and variable.

```ebnf
object      ::= function | class | variable
function    ::= "f(" identifier ")"
class       ::= "c(" identifier ")"
variable    ::= "v(" identifier ")"
identifier  ::= letter { letter | digit | "_" }
```

| Symbol | Meaning |
|------|------|
| `f(name)` | function |
| `c(name)` | class |
| `v(name)` | variable |

---

## 3. Object Chain

Any object may be followed by one or more other objects to form a **chain**. A chain represents a nested path, such as a namespace path; the final member of the chain is the primary target of the description or operation.

```ebnf
chain ::= object { object }
```

### 3.1 Uses of a Chain

A chain can represent two different semantic relationships:

**Membership:** When the first member is a class, subsequent members belong to it.

```
c(App)c(Router)f(add_get)
```

This means `add_get` belongs to `Router`, which itself belongs to `App`.

**Composition/Output:** When the first member is a function or variable, the next member is considered an output or related value.

```
f(login)v(user)
```

This means that after calling `login`, a value named `user` is available.

> **Rule:** The relationship type is determined by the type of the first object in the chain, not by a separate syntax symbol. If the first object is of type `c()`, the relationship is membership; otherwise, it is composition/output.

---

## 4. Description, Output, and Their Combination

There are three operators for attaching information to a chain: description (`-`), output (`->`), and error/side effect (`!-`).

```ebnf
description   ::= chain "-" text
output        ::= chain "->" type
output_desc   ::= chain "->" type "-" text
error_desc    ::= chain "!-" text
```

| Syntax | Meaning |
|-----|------|
| `chain-text` | simple description |
| `chain->type` | output type |
| `chain->type-text` | output type with description |
| `chain!-text` | error or side effect (section 8) |

Examples:

```
f(foo)-a simple example function
f(foo)->str
f(foo)->str-a simple example function
c(Router)!-shuts down the visual server and exits
```

> **Detection rule (for Validators):** The `->` operator must be searched for before an independent `-`; in other words, a line should first be checked for the `->` pattern, then for a `-` that is not immediately preceded by `>`.

---

## 5. Types

The type system in DocScript is open; any identifier can be used as a type.

```ebnf
type ::= identifier [ "(" type {"," type} ")" ]
```

Examples: `str`, `int`, `Router`, `list(str)`, `dict(str, int)`, `optional(User)`

---

## 6. Control Structures

```ebnf
if_stmt     ::= "if" condition "->" body
for_stmt    ::= "for" identifier "in" iterable "->" body
while_stmt  ::= "while" condition "->" body
use_stmt    ::= "use" chain

condition   ::= use_stmt | expression
body        ::= text | statement
statement   ::= if_stmt | for_stmt | while_stmt | use_stmt
              | description | output | output_desc | error_desc
```

The `body` of a control structure may be free-form text or another formal statement, including a nested control structure.

Examples:

```
if use c(Router)f(close) -> close all open routes
if use f(login) -> if use f(getUser) -> return v(user)
```

---

## 7. Nesting

Control structures can be written in two forms:

### 7.1 Inline Form

Suitable for shallow nesting, usually one or two levels:

```
if use f(login) -> if use f(getUser) -> return v(user)
```

### 7.2 Indent-Based Block Form

Suitable for deeper nesting or higher readability:

```
if use f(login) ->
    if use f(getUser) ->
        return v(user)
```

```ebnf
body ::= text | statement | NEWLINE INDENT statement DEDENT
```

> **Style guide:** For bodies with more than one level of nesting, the indented form is recommended. This is a style rule, not a grammar restriction.

---

## 8. Error & Side Effect

The `!-` symbol is used both to describe behavior under error conditions and to describe automatic side effects.

```ebnf
error_desc ::= chain "!-" text
```

Examples:

```
c(Router)!-shuts down the visual server and exits
f(connect)!-raises ConnectionError if the host is unreachable
```

### 8.1 Limitation: One `!-` per Statement

Each statement may contain only one `error_desc`. If an object has multiple error or side-effect scenarios, each scenario must be written as a separate statement, on a separate line.

```ebnf
statement ::= ... | error_desc     ; at most one error_desc per statement
```

**Correct:**

```
c(AuthService)f(refresh)!-raises TokenExpired if refresh_token has expired
c(AuthService)f(refresh)!-shuts down the current session and requires re-login
```

**Incorrect:**

```
c(AuthService)f(refresh)!-raises TokenExpired if refresh_token has expired!-shuts down the current session and requires re-login
```

---

## 9. Async

```ebnf
async_marker ::= "async" chain
```

Example:

```
async f(fetch)-fetches a resource asynchronously
```

---

## 10. Parameters

Parameters of a function or class are written immediately after it, inside a separate pair of parentheses.

```ebnf
params      ::= "(" [param {"," param}] ")"
param       ::= identifier ":" type
chain_head  ::= object [params]
chain       ::= chain_head { object [params] }
```

Example:

```
f(add_get)(path:str, handler:func)->bool-adds a new GET route
c(Router)(base_path:str)-creates a new router with a base path
```

### 10.1 Rule: Parameters End the Chain

When parameters are defined on an object in a chain, that object is considered the **final target** of the statement, and the chain ends there. After an object with parameters, no other member may be added to the chain.

```ebnf
chain ::= object { object } | object params
```

**Correct:**

```
c(Router)(base_path:str)-creates a new router with a base path
```

**Incorrect:**

```
c(Router)(base_path:str)f(create_router)-creates a new router with a base path
```

To describe a method with parameters, the parameters must be placed on the method itself:

```
c(Router)f(create_router)(base_path:str)-creates a new router with a base path
```

---

## 11. Complete Grammar

```ebnf
statement    ::= async_marker | if_stmt | for_stmt | while_stmt
               | use_stmt | description | output | output_desc | error_desc

async_marker ::= "async" chain

if_stmt      ::= "if" condition "->" body
for_stmt     ::= "for" identifier "in" iterable "->" body
while_stmt   ::= "while" condition "->" body
use_stmt     ::= "use" chain

condition    ::= use_stmt | expression
body         ::= text | statement | NEWLINE INDENT statement DEDENT

description  ::= chain "-" text
output       ::= chain "->" type
output_desc  ::= chain "->" type "-" text
error_desc   ::= chain "!-" text

chain        ::= chain_head { object [params] }
chain_head   ::= object [params]

object       ::= function | class | variable
function     ::= "f(" identifier ")"
class        ::= "c(" identifier ")"
variable     ::= "v(" identifier ")"

params       ::= "(" [param {"," param}] ")"
param        ::= identifier ":" type
type         ::= identifier [ "(" type {"," type} ")" ]

identifier   ::= letter { letter | digit | "_" }
```

---

## 12. Validator Implementation Guide (Optional)

DocScript is designed to be used without any tools. If you need to build a Validator or Linter, the recommended parsing order for a statement is:

1. Check for `async` at the beginning of the statement.
2. Check for control-structure keywords (`if`, `for`, `while`, `use`) at the beginning of the statement.
3. Search for the object chain (`chain`) by separating `object` and `params` according to section 10.
4. Search for the `->` operator before an independent `-`, to distinguish `output` from `description`.
5. Search for the `!-` operator and enforce the "one `!-` per statement" limitation (section 8.1).
6. Enforce the "parameters end the chain" rule (section 10.1) to reject invalid chains.
