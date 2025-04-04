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

![Fluffy monsters addressing danger](fluffy_monsters_addressing_danger.png){: w="350"}

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

> At the moment of writing this post, the highlighting in the code blocks doesn't support literals as types properly.
> Please ignore those few places where some parts are red for no reason.
{: .prompt-info }

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
    if (username.length < 3) throw new UsernameTooShortException(username)
    if (age < 0) throw new NegativeAgeException(age)
    ClassicUser(username, age)
  }
}
```

[//]: # (@formatter:on)

> Note: I'm using exceptions just for the sake of simplicity. In further sections, I'll list some alternatives that are
> native to functional world.
{: .prompt-info }

In my opinion, at runtime, refined types are just a glorified constructors. They don't provide any additional operations
that can be performed on the data. They just validate the instance when it's created. Even if the validation passes, we
still have to write the logic to manipulate the data, even though we assume it has a well-defined structure. What is
more, we still have to test out domain logic as the refined type doesn't prevent us from performing invalid operations
as in `secondInitial` example above.

We can also see the first inflexibility of refined types validation. In case of `ClassicUser`, we could throw any error
type with any message. For instance, we could throw a custom `EmptyStringError` when the string is empty,
and `StringTooShortError` when it's too short. With refined types, we always get an `IllegalArgumentException` with a
predefined message that the predicate is not satisfied.

### Fields depending on other fields

Continuing the topic of validation, usually an aggregate in domain driven design has to fulfill some invariants. That
doesn't always mean that only the fields have a valid structure, but that the whole object is valid in domain terms.

Let's use a simple example of a domain in which the business sells laptops. The constraints are:

- the laptop has 8 or 10 core CPU,
- the laptop has 16, 24 or 32 GB of RAM,
- the laptop has 512 GB or 1 TB SSD,
- when the laptop has 10 core CPU, it must have at least 24 GB of RAM.

We can easily represent the first 3 constraints even without refined types, by using Scala union types:

[//]: # (@formatter:off)

```scala
case class LaptopConfiguration(
  cpuCores: 8 | 10,
  ramGB: 16 | 24 | 32,
  ssdGB: 512 | 1024
)
```

[//]: # (@formatter:on)

However, the last constraint is not possible to represent on the type level in simple way. I assume that there is some
fancy Scala magic that could be used to represent it, but I don't think it's worth the effort and further confusion. If
we had a simple constructor, the validation is very straightforward:

[//]: # (@formatter:off)

```scala
case class LaptopConfiguration private (
  cpuCores: Int,
  ramGB: Int,
  ssdGB: Int
)

object LaptopConfiguration{
  def create(cpuCores: Int, ramGB: Int, ssdGB: Int): LaptopConfiguration = {
    if (!Set(8, 10).contains(cpuCores)) throw new InvalidCpuCoresException(cpuCores)
    if (!Set(16, 24, 32).contains(ramGB)) throw new InvalidRAMException(ramGB)
    if (!Set(512, 1024).contains(ssdGB)) throw new InvalidSSDException(ssdGB)
    if (cpuCores == 10 && ramGB < 24) throw new TooLowRamForCPUException(cpuCores, ramGB)
    LaptopConfiguration(cpuCores, ramGB, ssdGB)
  }
}
```

[//]: # (@formatter:on)

Again, we get full flexibility of the validation. We can throw any exception we want with any message. With refined
types, we'd have to apply the validation of the last constraint in constructor anyway. This is also true for any other
validation of our aggregate after we apply some domain logic on it.

> Some of you might suggest introducing a separate type for each possible configuration, that'd also prevent us from
> creating an invalid state. While for some domains it's possible, for others it may be
> impractical. [Here](https://virtuslab.com/success-stories/car-configurator-modernization/) is an example of a domain
> in which validation of the whole configuration was one of the main concerns because the number of parameters was so
> huge.
{: .prompt-info }

### Backward/forward compatibility of models

In the world of agile development, we often have to deal with backward and forward compatibility of our models. The
requirements change, the data structure changes, and we have to adapt our code to it. This is especially true for
database models (which require at least backward compatibility) and API models (which often requires forward
compatibility).

Let's focus on the backward compatibility. When we change the structure of our models, we have to make sure that
the old data can still be read by the new code. How to do that with refined types? It actually depends on the type of
the change we're introducing.

Hipothetically, let's say we have the following sequence of business requirements:

1. Introduce an identifier that is contains exactly 4 characters that will be persisted in a database.
2. Allow identifiers that are 4 or 5 characters long.
3. Identifiers now have to be 5 characters long. If the identifier is 4 characters long, it should be padded with `0`.

Point 1 is easy to implement. We can just introduce a refined type with predicate `FixedLength[4]`:

```scala
type IdentifierV1 = String :| FixedLength[4]
```

Let's now try to implement point 2. We can use `MinLength[4]` and `MaxLength[5]` predicates:

```scala
type IdentifierV2 = String :| (MinLength[4] & MaxLength[5])
```

So far so good. Remember that we persisted `Identifier1` values in the database, and we have to maintain backward
compatibility. Because fixed length of 4 from `Identifier1` fulfils the requirements of `Identifier2`, we can say that
the change is backward-compatible. We can use a value of type `Identifier1` as a value of type `Identifier2`:

```scala
val myIdentifierV1: IdentifierV1 = "abcd"
val myIdentifierV2: IdentifierV2 = myIdentifierV1
```

We can run it and get... a compilation error.

> Cannot refine value at compile-time because the predicate cannot be evaluated.<br>
> This is likely because the condition or the input value isn't fully inlined.
>
> To test a constraint at runtime, use one of the `refine...` extension methods.

We could try to fix it, but I can't be bothered ¯\_(ツ)_/¯ Why? As mentioned above, we mostly care about the runtime
validation anyway. If we use one of the `refine...` methods as suggested in the error, it'll work just fine:

```scala
val myIdentifierV2: IdentifierV2 = myIdentifierV1.refineUnsafe // works
```

Okay, now to the point 3. We can just use `FixedLength[5]` predicate:

```scala
type IdentifierV3 = String :| FixedLength[5]
```

This change is not backward-compatible, because previously we allowed identifiers that were 4 characters long. Luckily,
the requirement describes the way in which we can transform the old identifiers to the new ones. We can just pad them
with `0`. But how are we supposed to do that? Well, again, with a transformation on the runtime:

```scala
val myIdentifierV2: IdentifierV2 = "abcd"
val myIdentifierV3: IdentifierV3 = myIdentifierV2.padTo(5, '0').refineUnsafe
```

Refined types again don't help us with the transformation. In this case, they don't differ from a regular transformation
that we'd apply manually between the models. We'd also have to write a few tests to make sure that the transformation
works as expected. It's another proof that runtime validation is intrinsic.

I could further describe forward compatibility or transitive compatibilities, but I think the point is clear. Other
kinds of compatibility fall into the same category, in which we are required to remember about the types we used in the
past.

### Compatibility with libraries

Another caveat is the support of libraries. I think it's quite obvious that not every library that we use will have
support for refined types. For instance, [avro4s supports refined](https://github.com/sksamuel/avro4s#refined-support)
but [iron doesn't support avro4s](https://github.com/Iltotore/iron/issues/215). Of course, I don't want to say that
refined types are the culprit here. It's in the nature of many libraries in the ecosystem. However, because we're
talking about types here, it's quite cumbersome to write all the boilerplate, e.g. Avro codecs, yourself when you have
lots of models in your domain. You wouldn't have that problem if you used regular types. Again, we care about runtime
transformations so runtime validation on regular types doesn't differ that much from refined types with bindings for
other libraries.

Another aspect worth highlighting are how those bindings for libraries work. Let's look at `iron-circe` integration for
decoding `RefinedUser` defined above:

[//]: # (@formatter:off)

```scala
case class RefinedUser(username: String :| MinLength[3], age: Int :| Positive)

object RefinedUser {
  def fromJson(json: Json): Either[Error, RefinedUser] =
    decode[RefinedUser](json)
}

val validUser = RefinedUser.fromJson("""{"username": "John", "age": 21}""")
val invalidUser = RefinedUser.fromJson("""{"username": "Jo", "age": -100}""")
```

[//]: # (@formatter:on)

For valid user, we just get a `RefinedUser` instance. For invalid user, we get a `io.circe.DecodingFailure`. Great,
isn't it? Well, in my opinion, it isn't. `io.circe.DecodingFailure` is a very generic exception. If you wanted to
introduce error handling based on it, e.g. display custom messages, introduce fallbacks, the only way to discover what
went wrong is by inspecting the error message. This is not what functional programming is praised for - we love not only
types in our models, we love types of the exceptions.

What would happen if we implemented it for `ClassicUser`? As you might have guessed, full flexibility:

[//]: # (@formatter:off)

```scala
case class ClassicUser private (username: String, age: Int)

object ClassicUser {
  def fromJson(json: String): Either[Error, ClassicUser] =
    decode[ClassicUser](json).map { user =>
      if (user.username.length < 3) throw new UsernameTooShortException(user.username)
      if (user.age < 0) throw new NegativeAgeException(user.age)
      user
    }
}

val validUser = ClassicUser.fromJson("""{"username": "John", "age": 21}""")
val invalidUser = ClassicUser.fromJson("""{"username": "Jo", "age": -100}""")
```

[//]: # (@formatter:on)

Of course, for simplicity, I mixed exceptions with an `Either` here. However, even despite that odd mixture, another
benefit is revealed. We separate the domain exceptions (which we could potentially handle) from errors from `circe`
decoders (which are probably irrecoverable). Another point for the classic approach.

The last thing about libraries interoperability that I want to mention is `chimney` and `ducktape`. Both libraries are
widely used to transform data between models e.g. domain to database, API to domain, etc. They both provide a lot of
value by reducing the boilerplate significantly. Unfortunately, if both the source and target models use refined types,
the transformation is not seamless. You have to define the transformers manually, similarly as you'd do without using
refined types. However, I found one positive aspect - if you transform a refined type to a regular type, it works
without any boilerplate in both `chimney` and `ducktape`. Here's an example for `chimney`:

[//]: # (@formatter:off)

```scala
case class User(username: String :| MinLength[3], age: Int :| Positive)
case class DatabaseUser(username: String, age: Int)

val refinedUser = User("John", 21)
val databaseUser = refinedUser.transformInto[DatabaseUser] // or .to[...] for ducktape
```

[//]: # (@formatter:on)

This can be handy when have a strict model (e.g. for the domain) and a more permissive one (e.g. for database).

## Alternatives

Okay, so, you now know why I think refined types are not the best solution for production code. But what do I recommend
instead? Well, I think the best solution is to use regular types and smart constructors. This is nothing new, nothing
fancy, and widely used across many programming languages. They provide the same benefits at runtime as refined types and
much greater flexibility, at least in my opinion.

Despite smart constructors being a well-known pattern, I'd like to show a few examples of how to implement them in
Scala. We have quite a few options, and you can choose the one that fits your needs the best.

Throughout this section, I'll use the following example of a `User` class:

[//]: # (@formatter:off)

```scala
case class User private (username: String, age: Int)
```

and related exceptions:

```scala
class UsernameTooShortException(username: String) extends RuntimeException(s"Username '$username' is too short")
class NegativeAgeException(age: Int) extends RuntimeException(s"Age '$age' is negative")
type UserValidationException = UsernameTooShortException | NegativeAgeException
```

[//]: # (@formatter:on)

### Smart constructors with exceptions

This one is the simplest and most straightforward. You just throw an exception if the validation fails. Actually, this
is the approach I used in the examples above. It's simple, easy to understand, and doesn't require any additional
libraries. Here's an example how it'd look like for the `User` class:

[//]: # (@formatter:off)

```scala
def smartConstructorWithExceptions(username: String, age: Int): User = {
  if (username.length < 3) throw UsernameTooShortException(username)
  if (age < 0) throw NegativeAgeException(age)
  User(username, age)
}
```

[//]: # (@formatter:on)

The downsides?

- Exceptions are side effects, and you have to remember about handling them
- The constructor fails on the first validation error, so you don't get all the errors at once

Let's see other alternatives.

### Smart constructors with Option/Either

This is a more functional approach. Instead of throwing exceptions, you return an `Option` or `Either` type. Here's how
it'd look like for the `User` class with `Option`:

[//]: # (@formatter:off)

```scala
def smartConstructorWithOption(username: String, age: Int): Option[User] =
  Option.unless(username.length < 3 || age < 0) {
    User(username, age)
  }
```

[//]: # (@formatter:on)

While it's more functional, we lost information what went wrong. We can only tell if the validation passed or not.

> [Here](https://medium.com/virtuslab/option-the-null-of-our-times-77f3bfd7c234) is an interesting article about
> how `Option` can be percieved as a `null` of our times.
{: .prompt-tip }

We can use `Either` to retain the information about the error:

[//]: # (@formatter:off)

```scala
def smartConstructorWithEither(username: String, age: Int): Either[UserValidationException, User] =
  if (username.length < 3)
    Left(UsernameTooShortException(username))
  else if (age < 0)
    Left(NegativeAgeException(age))
  else
    Right(User(username, age))
```

[//]: # (@formatter:on)

While we now have the information about the error, we still fail on the first validation error. Can we do better?

### Smart constructors with Validated

For me, the best solution is to use `Validated` type from `cats` or similar. This way we get the benefits of `Either`,
while also being able to accumulate the errors. Here's how it looks like for the `User` class:

[//]: # (@formatter:off)

```scala
def smartConstructorWithValidated(username: String, age: Int): ValidatedNel[UserValidationException, User] =
  (
    if (username.length < 3) (UsernameTooShortException(username): UserValidationException).invalidNel else username.validNel,
    if (age < 0) (NegativeAgeException(age): UserValidationException).invalidNel else age.validNel
  ).mapN(User.apply)
```

[//]: # (@formatter:on)

> Note: we have to help the compiler a bit by explicitly widening the exception type to `UserValidationException`,
> otherwise `mapN` won't be able to concatenate the list of errors.
{: .prompt-info }

Of course, you can extract each validation to a separate method to make it more readable or to be able to reuse it. With
this approach we get either a valid `User` instance or a list of all validation errors that occurred. In my opinion, in
most cases this is the best solution.

You can find the code for the examples above [on Scastie](https://scastie.scala-lang.org/03eGrI5tRp2F9VwNzbUH3A).

## Summary

And that would be it. I hope that even if you have a different opinion than mine, the post didn't discourage you from
thinking about the topic. I think that refined types are a great tool, but I don't see any broad use cases for them in
production code. We have a powerful type system in Scala, and we should use it to our advantage. This doesn't
necessarily mean that we have to use refined types. In my opinion, we can achieve much better results with regular types
and smart constructors. They provide the same benefits at runtime, while also being more flexible and easier to maintain
in the long run.

Also, novadays we quite often raise the topic of how easy it is to onboard new developers to Scala. I think that refined
types are a great example of how we can make it harder. It's another tool to learn, and another library to maintain.
Smart constructors are a well-known pattern, and most developers will be able to understand them without any additional
learning curve. While compilation errors might be useful, I think that overall a well-defined validation logic is much
more readable and robust (but always remember to test it!).

If you have any questions or comments to share, feel free to leave them below. I'm always happy to discuss different
opinions and learn about your experiences. Thanks for reading!
