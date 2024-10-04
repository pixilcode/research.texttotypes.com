+++
title = "Fixed-Point Parser: Types and Mechanics"
description = "An dive into the types and mechanics of the fixed-point parser"
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


# State Transformer

Once it has the index and the continuation, it then **transforms the state that
it is given**. Thus, this state transformation could be represented as a
function that takes a state and returns a state.

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
   somewhat-complicated OCaml one *)
type state = (parser_location, location_info) Map
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

Earlier, we mentioned that the 'parser call' `P 1` returns some parse
information. What is this information? It depends on exactly what you're trying
to parse, but there are some things that we will need every time for the
fixed-point parser.

When we call a 


# Parser Continuation

TODO


# Building a simple parser

TODO


# Tagging and Memoizing a Parser

TODO

