---
Version: 0.3
Status: Stable Draft
---
# DocScript Language Specification

This document defines the formal syntax and semantic rules of the DocScript documentation language.

## Table of contents

- [1) Introduction](#1-introduction)
  - [1.1) What DocScript Is](#11-what-docscript-is)
  - [1.2) Design Principles](#12-design-principles)
- [2) Comments](#2-comments)
  - [2.1) Rules](#21-rules)
  - [2.2) Examples](#22-examples)
- [3) Core Objects](#3-core-objects)
- [4) Object Chain](#4-object-chain)
  - [4.1) Uses of a Chain](#41-uses-of-a-chain)
- [5) Description, Output, and Their Combination](#5-description-output-and-their-combination)
  - [5.1) Referring to a Specific Parameter](#51-referring-to-a-specific-parameter)
  - [5.2) Constraint: `p()` Cannot Take Its Own Parameters](#52-constraint-p-cannot-take-its-own-parameters)
- [6) Types](#6-types)
- [7) Control Structures](#7-control-structures)
- [8) Nesting](#8-nesting)
  - [8.1) Inline Form](#81-inline-form)
  - [8.2) Indent-Based Block Form](#82-indent-based-block-form)
- [9) Error & Side Effect](#9-error--side-effect)
  - [9.1) Limitation: One `!-` per Statement](#91-limitation-one---per-statement)
- [10) Async](#10-async)
- [11) Parameters](#11-parameters)
  - [11.1) Rule: Parameters End the Chain](#111-rule-parameters-end-the-chain)
  - [11.2) Constraint: `v()` Cannot Take Parameters](#112-constraint-v-cannot-take-parameters)
- [12) Complete Grammar](#12-complete-grammar)
- [13) Validator Implementation Guide (Optional)](#13-validator-implementation-guide-optional)

---

## 1. Introduction

### 1.1 What DocScript Is

DocScript is a **pseudocode-style documentation language**. Its goal is to describe the structure, behavior, input/output, and error conditions of a software system using syntax that visually resembles code.

### 1.2 Design Principles

- DocScript **is not an executable language.** No compiler or interpreter is required for it.
- DocScript is **programming-language independent**; it can be used to document code written in any language.
- Each line or block is considered an independent **statement**, unless it is combined using nesting notation (section 8).
- This document defines the syntax using pseudo-EBNF notation. Implementing an official parser or Validator is optional; section 13 provides guidance for implementing such a tool.

---

## 2. Comments

DocScript supports single-line comments, used to annotate a DocScript document without the annotation being treated as part of the language's syntax.

```ebnf
comment ::= "#" { any_char_except_newline }
```

### 2.1 Rules

- A comment starts with `#` and extends to the end of the line.
- A comment may appear alone on its own line, or after a statement on the same line (a **trailing comment**).
- A comment carries no syntactic meaning. It is not part of any chain, description, output, or control structure, and must be discarded before a statement is processed (see the parsing order in section 13).
- Comments do not nest, and there is no multi-line or block comment form. Each `#` only affects the remainder of that single line.
- Everything between `#` and the end of the line is free-form text; no DocScript syntax inside a comment is interpreted.

### 2.2 Examples

A comment on its own line:

```
# schema in example.org
c(Order)f(checkout)->map
```

A trailing comment after a statement:

```
c(Order)f(checkout)->map    # schema in example.org
```

A comment used as an inline note before a more complex statement:

```
# TODO: document the retry behavior of this method
async c(Payment)f(charge)(amount:float, method:PaymentMethod)->bool-charges the customer asynchronously
```

---

## 3. Core Objects

DocScript has four core object types: function, class, variable, and parameter reference.

```ebnf
object      ::= function | class | variable | param_ref
function    ::= "f(" identifier ")"
class       ::= "c(" identifier ")"
variable    ::= "v(" identifier ")"
param_ref   ::= "p(" identifier ")"
identifier  ::= letter { letter | digit | "_" }
```

| Symbol | Meaning |
|------|------|
| `f(name)` | function |
| `c(name)` | class |
| `v(name)` | variable |
| `p(name)` | parameter reference (see section 5.1) |

---

## 4. Object Chain

Any object may be followed by one or more other objects to form a **chain**. A chain represents a nested path, such as a namespace path; the final member of the chain is the primary target of the description or operation.

```ebnf
chain ::= object { object }
```

### 4.1 Uses of a Chain

A chain can represent three different semantic relationships between two adjacent members:

**Membership:** When the earlier member is a class, the later member belongs to it.

```
c(App)c(Router)f(add_get)
```

This means `add_get` belongs to `Router`, which itself belongs to `App`.

**Composition/Output:** When the earlier member is a function or variable, the later member is considered an output or related value.

```
f(login)v(user)
```

This means that after calling `login`, a value named `user` is available.

**Parameter Elaboration:** When the later member is a `p()` parameter reference (section 5.1), it refers to a parameter already declared on the earlier member, regardless of the earlier member's type.

```
c(Order)p(user)
```

This means `user` refers to a parameter of `Order` (i.e., of its constructor), not a member or an output of it.

> **Rule:** The relationship between two adjacent members is determined in the following order:
> 1. If the later member is a `p()`, the relationship is **Parameter Elaboration**. This check takes precedence over the earlier member's type.
> 2. Otherwise, if the earlier member is of type `c()`, the relationship is **Membership**.
> 3. Otherwise (the earlier member is `f()` or `v()`), the relationship is **Composition/Output**.
>
> This rule is applied locally to each adjacent pair in a chain, not once for the chain as a whole; a single chain may combine more than one relationship type across its members.

---

## 5. Description, Output, and Their Combination

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
| `chain!-text` | error or side effect (section 9) |

Examples:

```
f(foo)-a simple example function
f(foo)->str
f(foo)->str-a simple example function
c(Router)!-shuts down the visual server and exits
```

> **Detection rule (for Validators):** The `->` operator must be searched for before an independent `-`; in other words, a line should first be checked for the `->` pattern, then for a `-` that is not immediately preceded by `>`.

### 5.1 Referring to a Specific Parameter

Sometimes a chain needs to describe not the object itself, but **one specific parameter** belonging to it — for example, to give a fuller explanation or a distinct output type for a single argument, beyond what fits inside the compact `params` list defined in section 11.

For this, DocScript defines the **parameter reference**, `p(name)`, introduced as a fourth core object type in section 3:

```ebnf
param_ref ::= "p(" identifier ")"
```

A `p(name)` behaves like `f()`, `c()`, and `v()` in one respect: it can appear as a member of a chain, and it supports the same description, output, output-with-description, and error/side-effect operators defined earlier in this section. When `p()` follows another object in a chain, the relationship between them is always **Parameter Elaboration** (section 4.1), regardless of whether the preceding object is a class, function, or variable.

Examples:

```
c(Order)p(user)->User-details of the user who placed the order
c(Order)p(user)-the user object associated with this order
f(add_get)p(handler)!-raises TypeError if handler is not callable
```

> **Note:** `p()` is used to elaborate on a parameter that has already been declared elsewhere — typically inside the `params` list of the object it belongs to (section 11) — not to declare a new parameter by itself. It is a way to attach a longer description, an output type, or an error/side-effect condition to something that already exists as a parameter.

### 5.2 Constraint: `p()` Cannot Take Its Own Parameters

Unlike `f()` and `c()`, a `p()` reference can never be followed by its own `params` list (section 11). Since `p()` only refers to a parameter that has already been declared elsewhere, giving it a parameter list of its own has no meaning.

**Correct:**

```
c(Order)p(user)->User-details of the user who placed the order
```

**Incorrect:**

```
c(Order)p(user)(role:str)->User-details of the user who placed the order
```

This constraint applies wherever `p()` appears in a chain, including as the final member. `v()` is subject to the same constraint, for the same reason; see section 11.2.

---

## 6. Types

The type system in DocScript is open; any identifier can be used as a type.

```ebnf
type ::= identifier [ "(" type {"," type} ")" ]
```

Examples: `str`, `int`, `Router`, `list(str)`, `dict(str, int)`, `optional(User)`

---

## 7. Control Structures

```ebnf
if_stmt     ::= "if" condition "->" body
for_stmt    ::= "for" identifier "in" iterable "->" body
while_stmt  ::= "while" condition "->" body
use_stmt    ::= "use" chain
ret_stmt    ::= "ret" chain

condition   ::= use_stmt | expression
body        ::= text | statement
statement   ::= if_stmt | for_stmt | while_stmt | use_stmt | ret_stmt
              | description | output | output_desc | error_desc
```

The `body` of a control structure may be free-form text or another formal statement, including a nested control structure.

`ret` marks the value returned by the enclosing description — typically the last statement inside an `if`/`for`/`while` body, or the final line of a chain of nested conditions. Its operand is a `chain`, most often a single `v()` or the result of a `use` expression.

Examples:

```
if use c(Router)f(close) -> close all open routes
if use f(login) -> if use f(getUser) -> ret v(user)
```

---

## 8. Nesting

Control structures can be written in two forms:

### 8.1 Inline Form

Suitable for shallow nesting, usually one or two levels:

```
if use f(login) -> if use f(getUser) -> ret v(user)
```

### 8.2 Indent-Based Block Form

Suitable for deeper nesting or higher readability:

```
if use f(login) ->
    if use f(getUser) ->
        ret v(user)
```

```ebnf
body ::= text | statement | NEWLINE INDENT statement DEDENT
```

> **Style guide:** For bodies with more than one level of nesting, the indented form is recommended. This is a style rule, not a grammar restriction.

A comment-only line (section 2) may appear anywhere inside an indented block, on its own line and at any indentation level. It does not count as the `statement` the block requires, and it does not affect indentation tracking.

```
if use f(login) ->
    # verify credentials before issuing a token
    if use f(getUser) ->
        ret v(user)
```

---

## 9. Error & Side Effect

The `!-` symbol is used both to describe behavior under error conditions and to describe automatic side effects.

```ebnf
error_desc ::= chain "!-" text
```

Examples:

```
c(Router)!-shuts down the visual server and exits
f(connect)!-raises ConnectionError if the host is unreachable
```

### 9.1 Limitation: One `!-` per Statement

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

## 10. Async

```ebnf
async_marker ::= "async" chain
```

Example:

```
async f(fetch)-fetches a resource asynchronously
```

---

## 11. Parameters

Parameters of a function or class are written immediately after it, inside a separate pair of parentheses. `v()` and `p()` do not take a `params` list of their own (section 11.2).

```ebnf
params      ::= "(" [param {"," param}] ")"
param       ::= identifier ":" type
chain_head  ::= (function | class) [params] | variable | param_ref
chain       ::= chain_head { (function | class) [params] | variable | param_ref }
```

Example:

```
f(add_get)(path:str, handler:func)->bool-adds a new GET route
c(Router)(base_path:str)-creates a new router with a base path
```

### 11.1 Rule: Parameters End the Chain

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

### 11.2 Constraint: `v()` Cannot Take Parameters

Only `f()` and `c()` may be followed by a `params` list. `v()` refers to a variable, not something invoked with arguments, so giving it a parameter list has no meaning — the same reasoning that excludes `p()` (section 5.2).

**Correct:**

```
c(Cart)v(total)->float-the current total price of the cart
```

**Incorrect:**

```
c(Cart)v(total)(currency:str)->float-the current total price of the cart
```

This restricts `params` specifically; it does not restrict chaining. `v()` can still be followed by other objects in a chain under the Composition/Output relationship (section 4.1) — for example, `f(login)v(user)` remains valid, since no `params` list is involved.

---

## 12. Complete Grammar

```ebnf
document     ::= { line }
line         ::= [ statement ] [ comment ] NEWLINE

comment      ::= "#" { any_char_except_newline }

statement    ::= async_marker | if_stmt | for_stmt | while_stmt
               | use_stmt | ret_stmt
               | description | output | output_desc | error_desc

async_marker ::= "async" chain

if_stmt      ::= "if" condition "->" body
for_stmt     ::= "for" identifier "in" iterable "->" body
while_stmt   ::= "while" condition "->" body
use_stmt     ::= "use" chain
ret_stmt     ::= "ret" chain

condition    ::= use_stmt | expression
body         ::= text | statement | NEWLINE INDENT statement DEDENT

description  ::= chain "-" text
output       ::= chain "->" type
output_desc  ::= chain "->" type "-" text
error_desc   ::= chain "!-" text

chain        ::= chain_head { chain_member }
chain_head   ::= chain_member
chain_member ::= (function | class) [params] | variable | param_ref  ; only function/class may carry `params` (sections 5.2, 11.2)

object       ::= function | class | variable | param_ref
function     ::= "f(" identifier ")"
class        ::= "c(" identifier ")"
variable     ::= "v(" identifier ")"
param_ref    ::= "p(" identifier ")"

params       ::= "(" [param {"," param}] ")"
param        ::= identifier ":" type
type         ::= identifier [ "(" type {"," type} ")" ]

identifier   ::= letter { letter | digit | "_" }
```

---

## 13. Validator Implementation Guide (Optional)

DocScript is designed to be used without any tools. If you need to build a Validator or Linter, the recommended parsing order for a statement is:

1. Strip comments: remove everything from an unescaped `#` to the end of the line before any further processing (section 2).
2. Check for `async` at the beginning of the statement.
3. Check for control-structure keywords (`if`, `for`, `while`, `use`, `ret`) at the beginning of the statement.
4. Search for the object chain (`chain`) by separating `object` — including `p()` parameter references, which parse identically to `f()`, `c()`, and `v()` — and `params` according to section 11.
5. Reject any `p()` or `v()` that is immediately followed by a `params` list (sections 5.2, 11.2).
6. Search for the `->` operator before an independent `-`, to distinguish `output` from `description`.
7. Search for the `!-` operator and enforce the "one `!-` per statement" limitation (section 9.1).
8. Enforce the "parameters end the chain" rule (section 11.1) to reject invalid chains.