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

Before I try to convince you that refined types aren't always a good idea, let me first describe some of the benefits
they provide.

### Elimination of invalid states

Constraints in refined types are enforced on the type level. This means that every instance of a class that uses refined
types has to satisfy the constraints. This eliminates the possibility of invalid states at compile time.

[//]: # (@formatter:off)

```scala
case class User(
  username: String :| MinLength[3],
  age: Int :| Positive,
  website: String :| ValidURL
)
```

[//]: # (@formatter:on)

In the example above, we can be sure that every `User` instance will have a valid username, age, and website URL.

### Improved readability and reduction of boilerplate

This one is a bit subjective. The biggest advantage I see in refined types that set them apart from any other means of
validation is that validation is described right next to the type. You don't have to look for the validation logic
somewhere else in the codebase. Of course, if you inline a lengthy regex, it might be hard to read, but if used wisely,
it can improve the readability of the code.

### Runtime validation utils

Every refined type can be used with various methods that provide runtime validation. In `iron`, you can use:

- `refineUnsafe` - to throw an exception if the value doesn't satisfy the constraints
- `refineOption` - to return `None` if the value doesn't satisfy the constraints
- `refineEither` - to return `Left` with an error message if the value doesn't satisfy the constraints
- `Cats` or `ZIO` integration to accumulate validation errors

This gives you a lot of flexibility in how you want to handle invalid values.

## The problems

It's not all sunshine and rainbows, though. I'd say that the problems with refined types outweigh the benefits in the
long run. Here are some of the pitfalls you may see if you get distracted by shiny.

### Validations are mostly needed during runtime

When I read documentation or articles about refined types, I often see examples that hail their compile-time validation
as the main benefit. The reality is that most often our applications are driven by the dreaded side effects. This could
be user input, database queries, or API calls. In these cases, the validation is needed at runtime, not compile time.
The majority of developer-written instances are created in the test code, where we want to validate our code anyway, so
we assume that the input is valid, at least at the type level.

You may say: "Okay, but if the validations are needed mostly at runtime, why not just drop the types completely and use
JavaScript?". Let's see a fragment of
the [definition of a type system from Wikipedia](https://en.wikipedia.org/wiki/Type_system):

> A type system dictates the operations that can be performed on a term. For variables, the type system determines the
> allowed values of that term.
{: .prompt-info }

I'm not saying that's the only definition of a type system, but I think it describes the essence of it. A type system
not only provides guarantees about the values but also determines the operations that can be performed on those values.
From my point of view, the operations are the most important part of the type system. At the end of the day, our
applications are designed to manipulate data that we assume is valid.

Here are some examples of what I mean:

- When I represent a URL, I want to be able to extract the protocol from it,
- When I represent a phone number, I want to be able to determine if it's a mobile or landline number,
- When I represent a temperature, I want to be able to convert it to different units.

All these operations require some assumption about the structure of the data. The domain logic for them has to be in
some way encapsulated in the classes used to represent it. What is more, the operations may also require some validation
after the result is calculated. For example, I can invent my own temperature scale that represents only temperatures
above the boiling point of water. In this case, not every value will be convertible to a different unit.

Operations happen at runtime and only because we assume that the data we execute them on is valid, we get the safety
guarantees. The thing is that validity of the data is also a runtime concern. We cannot be sure that the data we receive
from external sources is valid. Therefore, we can assume that creating an instance of a class is just another operation,
which may fail.

Here's a comparison between a `RefinedUser` built with refined types and a `ClassicUser` built with a constructor:

[//]: # (@formatter:off)

```scala
case class RefinedUser(username: String :| MinLength[3], age: Int :| Positive) {
  val firstInitial: Char = username.head // domain logic
  val secondInitial: Char = username.charAt(10) // buggy domain logic
}

object RefinedUser {
  def create(username: String, age: Int): RefinedUser = // runtime validation
    RefinedUser(username.refineUnsafe, age.refineUnsafe)
}

case class ClassicUser private (username: String, age: Int) {
  val firstInitial: Char = username.head // domain logic
  val secondInitial: Char = username.charAt(10) // buggy domain logic
}

object ClassicUser {
  def create(username: String, age: Int): ClassicUser = { // runtime validation
    if (username.length < 3) throw new IllegalArgumentException("Username must be at least 3 characters long")
    if (age < 0) throw new IllegalArgumentException("Age must be positive")
    ClassicUser(username, age)
  }
}
```

[//]: # (@formatter:on)

In my opinion, at runtime, refined types are just a glorified constructors. They don't provide any additional operations
that can be performed on the data. They just validate the instance when it's created. Even if the validation passes, we
still have to write the logic to manipulate the data, even though we assume it has a well-defined structure. What is
more, we still have to test out domain logic as the refined type doesn't prevent us from performing invalid operations
as in `secondInitial` example above.

### Fields depending on other fields

[//]: # (TODO: Example of a computer/car configuration)

### Backward/forward compatibility of models

### Compatibility with libraries

[//]: # (TODO: Matrix of libraries compatibility with refined types)

[//]: # (TODO: chimney transformations)

## Alternatives

### Smart constructors with exceptions

### Smart constructors with Option/Either

### Smart constructors with Validated
