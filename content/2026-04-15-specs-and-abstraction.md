+++
title = "Specifications and Abstraction"
description = "Specifications are just another layer of abstraction"
date = 2026-04-15
draft = true

[taxonomies]
tags = ["specification", "ai"]

[extra]
repo_view = true
+++

When it comes to coding with AI, one of the recent trends is to use "spec-driven development", where everything is written down in a Markdown file that the AI can reference.

One of the biggest arguments that I've heard against this kind of development can be summarized by this comic:

![A comic from commit strip that points out that a precise specification is equivalent to code](https://www.commitstrip.com/wp-content/uploads/2016/08/Strip-Les-specs-cest-du-code-650-finalenglish.jpg)
[Source](https://www.commitstrip.com/en/2016/08/25/a-very-comprehensive-and-precise-spec/)

I started thinking about the merits and flaws of spec-driven development after having a conversation with my uncle about his experience playing around with it. I read a couple articles about people's thoughts on specifications[^1][^2][^3], and I've found that a lot of people don't think that they could replace code.

However, after reading an article by Hillel Wayne titled [_A sufficiently comprehensive spec is not (necessarily) code_](https://buttondown.com/hillelwayne/archive/a-sufficiently-comprehensive-spec-is-not/), I've started thinking about how specifications might become the new code. In his article, Hillel said something that really stood out to me:

> A specification is an **abstraction** of code.

This idea that a specification is an abstraction started my wheels turning.
## Levels of abstraction

Programming languages are often classified as _low level_ or _high level_. While this classification is often treated as a dichotomy, in reality, it's more of a spectrum. A really low level "language" could be **assembly**. This is really close to the exact operations of the machine.

However, more often than not, we don't care too much about exactly which register is used to store a value or which calling convention we are using. So, we can use the **C programming language** with **a C compiler**. The compiler takes care of producing the exact assembly. It decides how registers will be allocated and which function arguments get placed where.

Adding the abstraction of the C compiler gives us the following benefits:
- **We no longer have to understand the exact workings of the computer** to be able to write a working program
- The compiler is smart enough that **it can add optimizations** that may be to complicated for us to add by hand
- The code that we write is now more understandable to someone else that needs to modify it

We can then add another layer of abstraction and add something like **Python**. This allows us to not have to worry about how the memory is being laid out. It doesn't matter whether the memory is on the stack or the heap, Python takes care of that for us. Plus, thanks to Python's automatic memory management, we never have to worry about memory leaks.[^4]

By increasing the amount of abstraction that we are using, we are able to do more things with less effort. Writing a web server is faster in Python than in C, and both are faster than writing a web server in assembly. This is because we worry less about the details of the implementation and more about the high level concepts. 

## Abstraction is giving up control

The downside of writing a web server in assembly is the same as the upside: we no longer have control over memory. This means that the memory usage may be extremely inefficient. A Python web server would be really slow compared to a C web server.

When we add layers of abstraction, we give up control of the whatever we are abstracting.

## Specifications are a higher-level language

Let's come back to what Hillel said about specifications.

> A specification is an **abstraction** of code.

In other words, maybe a specification is just _a higher level language_. It allows us to worry less about _how the program is implemented_ and more about _what the program does_.

This means that a specification may bring benefits like allowing for the use of an optimal algorithm or architecture. By using AI to implement the specification, we no longer have to do all of the research to rediscover the same architecture that has been used 10,000 times over, nor do we have to run into the same problems that others have run in to.

However, because we give up control of the _how_, we no longer understand the _why_. When we implement the code by hand, we discover for ourselves the best way to do something (or at least how _not_ to do it). Then, if someone questions our code, we can justify our decision. If there is a bug, we know how to navigate the code base. This is a lot harder to do when we don't understand the code.

So using specifications comes with a trade-off. It can be a lot faster to implement, and we may be able to take advantage of industry-standard algorithms and architectures. However, we give up control of _how_ the specification is implemented.

## Conclusion

Specifications won't completely replace code in the same way that Python hasn't replaced C, or C hasn't (completely) replaced assembly. There are still use cases for each of the varying levels of abstraction. However, some projects may be able to give up control of implementation details in exchange for faster implementation.

This post definitely doesn't cover everything that I've been thinking about. There are quite a few other topics that I'd like to cover in future posts, including:
- Specifications and non-determinism
- Different levels of specification
- The importance of learning lower-level languages
- How programming levels can assist in specification implementation

Hopefully I'll be able to get around to writing them down some time soon!

[^1]: https://haskellforall.com/2026/03/a-sufficiently-detailed-spec-is-code

[^2]: https://codemanship.wordpress.com/2026/01/15/yeah-about-your-precise-specification/

[^3]: https://medium.com/@olmedo2408/code-is-not-dead-why-specifications-arent-the-full-story-40a654736856

[^4]: Yes, I know there are technically ways to get memory to leak in Python, this is a generalization for the sake of making a point
