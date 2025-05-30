---
title: Why you don't need refined types in production?
date: 2025-04-05 12:00:00 +0000
categories: [ Blog ]
tags: [ scala, functional programming, ddd, domain-driven design, tips ]     # TAG names should always be lowercase
media_subpath: /assets/img/2025-04-05-why-you-dont-need-refined-types-in-production/
---

Scala's ecosystem is full of examples that prove just how powerful its type system really is. We can, for
instance, [implement the WHILE programming language using only types](https://scastie.scala-lang.org/IbyH3g4qQladbPe9rcGGzg)
or even
[model an entire domain purely with types](https://medium.com/virtuslab/data-modeling-in-scala-3-but-i-only-use-types-b6f11ead4c28).

Sure, these examples are a bit extreme and not exactly practical, but the influence of Scala's type system has
definitely made its way into real-world code — especially through some clever libraries.

In this post, I want to share my thoughts on the idea of "refined types." A couple of popular libraries in this space
are:

- [refined](https://github.com/fthomas/refined)
- [iron](https://github.com/Iltotore/iron)

As you might have guessed from the title, I'm not the biggest fan of refined types. Don't get me wrong — I'm genuinely
impressed by what they can do. But when it comes to long-term maintenance, I find them a bit... tricky. So, let me walk
you through why I feel this way — and what I'd suggest doing instead.

![Fluffy monsters addressing danger](fluffy_monsters_addressing_danger.png){: w="350"}

## What are refined types?

Refined types in Scala let you add extra constraints to existing types at the type level, so you can catch more
errors at compile time. Instead of just saying something is an `Int` or a `String`, you can say things like "this
integer must be positive" or "this string must be non-empty."

Here's a simple example using the `iron` library (which is newer than `refined` and has better support for Scala 3):

```scala
type PositiveInt = Int :| Positive
val age: PositiveInt = 25 // valid
// val invalidAge: PositiveInt = -5 // compile-time error
```

We're taking a regular `Int` and refining it with the constraint `Positive`.

Let's take it up a notch with a slightly fancier example. This one validates an email using a simplified regex:

```scala
type Email = String :| Match["""^[a-z0-9]+@[a-z0-9]+\.[a-z]{2,3}$"""]
val validEmail: Email = "user@example.com" // valid
// val invalidEmail: Email = "invalid-email" // compile-time error
```

> At the time of writing, the syntax highlighting in code blocks doesn't fully support literals used as types. So if you
> see some weird red squiggles — don't worry about them.  
{: .prompt-info }

Pretty cool, right? You can also validate values at runtime, for example by using `refineEither`:

```scala
val input: String = "user@example.com"
val validatedEmail: Either[String, Email] = input.refineEither

validatedEmail match {
  case Right(email) => println(s"Valid email: $email")
  case Left(error) => println(s"Error: $error")
}
```

And this is just the tip of the iceberg. `iron` offers plenty more — like combining multiple predicates, defining your
own ones, and integrating with other libraries. If you want to try these examples
yourself, [you can run them in Scastie](https://scastie.scala-lang.org/5P05Y4KRQg2pQsXUgGCkfw).

## Benefits of using refined types

Before I try to convince you that refined types aren't always the best idea, let's give credit where it's due — they do
have some bright spots.

### Elimination of invalid states

Refined types enforce constraints at the type level. That means any value used in our code must already satisfy those
constraints — right from the moment it's created. The result? No more invalid states sneaking in at compile time.

[//]: # (@formatter:off)

```scala
case class User(
  username: String :| MinLength[3],
  age: Int :| Positive,
  website: String :| ValidURL
)
```

[//]: # (@formatter:on)

In this example, every `User` instance is guaranteed to have a username that's long enough, an age that's positive, and
a valid website URL.

### Improved readability and less boilerplate

This one's a bit subjective, but here's the biggest benefit from my point of view: with refined types, the validation
lives right next to the type definition. We don't have to hunt through the codebase to figure out what's being
checked — it's all there in front of us. Sure, if we inline a monster regex, things can get a little messy. But when
used with care, refined types can sometimes make our code easier to understand.

### Runtime validation utils

Refined types also come with handy runtime validation tools. In `iron`, you can pick the flavor that best suits your
needs:

- `refineUnsafe` – throws an exception if the validation fails
- `refineOption` – returns `None` if the validation fails
- `refineEither` – returns a `Left` with an error message if the validation fails
- Integration with `Cats` or `ZIO` – to accumulate multiple validation errors

This gives you some flexibility in how you want to handle bad data — whether you prefer exceptions, `Option`s, or richer
error handling.

## The problems

It's not all sunshine and rainbows though. In my experience, the drawbacks of refined types tend to outweigh the
benefits over time. Here are some of the pitfalls you may encounter if you get distracted by shiny.

### Validations are mostly needed during runtime

Articles, blog posts, and docs praise refined types for their compile-time validation. And sure, that's neat in theory.
But in practice? Most of our applications revolve around messy, real-world side effects — user input, database records,
API responses. All of these require *runtime* validation, not compile-time checks.

Do a little exercise and take a look at your codebases. How many times have you defined an instance of a class with
static values, resolvable at compile time? I bet it's not that often. In my experience, the majority of
developer-created values in a production-grade application live in test code, where we're assuming valid input anyway.
So the compile-time constraints don't do much heavy lifting there.

You might be thinking: *"Okay, but if it all comes down to runtime anyway, why not just toss types altogether and go
full JavaScript?"* Fair question. But here's a snippet
from [Wikipedia's definition of a type system](https://en.wikipedia.org/wiki/Type_system):

> A type system dictates the operations that can be performed on a term. For variables, the type system determines the
> allowed values of that term.
{: .prompt-info }

That's not the only way to define a type system, but I think it captures the heart of it. A good type system isn't just
about what values are allowed — it's about what operations make sense for them. And in my view, *that's* the real value.
At the end of the day, our applications are designed to *do things* with valid data. The type system helps ensure we're
doing those things safely and correctly.

Here are some examples of what I mean:

- If I'm working with a URL, I want to be able to extract the protocol from it.
- If I have a phone number, I want to be able to tell whether it's a mobile or a landline.
- If I'm dealing with a temperature, I want to have options to convert it between different units.

Each of these operations relies on certain assumptions about the structure of the data. That logic has to live
somewhere, and ideally, it's encapsulated in the class representing the data.

And it doesn't stop at structure. Sometimes, operations require validation *after* doing their thing. Say I invent my
own temperature scale that only supports values above the boiling point of water. In this case, not every temperature
can be converted to that scale, and we have to capture that.

These operations happen at runtime. And the only reason they feel safe is because we *assume* the data we're working
with is valid. The thing is: data validity is also a runtime concern. We can't trust that values coming from users,
APIs, or databases are always correct. Therefore, we can say creating an instance of a class is just another operation.
And like any operation, it can fail.

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
    if (username.length < 3) throw UsernameTooShortException(username)
    if (age < 0) throw NegativeAgeException(age)
    ClassicUser(username, age)
  }
}
```

[//]: # (@formatter:on)

> Note: I'm using exceptions here just to keep things simple. In later sections, I'll go over some more functional
> alternatives.  
{: .prompt-info }

At runtime, refined types feel like glorified constructors. They don't offer any extra operations we can perform on the
data — they just validate the input when an instance is created. And once the object exists, we're on our own. We
still have to implement the logic that actually *uses* the data, assuming it's valid. And we still have to write tests
for that logic, because as shown in the `secondInitial` example, refined types won't stop us from introducing bugs.

This is also where we start to see some limitations. With `ClassicUser`, we can throw any exception type, with a clear
and specific message. For instance, we might throw custom `EmptyStringError` or a `StringTooShortError`. With refined
types? We're stuck with a generic `IllegalArgumentException`. Of course, we can try to catch it and map it to our own
exception, but that's just more boilerplate, which is exactly what we're trying to avoid.

### Fields depending on other fields

Let's stay on the topic of validation for a moment. In domain-driven design, an aggregate often has to satisfy certain
invariants — not just "each field looks valid," but "the entire object makes sense in business terms."

Take a simple example from a laptop-selling domain. The business rules are:

- The laptop must have an 8-core or 10-core CPU
- It must have 16, 24, or 32 GB of RAM
- It must have either 512 GB or 1 TB of SSD storage
- If it has a 10-core CPU, it must have *at least* 24 GB of RAM

The first three constraints are easy to model even without refined types, using Scala 3's union types:

[//]: # (@formatter:off)

```scala
case class LaptopConfiguration(
  cpuCores: 8 | 10,
  ramGB: 16 | 24 | 32,
  ssdGB: 512 | 1024
)
```

[//]: # (@formatter:on)

However, the last constraint is not something we can easily represent at the type level. Sure, we could probably pull
off some fancy Scala magic to handle it, but I don't think it's worth the complexity or potential confusion.

If we use a simple constructor, the validation becomes straightforward:

[//]: # (@formatter:off)

```scala
case class LaptopConfiguration private (
  cpuCores: Int,
  ramGB: Int,
  ssdGB: Int
)

object LaptopConfiguration{
  def create(cpuCores: Int, ramGB: Int, ssdGB: Int): LaptopConfiguration = {
    if (!Set(8, 10).contains(cpuCores)) throw InvalidCpuCoresException(cpuCores)
    if (!Set(16, 24, 32).contains(ramGB)) throw InvalidRAMException(ramGB)
    if (!Set(512, 1024).contains(ssdGB)) throw InvalidSSDException(ssdGB)
    if (cpuCores == 10 && ramGB < 24) throw TooLowRamForCPUException(cpuCores, ramGB)
    LaptopConfiguration(cpuCores, ramGB, ssdGB)
  }
}
```

[//]: # (@formatter:on)

With this approach, we have complete flexibility in validation. We can throw any exception we like, with a custom
message that fits the context. With refined types, we'd still need to do the last bit of validation inside the
constructor anyway. This pattern holds true for any other aggregate validation after performing domain logic.

> Some of you might suggest introducing a separate type for each possible configuration, that'd also prevent us from
> creating an invalid state. While for some domains it's possible, for others it may be
> impractical. [Here](https://virtuslab.com/success-stories/car-configurator-modernization/) is an example of a domain
> in which validation of the whole configuration was one of the main concerns because the number of parameters was so
> huge.
{: .prompt-info }

### Backward/forward compatibility of models

In the world of agile development, we often need to navigate the challenges of backward and forward compatibility.
Requirements evolve, data structures change, and our code must adapt accordingly. This is especially critical for
database models (which typically need at least backward compatibility) and API models (which often require forward
compatibility).

Let's focus on backward compatibility. When the structure of our models changes, we must ensure that old data can still
be read by the new code. But how do we handle this with refined types? The answer depends on the nature of the change
we're introducing.

Hypothetically, let's imagine the following sequence of business requirements that are given to us over time:

**Requirement 1**: Introduce an identifier that must contain exactly 4 characters, which will be persisted in a
database.<br>
**Requirement 2**: Allow identifiers that are either 4 or 5 characters long.<br>
**Requirement 3**: The identifier must now always be 5 characters long. If the identifier was previously 4 characters
long, it should be padded with `0`.

Point 1 is easy to implement. We can simply introduce a refined type with the predicate `FixedLength[4]`:

[//]: # (@formatter:off)

```scala
type IdentifierV1 = String :| FixedLength[4]
```

[//]: # (@formatter:on)

Now, let's move on to point 2. We can use `MinLength[4]` and `MaxLength[5]` predicates:

[//]: # (@formatter:off)

```scala
type IdentifierV2 = String :| (MinLength[4] & MaxLength[5])
```

[//]: # (@formatter:on)

So far, so good. Remember, we've persisted `IdentifierV1` values in the database, and we need to ensure backward
compatibility. Since a 4-character `IdentifierV1` satisfies the requirements of `IdentifierV2`, we can say the change is
backward-compatible. We can use a value of type `IdentifierV1` as a value of type `IdentifierV2`:

[//]: # (@formatter:off)

```scala
val myIdentifierV1: IdentifierV1 = "abcd"
val myIdentifierV2: IdentifierV2 = myIdentifierV1
```

[//]: # (@formatter:on)

Now we can run it and get... a compilation error.

> Cannot refine value at compile-time because the predicate cannot be evaluated.<br>
> This is likely because the condition or the input value isn't fully inlined.
>
> To test a constraint at runtime, use one of the `refine...` extension methods.

At this point, we could try to fix it, but honestly, I can't be bothered ¯\\\_(ツ)_/¯ Why? As mentioned before, runtime
validation is what truly matters. If we use one of the `refine...` methods as suggested in the error, everything works
just fine:

[//]: # (@formatter:off)

```scala
val myIdentifierV2: IdentifierV2 = myIdentifierV1.refineUnsafe // works
```

[//]: # (@formatter:on)

Now, for point 3. We can just use the `FixedLength[5]` predicate:

[//]: # (@formatter:off)

```scala
type IdentifierV3 = String :| FixedLength[5]
```

[//]: # (@formatter:on)

This change is not backward-compatible, because we previously allowed identifiers with only 4 characters. Fortunately,
the new requirement specifies how we can transform old identifiers into the new format: by padding them with `0`. But
how do we achieve this? Well, once again, we perform the transformation at runtime:

[//]: # (@formatter:off)

```scala
val myIdentifierV2: IdentifierV2 = "abcd"
val myIdentifierV3: IdentifierV3 = myIdentifierV2.padTo(5, '0').refineUnsafe
```

[//]: # (@formatter:on)

Refined types don't help us with this transformation. In fact, they don't differ from a regular transformation we would
apply manually between models. We'd still need to write a few tests to ensure that the transformation works as expected.
This highlights once again that runtime validation is intrinsic.

I could continue discussing forward or transitive compatibilities, but I think the point is clear. All types
of compatibility fall into the same category, where we have to remember about the types we've used in the past and how
they affect our application.

### Compatibility with libraries

Another caveat is library support. I think it's quite obvious that not every library we use will have support for
refined types. For example, [avro4s supports refined](https://github.com/sksamuel/avro4s#refined-support),
but [iron doesn't support avro4s](https://github.com/Iltotore/iron/issues/215). Of course, I'm not saying that refined
types are the cause of this issue — it's a common challenge with many libraries in the ecosystem. However, when dealing
with types, it becomes cumbersome to write all the boilerplate, such as Avro codecs, manually, especially when we have
many models in our domain.

If we were using regular types, we wouldn't face this problem. Once again, because we care about runtime
transformations, the difference in runtime validation between regular types and refined types with external library
bindings isn't significant. The result at runtime is the same, but the latter requires more boilerplate to even get the
result, let alone validate it.

Another aspect worth highlighting is how the bindings for libraries work. Let's look at the `iron-circe` integration for
decoding the `RefinedUser` defined above:

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

For a valid user, we simply get a `RefinedUser` instance. For an invalid user, we get a `io.circe.DecodingFailure`.
Great, right? Well, in my opinion, not quite. `io.circe.DecodingFailure` is a very generic exception. If we wanted to
implement error handling, such as displaying custom messages or introducing fallbacks, the only way to determine what
went wrong is by inspecting the error message. This isn't exactly what functional programming is praised for — we not
only love types in our models, but also we love types of exceptions.

What would happen if we implemented this for a `ClassicUser`? As you might have guessed, we get full flexibility:

[//]: # (@formatter:off)

```scala
case class ClassicUser private (username: String, age: Int)

object ClassicUser {
  def fromJson(json: String): Either[Error, ClassicUser] =
    decode[ClassicUser](json).map { user =>
      if (user.username.length < 3) throw UsernameTooShortException(user.username)
      if (user.age < 0) throw NegativeAgeException(user.age)
      user
    }
}

val validUser = ClassicUser.fromJson("""{"username": "John", "age": 21}""")
val invalidUser = ClassicUser.fromJson("""{"username": "Jo", "age": -100}""")
```

[//]: # (@formatter:on)

Of course, for simplicity, I've mixed exceptions with an `Either` here. However, even despite this odd mixture, another
benefit comes to light. We can separate domain-specific exceptions (which we can potentially handle) from errors that
arise from `circe` decoders (which are probably irrecoverable). Another point in favor of the classic approach.

The last point about library interoperability I want to touch on concerns the use of `chimney` and `ducktape`. Both
libraries are widely used to transform data between models, such as from domain to database or API to domain. They
provide significant value by reducing boilerplate code. Unfortunately, if both the source and target models use
different refined types, the transformation process isn't seamless. Even when the transformation makes sense (e.g., from
`String :| FixedLength[3]` to `String :| MinLength[3]`) we need to define the transformers manually, similar to how
we'd do it without using refined types.

However, I did find a positive aspect — if we transform a refined type to a regular type, it works without any
boilerplate in both `chimney` and `ducktape`. Here's an example using `chimney`:

[//]: # (@formatter:off)

```scala
case class User(username: String :| MinLength[3], age: Int :| Positive)
case class DatabaseUser(username: String, age: Int)

val refinedUser = User("John", 21)
val databaseUser = refinedUser.transformInto[DatabaseUser] // or .to[...] for ducktape
```

[//]: # (@formatter:on)

This can be quite handy when you have a strict model (e.g., for the domain) and a more permissive one (e.g., for the
database).

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

[//]: # (@formatter:on)

and related exceptions:

[//]: # (@formatter:off)

```scala
class UsernameTooShortException(username: String) extends RuntimeException(s"Username '$username' is too short")
class NegativeAgeException(age: Int) extends RuntimeException(s"Age '$age' is negative")
type UserValidationException = UsernameTooShortException | NegativeAgeException
```

[//]: # (@formatter:on)

### Smart constructors with exceptions

This is the simplest and most straightforward approach: throw an exception when validation fails. In fact, this is the
method I used in the examples above. It's easy to understand and doesn't require any additional libraries. Here's how it
would look for the `User` class:

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

- Exceptions are side effects, so we need to remember to handle them
- The constructor fails on the first validation error, meaning we won't get all errors at once

Let's explore some other alternatives.

### Smart constructors with Option/Either

This is a more functional approach. Instead of throwing exceptions, we return an `Option` or `Either` type. Here's how
it would look for the `User` class using `Option`:

[//]: # (@formatter:off)

```scala
def smartConstructorWithOption(username: String, age: Int): Option[User] =
  Option.unless(username.length < 3 || age < 0) {
    User(username, age)
  }
```

[//]: # (@formatter:on)

While this is more functional, we lose valuable information about what went wrong. We can only tell if the validation
passed or failed.

> [Here](https://medium.com/virtuslab/option-the-null-of-our-times-77f3bfd7c234) is an interesting article discussing
> how `Option` can be perceived as the `null` of our times.
{: .prompt-tip }

To retain error information, we can use `Either`:

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

With `Either`, we now have error details, but we still fail on the first validation error. Can we do better?

### Smart constructors with Validated

For me, the best solution is to use the `Validated` type from `cats` (or an equivalent). This approach combines the
benefits of `Either` while also allowing us to accumulate errors. Here's how it looks for the `User` class:

[//]: # (@formatter:off)

```scala
def smartConstructorWithValidated(username: String, age: Int): ValidatedNel[UserValidationException, User] =
  (
    if (username.length < 3) (UsernameTooShortException(username): UserValidationException).invalidNel else username.validNel,
    if (age < 0) (NegativeAgeException(age): UserValidationException).invalidNel else age.validNel
  ).mapN(User.apply)
```

[//]: # (@formatter:on)

> Note: We have to help the compiler a bit by explicitly widening the exception type to `UserValidationException`,
> otherwise, `mapN` won't be able to concatenate the list of errors.
{: .prompt-info }

Of course, you can extract each validation into a separate method to make the code more readable or to be able to reuse
it. With this approach, we either get a valid `User` instance or a list of all the validation errors that occurred. In
my opinion, this is the best solution in most cases as it gives a complete picture of what went wrong. We can then
easily
handle each error individually or display them to the user.

You can find the code for the examples above [on Scastie](https://scastie.scala-lang.org/03eGrI5tRp2F9VwNzbUH3A).

## Summary

And that's a wrap! I hope that, even if you don't fully agree with my perspective, this post has sparked some thoughtful
consideration on the topic. I believe refined types are a unique tool, but I don't see many broad use cases for them in
production code. Scala's type system is powerful, and we should leverage it to our advantage. That doesn't necessarily
mean we need to rely on refined types. In my opinion, regular types paired with smart constructors can achieve the same
benefits at runtime while offering more flexibility and easier long-term maintenance.

Nowadays, we often raise the topic of how easy it is to onboard new developers to Scala. I think that refined
types are a great example of how we can make it harder. It's another tool to learn, and another library to maintain.
Smart constructors, on the other hand, are a well-established pattern, and most developers can grasp them with little
extra learning. While compilation errors are valuable, I believe that well-defined runtime validation logic is generally
more important — just be sure to test it thoroughly!

If you have any questions or comments, feel free to share them below. I'm eager to hear different opinions and learn
about your experiences. Thanks for reading!
