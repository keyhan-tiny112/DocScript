# DocScript

**A lightweight standard for documenting code — in the shape of code.**

DocScript is not a language, it is not an executable tool, and it is not compiled. It is simply a **writing convention** for describing classes, functions, variables, and their behavior — with syntax that a programmer's eye can understand in seconds.

```
c(AuthService)(secret_key:str, token_ttl:int)-create a new auth service with a signing secret and token lifetime
c(AuthService)f(login)(username:str, password:str)->Token-authenticate a user and return an access token
c(AuthService)f(login)!-raises InvalidCredentials if username or password is wrong
```

## Table of contents

- [Why DocScript?](#why-docscript)
- [Installation](#installation)
- [Quick start](#quick-start)
  - [1) Describe an object](#1-describe-an-object)
  - [2) Show an output](#2-show-an-output)
  - [3) Combine description and output](#3-combine-description-and-output)
  - [4) A method of a class (object chain)](#4-a-method-of-a-class-object-chain)
  - [5) Add parameters to an object](#5-add-parameters-to-an-object)
  - [6) Error behavior or side effects](#6-error-behavior-or-side-effects)
  - [7) Conditional invocation (`use`)](#7-conditional-invocation-use)
  - [8) Loops](#8-loops)
  - [9) Complex nesting — indent-based form](#9-complex-nesting--indent-based-form)
  - [10) Async functions](#10-async-functions)
- [Full Documentation](#full-documentation)
- [Design Philosophy](#design-philosophy)
- [Contributing](#contributing)
- [License](#license)
- [Examples](#examples)

---

## Why DocScript?

Conventional documentation formats, such as Google-style docstrings or JSDoc, are useful, but they share one problem: **they are free-form text.** Reading them takes about as long as reading an ordinary paragraph, even when the subject is fully technical and structured.

DocScript starts from a simple assumption:

> Programmers are more comfortable with code structure than with free-form prose. If documentation looks like code, it is absorbed faster.

The result:

- ⚡ **Faster to read** — the eye is already trained on patterns like `if/for/function`, so it scans instead of reading line by line.
- 🎯 **Less ambiguity** — free-form prose can be vague in places ("usually", "in most cases"); DocScript's structure pushes you to state conditions, outputs, and errors explicitly.
- 🌐 **Programming-language independent** — DocScript syntax is not tied to Python, JavaScript, or any other language; it describes the concept.
- 🪶 **Lightweight and tool-free** — it requires no compiler, parser, or dependency. A `.md` file, or even a comment above the code, is enough.

DocScript is not a replacement for conventional documentation; it is a **quick, scannable summary layer** over system behavior — for moments when someone only wants to quickly understand "what this class/function does, what it takes, what it returns, and where it fails."

---

## Installation

There is nothing to install. DocScript is a syntax, not a tool. All you need to do is:

1. Read [`SPEC.md`](./SPEC.md) to become familiar with the rules.
2. Write your documentation using the same syntax — either in a separate `.md` file or inside comments above a class/function.

---

## Quick Start

### 1. Describe an object

Every object is either `f()` (function), `c()` (class), or `v()` (variable):

```
f(foo)-a simple example function
```

### 2. Show an output

```
f(foo)->str
```

### 3. Combine description and output

```
f(foo)->str-a simple example function
```

### 4. A method of a class (object chain)

```
c(Router)f(add_get)-add a route with the GET method
```

You can go several levels deep as well, like a nested namespace:

```
c(App)c(Router)f(add_get)-add a route with the GET method
```

### 5. Add parameters to an object

```
f(add_get)(path:str, handler:func)->bool-add a new GET route
```

⚠️ Note: when you put parameters on an object, that object becomes the "final target" of the chain, and you can no longer chain another method after it. For example:

```
c(Router)(base_path:str)-create a new router with a base path      ✅ correct
c(Router)(base_path:str)f(create_router)-...                       ❌ wrong
```

### 6. Error behavior or side effects

```
c(AuthService)f(refresh)!-raises TokenExpired if refresh_token has expired
```

If you have multiple scenarios, write each one on its own line, instead of using multiple `!-` markers on a single line:

```
c(AuthService)f(refresh)!-raises TokenExpired if refresh_token has expired
c(AuthService)f(refresh)!-shutdown current session and require re-login
```

### 7. Conditional invocation (`use`)

```
if use c(Router)f(close) -> close routes
```

### 8. Loops

```
for r in v(routes) -> if use c(Router)v(is_active) -> c(Router)f(close)
while use f(server_running) -> f(cleanup_expired_tokens)-run cleanup every cycle
```

### 9. Complex nesting — indent-based form

When nesting becomes deep, use indentation instead of one long line:

```
if use f(login) ->
    if use f(getUser) ->
        return v(user)
```

### 10. Async functions

```
async f(fetch)(url:str)->Response-fetch a resource asynchronously
```

---

## Full Documentation

For the complete formal syntax, written in EBNF style, along with all grammar rules and the reasoning behind each decision, see [`SPEC.md`](./SPEC.md).

To see a complete real-world example, an authentication API with login/logout/refresh/session, visit [`Examples section`](#examples).

---

## Design Philosophy

- **No mandatory parser**: DocScript is designed for humans to read, not machines to execute. If you ever want to build a Validator or Linter for it, `SPEC.md` contains the required rules — but DocScript itself is fully usable without any tools.
- **Open, not closed**: its type system is completely open; any name can be a type.
- **Extensible**: the rules are designed to work for both a simple function and a complete multi-layer architecture.

---

## Contributing

DocScript is still in its early stages. If you find a suggestion, ambiguity, or edge case that the spec does not cover, open an Issue or send a PR.

## License

[MIT](./LICENSE)

---

## Examples

- [Auth API](Docs/Examples/auth-api.md)
- [Shoping Cart API](Docs/Examples/shoping-cart-api.md)
