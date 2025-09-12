+++
title = "Fixed-Point Parsing: Essentials"
description = "The essential components of a fixed-point parsing library"
date = 2024-03-12

[taxonomies]
tags = ["ocaml", "fixed-point parsing"]

[extra]
repo_view = true
+++

Okay, so by now, I've gotten an implementation of the fixed-point parser that we
are researching that I feel good about. The implementation, [found
here](https://github.com/pixilcode/fpp_personal_parsers/tree/main/fpp_framework),
includes a `Parser_base` module that provides the very basics of the parser
framework, a `Combinators` module that adds on a lot of useful combinators, and
a `Strings` module that helps when dealing with strings.

In this post, I want to go over the essentials of the fixed-point parser
framework. This includes one parser combinator (`choice`) and two other
essential functions: `memo` for memoizing parsers (this solves left recursion
problems); `no_results` for indicating that a parser returns no results; and
`run` for running a parser. Together, these ensure that the user of the library
don't need to know anything about the `state`.

I will also explain two other combinators that make this base more
user-friendly, `sequence` and `unit`, and one function that allows for better
debugging, `run_raw`.

*NOTE: though the types have changed slightly since I wrote about them, [my post
about fixed-point parser types] will give you a good idea about the types that
are being used here. If you want to understand the exact implementation,
reference the code itself for now*

# The `choice` combinator

```ocaml
let choice (parser1 : 'a parser) (parser2 : 'a parser) : 'a parser =
  fun (idx, callback) ->
    fun state ->
      state
      |> parser1 (idx, callback)
      |> parser2 (idx, callback)
```

The choice combinator takes two parsers, `parser1` and `parser2`, and it returns
a parser that tries both parsers. Note that this is *unordered* choice. In other
words, the parser will try `parser2` *even if* `parser1` succeeds.

This combinator works by simply returning a `state_transformer` that runs the
initial `state` through the `state_transformer` produced by running `parser1`,
then running the resulting `state` through the `state_transformer` produced by
running `parser2`.


# The `memo` function

```ocaml
let memo ~(tag : tag) (parser : cached_value parser) : cached_value parser =
   fun (idx, callback) ->
    let loc : parser_position = (tag, idx) in
    fun state ->
      let (State state) = state in
      match Hashtbl.find state loc with
      | Some (parser_outputs, callbacks) ->
          let new_callbacks = callback :: callbacks in
          Hashtbl.set state ~key:loc ~data:(parser_outputs, new_callbacks) ;
          Set.fold parser_outputs ~init:(State state)
            ~f:(fun state parser_output ->
              let state_transformer = callback parser_output in
              state_transformer state )
      | None ->
          let new_callback : cached_value parser_callback =
           fun parser_output ->
            fun state ->
             let (State state) = state in
             match Hashtbl.find state loc with
             | Some (parser_outputs, callbacks) ->
                 if Set.mem parser_outputs parser_output then State state
                 else
                   let new_parser_outputs =
                     Set.add parser_outputs parser_output
                   in
                   Hashtbl.set state ~key:loc
                     ~data:(new_parser_outputs, callbacks) ;
                   List.fold_left callbacks ~init:(State state)
                     ~f:(fun state callback ->
                       let state_transformer = callback parser_output in
                       state_transformer state )
             | None ->
                 failwith "unreachable"
          in
          Hashtbl.set state ~key:loc
            ~data:(CachedParserOutputSet.empty, [callback]) ;
          parser (idx, new_callback) (State state)
```
