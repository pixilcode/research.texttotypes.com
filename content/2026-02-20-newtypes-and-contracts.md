+++
title = "Newtypes and Contracts"
description = "Using newtypes to enforce contracts in the type system"
date = 2026-02-20

[taxonomies]
tags = ["type-systems"]

[extra]
repo_view = true
+++

Newtypes are a pattern that I've used in Rust in order to use the type system
to **enforce rules** (such as "this number is greater than zero") and to
**differentiate between otherwise similar types** (for example, a module name
and a variable name might both be represented as strings). I've recently been
thinking about the first use case and how it relates to contracts and gradual
typing.


## Newtype Pattern

As a quick overview, the [newtype
pattern](https://rust-unofficial.github.io/patterns/patterns/behavioural/newtype.html)
involves storing a type in a wrapper type. Here are some examples of this in
Rust:

```rust
struct NonZeroU32(u32);

struct ModuleName(String);

struct VariableName(String);
```

As I mentioned before, this can be used to enforce rules at the type system level.
For example, one I can define `NonZeroU32` in the following way:

```rust
struct NonZeroU32(u32);

impl NonZeroU32 {
  fn new(n: u32) -> Self {
    assert!(n > 0, "n must not be zero");
    Self(n)
  }

  fn as_u32(&self) -> u32 {
    self.0
  }
}
```

> **NOTE:** Yes, `NonZeroU32` [does already
> exist](https://doc.rust-lang.org/std/num/type.NonZeroU32.html), I'm recreating
> it here for demonstration purposes.

Because `NonZeroU32` can only be constructed using `NonZeroU32::new`, I know
that anytime I have a `NonZeroU32`, the underlying number is in fact non-zero,
since the constructor will panic and crash the program otherwise.

Using this, I can define a safe version of division (that avoids dividing by
zero) as:

```rust
fn safe_div(a: u32, b: NonZeroU32) -> u32 {
  a / b.as_u32()
}
```


## Racket Contracts

What I've come to realize is that using newtypes this way is similar to how
contracts can be used in Racket. In Racket, I could define an equivalent
function using a contract:

```racket
(define/contract (safe-div a b)
  ; contract
  (->
   number?
   (and/c (lambda (b) (not (= b 0))) number?) ; non-zero denominator
   number?)
  
  ; implementation
  (/ a b))
```

This ensures that `b` is never zero. The way this works, though, is by inserting
a check at _runtime_ that will crash the program if `b` is zero.

This is very similar to how `NonZeroU32` is implemented above. If the check
fails, the program crashes.

> Side note: The biggest difference is that with Rust newtypes, the check only occurs
> once per construction, while with Racket's contracts, the check may occur more
> than once.
> 
> ```rust
> fn foo(n: NonZeroU32) {
>     bar(n)
> }
>
> fn bar(n: NonZeroU32) {}
>
> // check occurs once here
> let n = NonZeroU32::new(1);
> 
> foo(n)
> ```
>
> ```racket
> (define/contract (foo a)
>   (->
>    ; check occurs once here
>    (and/c (lambda (a) (not (= a 0))) number?)
>    void?)
> 
>   (bar a))
> 
> (define/contract (bar a)
>   (->
>    ; and once here
>    (and/c (lambda (a) (not (= a 0))) number?)
>    void?)
>   
>   (void))
> 
> (foo 1)
> ```
> 


## Gradual Typing

These contracts reminded me of some [research that I
did](https://static.texttotypes.com/f9652ac4-2c5c-11f0-b5e0-3aa5a1c13538-gradual-typing.pdf)
on gradual typing. Gradual typing can be implemented in two different ways: **type
erasure** (like Typescript does) and **type contracts** (like Rhombus does).

> **NOTE:** There's technically more nuance to this, since type contracts
> can take different shapes depending on the type system. The
> methods that I found in my research were _erasure_, _transient
> checking_, _natural checking_, and _concrete checking_. See [my research
> paper](https://static.texttotypes.com/f9652ac4-2c5c-11f0-b5e0-3aa5a1c13538-gradual-typing.pdf)
> for more details.

### Type Erasure (Typescript)

One is the "Typescript" way, which basically says that "if you give me a value with
an unknown type and tell me what its type is, I will blindly believe you". In other
words, **no runtime checks are performed**.

```ts
function foo(x: any): number {
  let y: number = x;
  let z = y + 1;
  return z;
}

foo(1); // -> 2
foo("hello"); // -> "hello1"

// ... becomes ...
function foo(x) {
  let y = x;
  let z = y + 1;
  return z;
}

foo(1); // -> 2
foo("hello"); // -> "hello1"
```


### Type Contracts (Rhombus)

The other way is to insert runtime checks at the boundaries. Whenever we want to
check the type of a value whose type we can't statically determine, we insert
a check at runtime that fails if the type isn't the type expected.

> **NOTE:** this is a sort of approximation of how Rhombus' type checking works. For
> more details, see [Rhombus documentation](https://docs.racket-lang.org/rhombus-guide/classes_and_patterns.html)

```rhombus
fun foo(x) :: Number:
  let (y :: Number) = x
  let z = y + 1
  z

foo(1) // -> 2
foo("hello") // -> runtime error

// ... becomes ...
fun foo(x):
  let y = x
  assert y is a number

  let z = y + 1

  let _result = z
  assert _result is a Number

  _result

foo(1) // -> 2
foo("hello") // -> runtime error
```


## Specification Verification

In the language [Dafny](https://dafny.org/), it is possible to specify conditions
that must hold at the beginning or end of a function. For example, `safe_div` can
be defined as follows:

```dafny
method safe_div(a: int, b: int) returns (result: int)
    requires b != 0
{
    return a / b;
}
```

Notice that the `requires` specifies a condition that `b` must
meet. In order to call `safe_div`, you need to statically prove that `b`
is not `0`, such as by checking first that `b != 0`.

This is a really nice feature, but most languages don't support this kind
of specification or proof directly. For this reason, we need a way to get
this into the type system ourselves.


## "Gradual" Dependent Typing

In a similar way to how gradual typing techniques can be used to bridge the gap
between dynamic and static typing, we can also use newtypes to bridge the gap
between dynamic and static contracts.

If we wanted to take an approach similar to _Type Contracts_, we could implement `NonZeroU32`
as:

```rust
struct NonZeroU32(u32);

impl NonZeroU32 {
  fn new(n: u32) -> Self {
    assert!(n > 0, "n must not be zero");
    Self(n)
  }

  fn as_u32(&self) -> u32 {
    *self.0
  }
}
```

We could alternatively take the _Type Erasure_ approach and implement `NonZeroU32`
as:

```rust
struct NonZeroU32(u32);

impl NonZeroU32 {
  fn new(n: u32) -> Self {
    // no assertion, trust the user
    Self(n)
  }

  fn as_u32(&self) -> u32 {
    *self.0
  }
}
```

Personally, I feel like the first approach is most useful. Either way, though,
newtypes can be used to move contracts from a dynamic world to a static world.


## Conclusion

This idea of using newtypes as contracts feels like a powerful tool to me. It's
one way to move as much as possible to the type system so that logic bugs can
be caught _when the developer compiles the code_ rather than _when the user uses
it_. As I've written these ideas down, I've realized that, because some things
can't be completely expressed statically, there is value in finding ways to
bridge the dynamic and static world.

> I've also realized that dependent type "proofs" can have
> practical uses. [PyO3](https://pyo3.rs/) passes around [a
> `Python<'py>` token that proves that you have access to the Python
> interpreter](https://pyo3.rs/main/python-from-rust.html#the-py-lifetime). When
> you use capability passing, the capabilities that you pass around are proofs
> that you have access to a given resource. I guess that's a post for another
> day, though.
