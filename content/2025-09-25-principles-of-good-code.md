+++
title = "Principles of Good Code (According to Brendon)"
description = "Brendon's thoughts on ways to write good code"
date = 2025-09-25

[taxonomies]
tags = ["rust", "principles"]

[extra]
repo_view = true
+++

Recently, I've been writing documentation for
[Oneil](https://github.com/careweather/oneil/tree/bb-model-updates) in order to
make it easier for others to contribute to the language. As I've been doing so,
I've been thinking a lot about what principles I follow as I write code.

Here are some of the principles that I've found help me to write code that feels
clean, elegant, and maintable to me. A lot of it is targeted towards Rust, but
most of them are also applicable in other situations.

> **DISCLAIMER:** I realize that writing "good code" is an art more than a
> science, and even the definition of "good code" is subjective. I realize that
> others may disagree with my analysis. Honestly, even *future me* may find
> fault in these principles.
>
> In addition, I don't claim that these are the *only* important principles.
> There are plenty of other useful principles and concepts, such as the New Type
> and Builder patterns, as well as the principle of Dependency Inversion. If I
> could write about all of them, I would! Alas, I have limited motivation, and
> it ran out at the end of this article. Maybe I'll write a future post about
> them
>
> Finally, I don't claim to be the one who discovered all of these principles.
> This is an accumulation of things that I've learned from coding, reading, and
> talking with others over the years.

# Error Handling

When it comes to errors, I often think in terms of the [Midori Error
Model](https://joeduffyblog.com/2016/02/07/the-error-model/), where there are
*recoverable* and *unrecoverable* errors.

## Recoverable Errors

*Recoverable errors* are errors that can be anticipated and *should be handled*.
This includes things like being unable to parse a file or find a model. Note
that one way of handling recoverable errors is to display the error to the user.

In general, use `Result<T, E>` to represent recoverable errors.

## Unrecoverable Errors

*Unrecoverable errors* indicate bugs in the code. This is generally caused by
an invalid state within the program. As often as possible, the type system
should make invalid states representable. However, this isn't always feasible.

As an example of this, model loading stores its results in a `HashMap`. Later,
when submodel resolution occurs, we assume that the submodel either exists in
the `HashMap` or has a corresponding error. If neither of those is true, the
program is in an invalid state.

In situations like these, I would rather fail loudly as soon as I know about
the invalid state rather than fail silently and learn about the invalid state
much later in the program.

To handle unrecoverable errors:
- use `Result::expect` and `Option::expect` to unwrap `Result`s/`Option`s that
  should succeed
- use `assert!(<condition>, <message>)` if you want to ensure that a condition
  (such as a function invariant) holds
  - If an assertion is expensive, use `debug_assert!(...)` instead. In all other
    cases, prefer `assert!(...)`.
  - If you have an `assert!(...)` followed by a `<value>.expect`, consider if
    you can remove the assertion and just use the call to `expect` to enforce
    the same invariant
- use `unreachable!(<message>)` to indicate a path that should never be taken
- use `panic!(<message>)` if none of the other use cases apply
  - This should rarely be used. The other options give a more clear reason for
    failure, and they generally cover most unrecoverable bugs.

Make sure that you *include an informative error message* no matter which option
you decide to use. Note that the messages for the macros can be formatted in the
same way as `println!`.


# Mark TODOs And Unimplemented Features

When implementing a feature, I may think of future improvements that could be
made to the code. In addition, I often don't have time to handle every path or
edge case. 

If the code works as is but could be improved in the future, I use a `// TODO`
comment. This makes these tasks easier to find.

When I am developing, I mark unhandled paths and edge cases with the
`todo!(<message>)` macro. This ensures that those paths fail immediately when I
encounter them.

I use the `todo!` macro for temporary TODO items. On the other hand, if I don't
intend to resolve the `todo!`s in the pull request, I change the `todo!` macro
to `unimplemented!`.

> Note that the `todo!` and `unimplemented!` macros returns a type that coerces
> to any other type. So the following code is valid.
> 
> ```rust
> let my_number =
>   if some_condition {
>     42
>   } else {
>     todo!("handle when `some_condition` is not true")
>   };
> ```


# Use Tools

There are lots of tools to improve the developer experience and code quality. I
use the following tools.

## `cargo fmt`

Using `cargo fmt` allows me to keep the code style consistent. If I am running
VS Code, I like to set `"editor.formatOnSave"` to `true` in my settings in order
to have `cargo fmt` run whenever I save a file.

## `cargo test`

I run `cargo test` frequently to test any changes made to the code. When I am
using a Cargo workspace, if some crates have compile errors, I use `cargo test
-p <crate>` to test a specific crate.

For more information on testing principles, see [the section on
testing](#testing).

## `cargo clippy`

I like to use `cargo clippy` to lint your code. This helps to catch potential
errors and to use a consistent style of coding. Lints are defined in
`Cargo.toml` in [the `[lints.*]` or `[workspace.lints.*]`
sections](https://doc.rust-lang.org/stable/clippy/configuration.html#lints-section-in-cargotoml).

If I am using `rust-analyzer` in VS Code, I like to ensure that I am using the
`clippy` linter by [updating my
settings](https://users.rust-lang.org/t/how-to-use-clippy-in-vs-code-with-rust-analyzer/41881)

The lints can be strict sometimes, and there are some cases where the lints are
not useful. In this case, I may insert `#[expect(clippy::<lint>, reason =
"<message>")]` with an included reason for *why* it should be disabled. Try to
keep the scope of the exception as limited as possible.

> I use `#[expect(...)]` rather than `#[allow(...)]` since [`#[expect]` will
> let me know when the attribute is no longer
> needed.](https://rust-lang.github.io/rfcs/2383-lint-reasons.html#expect-lint-attribute)

> The lint configuration that I developed is mostly untested and is likely too
> strict. However, if you're curious to know what it is, I've included below. It
> includes a lot of rules that I generally agree with, though I have to use
> `#[expect]` every once in a while.

<details>
  <summary>Lint configuration</summary>

  <pre><code>
# Clippy lint groups
pedantic = { level = "warn", priority = -2 }
correctness = { level = "warn", priority = -3 }
style = { level = "warn", priority = -4 }
complexity = { level = "warn", priority = -5 }
perf = { level = "warn", priority = -6 }

# Clippy cargo-specific groups
cargo = { level = "warn", priority = -7 }
cargo_common_metadata = { level = "warn", priority = -8 }

# individual Clippy lints (nursery)
branches_sharing_code = "warn"
collection_is_never_read = "warn"
derive_partial_eq_without_eq = "warn"
equatable_if_let = "warn"
iter_on_empty_collections = "warn"
iter_on_single_items = "warn"
iter_with_drain = "warn"
large_stack_frames = "warn"
literal_string_with_formatting_args = "warn"
missing_const_for_fn = "warn"
needless_collect = "warn"
needless_pass_by_ref_mut = "warn"
option_if_let_else = "warn"
path_buf_push_overwrite = "warn"
redundant_clone = "warn"
redundant_pub_crate = "warn"
single_option_map = "warn"
suboptimal_flops = "warn"
too_long_first_doc_paragraph = "warn"
trait_duplication_in_bounds = "warn"
type_repetition_in_bounds = "warn"
unused_peekable = "warn"
unused_rounding = "warn"
use_self = "warn"
useless_let_if_seq = "warn"

# individual Clippy lints (restriction)
allow_attributes = "warn"
assertions_on_result_states = "warn"
clone_on_ref_ptr = "warn"
dbg_macro = "warn"
decimal_literal_representation = "warn"
doc_include_without_cfg = "warn"
empty_drop = "warn"
exit = "warn"
float_cmp_const = "warn"
if_then_some_else_none = "warn"
lossy_float_literal = "warn"
map_with_unused_argument_over_ranges = "warn"
multiple_inherent_impl = "warn"
panic_in_result_fn = "warn"
partial_pub_fields = "warn"
pathbuf_init_then_push = "warn"
print_stderr = "warn"
print_stdout = "warn"
redundant_test_prefix = "warn"
renamed_function_params = "warn"
return_and_then = "warn"
self_named_module_files = "warn"
string_add = "warn"
string_lit_chars_any = "warn"
tests_outside_test_module = "warn"
todo = "warn"
try_err = "warn"
unnecessary_self_imports = "warn"
unseparated_literal_suffix = "warn"
unused_result_ok = "warn"
unwrap_used = "warn"
use_debug = "warn"
wildcard_enum_match_arm = "warn"
  </code></pre>
</details>


# Prefer Flat Code

Nested code is harder to reason about, since I have to remember what information
each level of nesting introduces. Whenever possible, I keep the nesting down to
a minimum. 

One way to do this is to use `let ... else` and `if ... let` combined with early
returns to handle `Options`.

```rust
fn foo(hash_map: HashMap<u32, u32>) -> u32 {
  let maybe_value = hash_map.get(0);
  let Some(value) = maybe_value else {
    return 0;
  };

  // ... do other things with `value` ...
}
```

```rust
fn bar(key: u32) -> u32 {
  let maybe_duplicate = find_duplicate_of(key)
  if let Some(duplicate) = maybe_duplicate {
    return 0;
  }

  // ... assume key is not a duplicate ...
}
```


# Avoid Writing Declarative Macros

I do not write macros using `macro_rules!` if I can avoid it. Declarative macros
can reduce boilerplate, but they come at the price of a syntax that's harder to
read and code that's harder to debug. It's also easy to introduce a
"sublanguage" that developers now have to learn.

## `assert_...!` Macros

The only exception to this rule is `assert_...!` macros in tests. Because tests
often have assertion steps in common, these can be combined in a macro. This
ensures that no necessary assertions are forgotten, and it can make it a lot
more clear what a grouping of assertions does.

The reason for this exception is that if I were to combine the assertions into
an `assert_...` function, then anytime an assertion in that function failed, it
would point to code inside the function, rather than where the function was
called. Using a macro means that the location of the assertion in the test is
displayed, rather than the code inside the macro.

Note, however, that these macros are written in a specific style.

1. Assert macros should be written in a way that makes them look equivalent to a
   function call. Arguments are passed in as comma seperated expressions, with
   an optional trailing comma at the end.

2. There is only one macro branch, so it's always clear which code the macro is
   using.

3. Macro "arguments" are immediately assigned to variables with explicit types
   in order to ensure that each argument has the expected type.

4. Assertions inside the macro have an explicit message in order to make it
   quickly clear which assertion failed.

```rust
macro_rules! assert_var_is_builtin {
  ($variable:expr, $expected_ident:expr $(,)?) => {
      let variable: ir::ExprWithSpan = $variable;
      let expected_ident: &str = $expected_ident;

      let ir::Expr::Variable(ir::Variable::Builtin(actual_ident)) = variable.value() else {
          panic!("expected builtin variable, got {variable:?}");
      };

      assert_eq!(
          actual_ident.as_str(),
          expected_ident,
          "actual ident does not match expected ident"
      );
  };
}
```

# Testing

## 3-Step Unit Tests

Unit tests should test a single function. A single unit test should test that
**one given input** produces **one expected output**. When writing a unit test,
it should follow three steps: **prepare**, **run**, then **assert**.

First, the test should **prepare** any inputs needed to run the function. This
could include constructing the string that needs to be parsed, building the set
of models that have previously been resolved, or making the IR that needs to be
evaluated.

Next, the test should **run** the function. Pass in the inputs and store the
result.

Finally, the test should **assert** things about the result. Use liberally
`assert!`, `assert_eq!`, `panic!`, `Option::expect`, `Result::expect`,
`Result::expect_err`, and anything else that panics.

Also, when I expect a certain variant of an enum, use `let ... else` to unwrap
it. I generally avoid using `match` since there's usually only one expected
path, and `let ... else` keeps the failure close to the unwrapping.

```rust
// PREFER THIS
let MyEnum::Variant1 { field1, field2 } = value else {
  panic!("Expected Variant1, got {value:?}");
};

assert_eq!(field1, expected_field1);
assert_eq!(field2, expected_field2);

// OVER THIS
match value {
  MyEnum::Variant1 { field1, field2 } => {
    assert_eq!(field1, expected_field1);
    assert_eq!(field2, expected_field2);
  }
  _ => panic!("Expected Variant1, got {value:?}");
}
```

## Test Coverage

**Testing doesn't have to have 100% coverage.** 100% coverage may be feasible
when a program is small and simple. However, the more complex a problem gets,
the harder it gets to cover every possible branch.

Instead of trying to cover every edge case, I write a unit test or two that test
the main functionality. This ensures that the expected use case works.

When I encounter a bug, I write a test that proves that the bug exists. I fix
the bug, then rerun the test to prove that it no longer exists.
