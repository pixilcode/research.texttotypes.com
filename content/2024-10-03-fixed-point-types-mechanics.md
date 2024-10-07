+++
title = "Fixed-Point Parser: Types and Mechanics"
description = "A dive into the types and mechanics of the fixed-point parser"
date = 2024-10-03

[taxonomies]
tags = ["fixed-point parsing"]

[extra]
repo_view = true
+++

A fixed-point parser can seem like a complicated idea. It's got lots of types,
and lots of functions being passed around. Plus, it's got some monads involved,
which can be quite intimidating!

However, it isn't as complex as it may seems once you understand some of the
underlying concepts. So, let's dive into what makes up a fixed-point parser!

# The Parser

In the fixed-point parser, a parser is represented as a function that takes
**the current index** in the parse string and **a continuation to call once it
is done**.  'done' could mean that it succeeded, it could mean that it failed,
or it could mean something else. In addition, the continuation doesn't ever have
to be called.

```ocaml
type parser = index -> parser_continuation -> state_transformer
```


# Parser Continuation

Notice that the parser takes a `parser_continuation` in addition to an `index`.
This parser continuation could be considered to be a 'done' function. We call
the parser continuation once the parser has completed.

A continuation is called from within the parser. It may be called once, such as
in the case of the unit parser `unit x`, where the parser doesn't parse anything
and immediately calls the continuation with the value `x` and the current index.
It also may be called multiple times, such as in the case of the alternation
parser `A|B`, where it may complete with both `A` and `B`. It may also never be
called, such as in the case of the `fail` parser, where it immediately fails,
and thus never completes.


## Parser "Output"

When you run a parser, it usually returns some value representing what was
parsed. It also advances the parent parser (if there is one) along the string.
We represent this with an `output_pair`. An `output_pair` is simply a
value/index pair. The value is the result of the parser, and the index is the
index on which the parser ended. For example, given the string `"12ab"` and a 
parser that parses numbers, the parser will return the number `12` and the index
`2` (assuming zero-based indexing).

```ocaml
type output_pair = index * value
```


### Value and Index

As a side note, we need to quickly discuss what `index` and `value` are. They can
change depending how you decide to represent the two concepts, so we define these
only so that our code later makes sense.

`index` provides information about where you are in the string. For our
purposes, we will use an `string * int` pair. The `string` is the string that we
are attempting to parse.  This doesn't change throughout the whole parsing. The
`int` is the index of the current character in the string, with the index of the first
character being `0`.

```ocaml
(* (parse string * current character index) *)
type index = string * int
```

`value` is the value that you want your parser to produce. In our case, we will
simply return the parsed string. Thus, we will use `string`s for our `value`s.

```ocaml
type value = string
```


## Parser Continuation Result

Whatever happens after a parser may possibly update the state of the program.
This doesn't make a ton of sense yet, since we don't know what is inside the
state. We'll get to that next. The continuation will thus return a
`state_transformer`.

```ocaml
type parser_continuation = output_pair -> state_transformer
```


# State Transformer

Once it has the index and the continuation, the parser then **transforms the
state that it is given**. Thus, this state transformation could be represented
as a function that takes a state and returns a state.

```ocaml
type state_transformer = state -> state
```

*Note that in imperative languages, the state could be stored as a sort of
'global state' that is updated by the parser. In a language that supports it,
the state could also be managed with algebraic effects.*


# State

What is the "state" of the program that we keep referring to? It's a map from a
given "parser location" to certain information associated with that location (we
will get into what this information is in a second).

```ocaml
(* imagine some easy-to-use hashmap instead of the
   relatively complicated OCaml one *)
type state = (parser_location, location_info) map
```


## Parser Location

The type of the `parser_location` is a pair, a `rule_tag` and an `index`. The
easiest way to explain this is by considering a parser as a sort of 'function'.
It takes as input the index to start the parse, and returns information about
what it parsed.

From this standpoint, we can view the `state` as a memoization of a parser call.
Given a parser tagged with `P`, we can treat mapping of `P 1 -> foo` to mean
that "when `P` is called at the index of `1`, it returns `foo`.

```ocaml
let my_state =
  (* ... run (P 1) and update the state with the result [foo] ... *)
in
Map.get_value_of ("P", 1) ~from:my_state (* => returns [foo] *)
```

Note that this requires a parser to be able to be tagged and memoized, which we
will get to later.

```ocaml
type parser_location = rule_tag * index
```


## Location Info

We mentioned that the parser location info is mapped to some information related
to that location. There are two pieces of information that we need to remember.

First, we need to remember all of the values that the parser call has produced
at this position. We want to do this so that if we arrive at this point with a
new way to continue from this point (or in other words, a new
`parser_continuation`), we will want to run that with each value that has been
produced by this parser at this location. We will use a `set` of `output_pair`s
for this.

Next, we also need to remember all of the different possible ways that we can
continue from this point. This is because if we discover that this parser has
produced a new value, then we want to rerun all of the different possible ways
to continue using that value so that we can see if any off those produce a valid
parse. Thus, we will keep a `list` of `parser_continuation`s.

```ocaml
(* imagine some easy-to-use set, instead of the
   relatively complicated OCaml one *)
type location_info = (output_pair set) * (parser_continuation list)
```


# Building a simple parser

Now that we have the basic concepts down, let's build a parser of our own! We will
build a parser that reads a single character from the string and advances to the
next character, called `any_char`.

First, let's remember what a parser is. A parser takes an `index`, the current
location in the parse string, and a `parser_continuation`, the 'done' function
for the parser. We will call these `idx` and `complete`, respectively. 

```ocaml
let any_char : parser =
  fun (idx : index) (complete : parser_continuation) ->
```

As mentioned previously, the index is a `string * int` pair, so we will unwrap that
so we can get the current character.

```ocaml
    let parse_str, curr_char_idx = idx in
```

We want to make sure that we handle the case if the index is past the end of the
parse string, so we will use an if statement to handle this.

```ocaml
    if curr_char_idx < (String.length parse_str) then
```

If the index is inside the parse string, we will get the character at the index.
We will then convert the character to a string (since we are returning the
string that was parsed). Finally, we will advance the index by 1. Once we have
done that, we can 'finish' our parser by calling `complete` with the new index
and our character string as an `output_pair`. This will generate a
`state_transformer` that will be returned by our parser.

```ocaml
      let c = String.get parse_str curr_char_idx in
      let c_str = Char.to_string c in
      let new_index = curr_char_idx + 1 in
      complete (new_index, c_str) (* generates a state_transformer *)
```

Now, if the index is outside of the parse string, we want the parser to fail. How
do we do this? We just don't succeed. In other words, if calling the `complete`
continuation represents a parser succeeding, then we never call `complete`. However,
we still need to return a `state_transformer`. So we will simply create a `state_transformer`
that leaves the state unchanged.

```ocaml
    else
      (fun state -> state) (* leave the state unchanged *)
```

And there you have it! A parser that parses any character and advances the index. The
full code for this parser is below.

```ocaml
let any_char : parser =
  fun (idx : index) (complete : parser_continuation) ->
    let parse_str, curr_char_idx = idx in
    if curr_char_idx < (String.length parse_str) then
      let c = String.get parse_str curr_char_idx in
      let c_str = Char.to_string c in
      let new_index = curr_char_idx + 1 in
      complete (new_index, c_str) (* generates a state_transformer *)
    else
      (fun state -> state) (* leave the state unchanged *)
```


# Tagging and Memoizing a Parser

TODO

