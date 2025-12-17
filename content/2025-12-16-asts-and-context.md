+++
title = "ASTs and Context"
description = "The connection between AST nodes and context, derived from Rhombus macros"
date = 2025-11-13

[taxonomies]
tags = ["parsing"]

[extra]
repo_view = true
+++

I was reading the paper
[*Rhombus: A New Spin on Macros without All the Parentheses*](https://jeapostrophe.github.io/home/static/rhombus-2023.pdf)
last night, and I made a mind-blowing connection.

In Rhombus, macros operate in a defined *space*. Here's what the paper
says about spaces:

> A Rhombus *space* represents a particular kind of program context,
> such as an expression context, binding context, or annotation
> context. A Rhombus module starts out in an expression context, and
> then some expression forms create other kinds of contexts, such
> as the binding context created for the left-hand side of `def` or
> the annotation context created on the right-hand side of `::`.
> Each space has its own sublanguage of forms that are implemented
> as macros specific to that space.

As I was reading this, I realized that this is how parsers and ASTs
work on an abstract level. The parser starts out in one context,
such as a "module" context. The "module" context may have different
"definition macros" that it attempts to parse, and each
of those "macros" will have sublanguages with their own contexts.

As the parser goes along, it produces a record of the contexts
that it has encountered. This record is known as the *AST*.

A standard AST implementation for me might look something like this
(written in pseudo-Rust):

```rust
struct Program(Vec<Defn>);

enum Defn {
  Constant {
    ident: Ident,
    typ: Type,
    value: Expr,
  },
  Function {
    ident: Ident,
    args: Vec<(Ident, Type)>,
    return_typ: Type,
    body: Expr,
  }
}

enum Expr {
  // possible expressions
}

struct Type(/* type info */)

struct Ident(/* ident info */)
```

The mind-blowing connection for me was between contexts and
AST nodes. Each `struct` or `enum` defines a different
*parsing context*. So this language would have a "program"
context, a "definition" context, an "expression" context,
a "type" context, and an "identifier" context.

This will help me to better define parsers and ASTs in the
future. Once, when designing a language similar to this
(it's unimplemented as
of yet), I was wondering whether types should just be stored
as an `Ident` or if they should get their own unique node
type, since their parsing is similar to identifiers.

Based on this new perspective, types should get their own
node type since their *context* is different. This would
allow me to develop types more effectively than trying to
make them work with just an identifier.

Anyway, this may seem obvious to some, but it was a pretty
cool revelation to me!

