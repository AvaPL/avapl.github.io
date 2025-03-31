---
title: Why you don't need refined types in production?
date: 2025-03-31 12:00:00 +0000
categories: [ Blog ]
tags: [ scala, functional programming, ddd, domain-driven design, tips ]     # TAG names should always be lowercase
media_subpath: /assets/img/2025-03-31-why-you-dont-need-refined-types-in-production/
---

Scala's ecosystem is full of examples that proves that its type system is exceptional. For instance, you can
implement [WHILE programming language using only types]( https://scastie.scala-lang.org/IbyH3g4qQladbPe9rcGGzg)
or [model a whole domain](https://medium.com/virtuslab/data-modeling-in-scala-3-but-i-only-use-types-b6f11ead4c28).
While these examples are rather extreme and not very practical, leveraging Scala's type system started to seep into our
production code via various libraries.

In this post, I want to share my thoughts regarding so-called "refined types". Examples of the libraries that provide
them are:

- [refined](https://github.com/fthomas/refined)
- [iron](https://github.com/Iltotore/iron)

As you might have guessed from the title, I'm not a big fan of refined types. While I'm impressed by their capabilities,
I don't think they are that easy to maintain in the long run. In this post, I will explain why I think that way and what
I would recommend instead.

[//]: # (TODO: Add image)

## What are refined types?

Refined types in Scala allow you to impose additional constraints on existing types at the type level, ensuring
correctness at compile-time. Instead of just defining a type like `Int` or `String`, refined types let you specify
constraints such as "this integer must be positive" or "this string must be non-empty".

Let's see an example of a simple refined type implemented via the `iron` library (which is newer than `refined` and
supports Scala 3 better):

```scala
type PositiveInt = Int :| Positive
val age: PositiveInt = 25 // valid
// val invalidAge: PositiveInt = -5 // compile-time error
```

We have a base type `Int` and we are refining it with a constraint `Positive`.

Let's now look at a more fancy example, which validates an email via a simplified regex:

```scala
type Email = String :| Match["""^[a-z0-9]+@[a-z0-9]+\.[a-z]{2,3}$"""]
val validEmail: Email = "user@example.com" // valid
// val invalidEmail: Email = "invalid-email" // compile-time error
```

Quite impressive, right? We can also validate the input at runtime using `refineV`:

```scala
val input: String = "user@example.com"
val validatedEmail: Either[String, Email] = input.refineEither

validatedEmail match {
  case Right(email) => println(s"Valid email: $email")
  case Left(error) => println(s"Error: $error")
}
```

This is only a tip of the iceberg. `iron` provides a lot of other features, such as combining multiple predicates,
creating custom predicates, and even integrating with other libraries. You can run the examples above
yourself [here in Scastie](https://scastie.scala-lang.org/5P05Y4KRQg2pQsXUgGCkfw).

## Benefits of using refined types

## The problems

### Validations are mostly needed during runtime

### Backward/forward compatibility of models

### Compatibility with libraries

[//]: # (TODO: Matrix of libraries compatibility with refined types)

[//]: # (TODO: chimney transformations)

## Alternatives

### Smart constructors with exceptions

### Smart constructors with Option/Either

### Smart constructors with Validated
