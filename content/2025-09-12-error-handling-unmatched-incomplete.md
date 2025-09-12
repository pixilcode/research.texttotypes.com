+++
title = "Error Handling: Unmatched/Incomplete Errors"
description = "An exploration of how error handling can be improved"
date = 2025-09-12

[taxonomies]
tags = ["oneil", "parsing", "error-handling"]

[extra]
repo_view = true
+++

In [my work on the Rust parser](https://github.com/careweather/oneil/pull/10)
for the [Oneil programming language](github.com/careweather/oneil), I was having
some trouble with figuring out how to get the error messages that I wanted.

The parser uses [parser
combinators](https://en.wikipedia.org/wiki/Parser_combinator) and the
[`nom`](https://docs.rs/nom/latest/nom/) library to parse. In `nom`, there are
two (relevant) kinds of errors: [`Err::Error` and
`Err::Failure`](https://docs.rs/nom/latest/nom/enum.Err.html). I tried to figure
out how to use these in order to get the error messages that I wanted, and in
the process, I came up with the approach to error handling that this article
describes.

<!-- side note -->
> For those that are curious, the error kinds described in this article
> correspond to the error kinds found in `nom`. **Unmatched errors** correspond
> to **`Err::Error`** and **incomplete errors** correspond to
> **`Err::Failure`**.

# Introduction

In the approach that I developed, when parsing, there are two kinds of errors:
**unmatched errors** and **incomplete errors**. **Unmatched errors** indicate
that the *current* parser did not match, but that *other* parsers might match.
**Incomplete errors** indicate that the current parser *partially* matched but
was unable to *complete* its pattern.

Imagine we are trying to parse a decimal number (`[0-9]*\.[0-9]+`). The
following strings would succeed.

```py
0.1 # standard decimal
.123 # no initial digit is required
123.4 # can have multiple initial digits
```

The following strings would produce **unmatched errors** because they are
completely **unmatched**.

```py
'hello'
my_var
```

The following strings would produce **incomplete errors** because the strings
match partially, but they are **incomplete**.

```py
123 # missing a decimal after the digits
0. # missing digits after the decimal
. # missing digits after the decimal
```


# What's the difference?

<!-- question -->
> Both of these errors cause the parser to fail. What's the difference between
> these two errors?

The difference between these two errors is significant when parsing multiple
possible parsers.

Given a parser `P` that parses `A | B`, if `A` fails with an **unmatched
error**, then `P` should try `B`. If `B` fails with an **unmatched error**, then
`P` should fail with an **unmatched error** because neither of it's alternatives
matched.

However, if `A` fails with an **incomplete error**, then `P` should immediately
fail with an **incomplete error** as well, rather than trying `B`. This is
because `P` knows that `A` *should* match the string, but the match is
incomplete.

For example, consider the following grammar (written in
[EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form)) for
parsing a simple language that accepts alphabetic characters and integers.

```ebnf
Value = "'", ("a"-"z"), "'"
      | 0-9, { 0-9 }
```

Examples of strings in this language are `'a'` and `1`.

If we were to build a parser for this, the pseudocode might look something like
this:

```rust
fn value_parser(input) -> ParseResult {
  // "'", { a-z }, "'"
  let char_parser = sequence(
    token_parser("'"),
    alpha_parser,
    token_parser("'"),
  );

  // 0-9, { 0-9 }
  let integer_parser = sequence(
    number_parser,
    zero_or_more(number_parser),
  );

  // ... | ...
  alternate(char_parser, integer_parser).parse(input)
}
```

<!-- side note -->
> This psuedocode uses [parser
> combinators](https://en.wikipedia.org/wiki/Parser_combinator), since that is
> what Oneil uses for parsing.


Let's assume that a parser fails with an **unmatched error** by default. If we
use this parser to try to parse the string `'a`, it would go as follows.

<!-- important note -->
> `_` indicates the current position in the parse.

```txt
_'a
=> value_parser
  _'a
  => alternate(char_parser, integer_parser)
    _'a
    => char_parser
      _'a
      => sequence(token_parser("'"), alpha_parser, token_parser("'"))
        _'a
        => token_parser("'")
        <= token_parser("'") "success"
        '_a
        => alpha_parser
        <= alpha_parser "success"
        'a_
        => token_parser("'")
        <= token_parser("'") "unmatched"
      <= sequence(...) "unmatched"
    <= char_parser "unmatched"
    _'a
    => integer_parser
      _'a
      => sequence(number_parser, zero_or_more(number_parser))
        _'a
        => number_parser
        <= number_parser "unmatched"
      <= sequence(...) "unmatched"
    <= integer_parser "unmatched"
  <= alternate(...) "unmatched"
<= value_parser "unmatched"
```

The value parser returned an **unmatched** error, even though it matched
*partially*.

If we want it to instead return an **incomplete** error, we would need to adjust
our parser. We'll use `required(<parser>)` to convert an **unmatched error**
into an **incomplete error**.

For `char_parser`, we know that, once we've hit a quote (`'`), it should be
followed by an alphabet character, then another quote. If it isn't, we know that
`char_parser` is incomplete. So, we'll mark the `alpha_parser` and the second
`token_parser("'")` as `required`.

```rust
fn value_parser(input) -> ParseResult {
  // "'", { a-z }, "'"
  let char_parser = sequence(
    token_parser("'"),
    required(alpha_parser), // UPDATED
    required(token_parser("'")), // UPDATED
  );

  // 0-9, { 0-9 }
  let integer_parser = sequence(
    number_parser,
    zero_or_more(number_parser),
  );

  // ... | ...
  alternate(char_parser, integer_parser).parse(input)
}
```

If we try again to parse the string `'a`, it would go as follows:

```txt
_'a
=> value_parser
  _'a
  => alternate(char_parser, integer_parser)
    _'a
    => char_parser
      _'a
      => sequence(token_parser("'"), required(alpha_parser), required(token_parser("'")))
        _'a
        => token_parser("'")
        <= token_parser("'") "success"
        '_a
        => required(alpha_parser)
          '_a
          => alpha_parser
          <= alpha_parser "success"
        <= required(alpha_parser) "success"
        'a_
        => required(token_parser("'"))
          => token_parser("'")
          <= token_parser("'") "unmatched"
        <= required(token_parser("'")) "incomplete"
      <= sequence(...) "incomplete"
    <= char_parser "incomplete"
  <= alternate(...) "incomplete"
<= value_parser "incomplete"
```

This result says that the `value_parser` had an **incomplete error**, which
matches what we would like.


# Why?

<!-- question -->
> Whether you use the unmatched or incomplete version of `value_parser`, you get
> an error either way, so you know that the parse failed. Why does it matter
> whether the error is "unmatched" or "incomplete"?

The main reason for this distinction is effective error reporting, especially
when there are multiple paths that could be taken. When you mark an error as
"incomplete", you can also include context about where and why it was
incomplete. If we extended the `required` function to also take an error type,
we could improve the error reporting of the parser.


```rust
fn value_parser(input) -> ParseResult {
  // "'", { a-z }, "'"
  let char_parser = sequence(
    token_parser("'"),
    required(alpha_parser, CharMissingAlphaError), // UPDATED
    required(token_parser("'"), CharMissingClosingQuoteError), // UPDATED
  );

  // 0-9, { 0-9 }
  let integer_parser = sequence(
    number_parser,
    zero_or_more(number_parser),
  );

  // ... | ...
  alternate(char_parser, integer_parser).parse(input)
}
```

In addition, we could also modify the `required` function to return the offset
within the string at which the error occurs.

With this improvement, we'd get clearer error messages.

```rust
// assume 0-indexed offset
value_parser("'") // CharMissingAlphaError, offset 1
value_parser("'a") // CharMissingClosingQuoteError, offset 2
value_parser("invalid") // UnmatchedError, offset 0
```


# Flaws

Although this approach works well generally, and is the approach that Oneil
uses, it has some flaws.

## Shared prefixes

The first is that this approach cannot handle prefixes effectively when they are
seperated into different alternation branches. For example, let's say that we
are trying to parse variable that could include array indexing, and we allow for
either a single index or a range.

```ebnf
Variable = ident
         | ident, "[", number, "]" (* single index *)
         | ident, "[", number, ":", number, "]" (* range index *)
```

When parsing this, we can't effectively express all incomplete expressions. If
we were parsing an expression that began with "arr[1", we would first try the
*single index* alternative. If we checked the next character, and it wasn't "]",
we could return an incomplete error, indicating that we had a partial parse.

However, if we returned an incomplete error, then the parser would never
reach the *range index* alternative, since the `:` token expected for that
alternative would cause the *single index* alternative to return an
incomplete error, which would immediately cause the `Variable` parser to
fail with an incomplete error. Therefore, the *single index* alternative must
return an unmatched error.

In this case, it is trivial to rewrite the grammar/parser so that this is not a
problem. However, in a more complex grammar, this might be more difficult to do.

## Relies on the developer to decide where shared prefixes exist

The second flaw of the approach builds on the first. This approach requires the
developer to identify *where* to start treating errors as incomplete errors.
Because this approach doesn't handle conflicting prefixes, it isn't so simple as
"once we've parsed one part, everything else afterwards should be treated as
incomplete". Instead, the developer has to consider at what point the shared
prefixes end.

<!-- important note -->
> For simplicity, we'll call this the **incompletion point**. Errors after the
> incompletion point should be considered **incomplete errors** rather than
> **unmatched errors**. 
>
> In EBNF grammars, we will mark this with `<!>`.
>
> ```ebnf
> Rule = A, B, <!> C, D
> ```

If the developer gets the incompletion point wrong, they could cause problems
with the parsing.

For example, let's say a developer defines the following grammar (with
incompletion points marked with `<!>`).

```ebnf
Expr = Number
     | FunctionCall

Number = digit, { digit }
FunctionCall = ident, <!> "(", Number, ")"
```

In this grammar, everything after `ident` in `FunctionCall` could be considered
incomplete. So `my_func`, `my_func(`, and `my_func(1` would all be considered
incomplete.

Now, let's say we add rule for an assignment expression to the grammar.

```ebnf
Expr = Number
     | FunctionCall
     | Assignment

Number = digit, { digit }
FunctionCall = ident, <!> "(", Number, ")"
Assignment = ident, "=", Number
```

This suddenly changes the incompletion point of the `FunctionCall` rule, since
both `FunctionCall` and `Assignment` begin with `ident`. If the developer
notices this, then they can adjust the completion point to after the `(` token.

```ebnf
FunctionCall = ident, "(", <!> Number, ")"
```

If they don't notice it, however, then the parser will be unable to parse the
`Assignment` rule, since an assignment would cause an incompletion error in
`FunctionCall` first.


# Further Exploration and Conclusion

There are other ways of handling this that merit further exploration.

As I've written this article, I've come to realize that the main problem that
unmatched/incomplete errors address could be stated is this:

<!-- key point -->
> When we try multiple branches, we could get back 1 or more errors. If there
> errors, how do we decide what error to return?

There are many different strategies that could be used:

- the strategy that [`nom`'s `alt`
  combinator](https://docs.rs/nom/latest/nom/branch/fn.alt.html) uses is to
  return the last error that occurs.

- the strategy that I propose in this article is to create a special error type,
  which I called "incomplete" errors, and to return the first special error that
  occurs.

- another strategy would be to return the error that advanced the farthest

Aside from the strategies above, which each pick one single error from the
errors that occur, there could be other strategies that combine all the errors
into one error.

It would be interesting to explore different strategies to find if there are any
that would be optimal for a given language, or even optimal for every language.

For now, though, thanks for joining me on this trip. I learned a lot even as I
was writing it. Hopefully you did as well!
