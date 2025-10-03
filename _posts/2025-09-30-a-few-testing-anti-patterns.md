---
title: A few testing anti-patterns
date: 2025-09-30 12:00:00 +0000
categories: [ Blog ]
tags: [ clean code, testing, tips ]     # TAG names should always be lowercase
media_subpath: /assets/img/2025-09-30-a-few-testing-anti-patterns/
---

Let's admit that - most of us don't like writing tests. Adding a new feature is often demanding enough that we don't
have enough energy to write proper tests for it. We usually treat them as an afterthought. This is especially true when
the feature is complex and the flow goes in various directions depending on the inputs, feature flags, or errors
encountered along the way.

Some of you might say that we should embrace TDD and the problem will be gone. From my point of view, that's an idyllic
vision that falls apart in practice. When we write code, we usually have a general idea of the requirement in mind. We
can describe the high level behavior we want to implement, but it's rather hard to attach it to a class right away. At
the early stages of development, we don't have any interfaces or classes yet - we have to design them and usually
refactor them along the way as the new structures emerge. This makes TDD illusory.

> While I find TDD impractical for developing new features, it shines for bugfixing. If you are able to reproduce a
> bug and isolate it in a unit test, it acts as a perfect verification whether the fix actually works. It also makes the
> code review much easier - you already have a proof that your solution is valid.
> {: .prompt-tip }

[//]: # (TODO: Add an image)

How many times have you encountered a situation when you introduced
a simple change and a whole host of tests fell apart? First of all, if you were in that situation, that might be a good
sign. The code that you modify has test coverage, which means you can validate (to some extent) whether your changes
broke the actual desired behavior. Tests in this case are a form of documentation that also acts as a safety net in case
you're not fully aware of every aspect of the logic a piece of code implements. For this reason, the fact that the tests
start failing shouldn't annoy you in general. What might be a reason for complaining is the reason **why** the tests are
failing. If they do so because you've changed the actual logic, it's all good. However, if that's something only related
to the implementation details of the test, then it's a sign that the test is not written properly.

> An example of such an implementation detail might be an assertion that relies on the order of the elements in the
> result when actually the order doesn't matter.
> {: .prompt-info }

That leads us to another aspect of testing, which is editing the tests. It's not always that easy to make the tests work
after even a simple change is introduced. Sometimes, the change is just a one-liner that requires you to modify tens
or hundreds of tests that relied on it. For this reason, we should strive to implement our tests not only to verify the
current logic, but also to facilitate editing them in the future.

This post covers a few anti-patterns that you might want to avoid in your codebase to make working with tests seamless
for you and your teammates. If you are already committed to some of the concepts described below, it might be quite
hard to get away from them. For this reason, you may either aim to avoid them from the beginning, or incrementally
migrate the tests as your codebase progresses.

## A few principles of my testing style

Before we begin with the main content of this post, let me introduce you to my testing style. I assume that every
person or team has their own rules and I'd like to ensure that we're on the same page.

### given/when/then

First of all, I always try to structure my tests so that they follow given/when/then structure. That doesn't always mean
that I introduce explicit comments that separate the sections from each other, but I tend to at least leave a whitespace
between the sections. The simplest example would be something like that:

[//]: # (@formatter:off)

```scala
val square = Square(width = 5)

val area = calculateArea(square)

assert(area === 25)
```

[//]: # (@formatter:on)

> The examples will be in Scala, but the whole post is language-agnostic. The presented concepts are universal and can
> be applied in any language or framework, unless there's a technical limitation that prevents it.
{: .prompt-info }

Sometimes I split the sections into subsections, especially when there are stubs involved. For example:

[//]: # (@formatter:off)

```scala
// GIVEN an email client and push notification service
val emailClient = stub[EmailClient]
val pushNotificationService = stub[PushNotificationService]
(emailClient.send _).returns(_ => ())
(pushNotificationService.send _).returns(_ => ())

// AND a notification service
val notificationService = new NotificationService(emailClient, pushNotificationService)

// AND user ID, notification title, and notification message
val userId = UUID.fromString("00000000-0000-0000-0000-000000000001")
val title = "Test title"
val message = "Test message"

// WHEN a notification is sent
notificationService.send(userId, title, message)

// THEN an email and a push notification are sent
assert((emailClient.send _).times === 1)
assert((pushNotificationService.send _).times === 1)
```

[//]: # (@formatter:on)

Sometimes I move the stubs behavior into the "when" section:

[//]: # (@formatter:off)

```scala
...

// WHEN the email client and push notification service can send notifications
(emailClient.send _).returns(_ => ())
(pushNotificationService.send _).returns(_ => ())

// AND a notification is sent
notificationService.send(userId, title, message)

...
```

[//]: # (@formatter:on)

Putting the behavior in "when" also makes sense if you use mocks instead - the expectations are then separated from the
"given" section, which also makes the tests easier to read. The disadvantage of using mocks is that you usually lose a
part of (or the whole) "then" section, as the expectations are already expressed in the "when". I tend to decide what to
use case by case.

> See [this section](https://martinfowler.com/articles/mocksArentStubs.html#TheDifferenceBetweenMocksAndStubs) of Martin
> Fowler's "Mocks Aren't Stubs" to get more information about the differences between stubs and mocks.
{: .prompt-tip }

I stick to the same convention in the description of the tests. Depending on the framework, my test may look like this:

[//]: # (@formatter:off)

```scala
test(
  """GIVEN a notification
    | WHEN the notification is sent
    | THEN an email and a push notification are sent
    |""".stripMargin
) {
  ...
}
```

[//]: # (@formatter:on)

In the description, I tend to omit the technical details if they are irrelevant for the test. In the example description
above, I didn't include the fact that we need 3 services to fulfill this task and focused on the behavior.

I quite often see tests that have descriptions like "happy path" or "should return a correct result". In my opinion,
they lack any information about the tested behavior. The description is the most flexible place of our tests, which
allows us to explain our intention the best way we can, using human language. Make use of it! You can treat laconic
descriptions as the first anti-pattern 😁

### Scope of the tests

The next thing that I'd like to explain is the scope of the tests. I usually try to reduce the scope of my tests to
a single class. There are two main reasons for it:

1. Smaller test suites are easier to reason about.
1. 1:1 mapping between a class and a test suite is easier to maintain. I prefer to have a single `FooTest` for `Foo`
   instead of a whole host of suites combining multiple classes, that I eventually forget about or struggle to find as
   they are in different packages.

I usually keep the same convention for unit tests and integration tests.

> By "integration tests", I mean a test that interacts with one external component. For instance, that may be a test
> of a repository integrating with a database. See:
> [The Confusion About Testing Terminology](https://martinfowler.com/articles/practical-test-pyramid.html#TheConfusionAboutTestingTerminology).
{: prompt-tip }

Because I usually stick to using interfaces as class dependencies, I hardly ever have to include non-stubbed versions
of other classes into my suites. This makes testing a lot easier, as I can test the behavior of the class under test,
and mock/stub the other ones. This significantly reduces the scope of the tests.

### Mocks and stubs

This part might be controversial, but I really frequently use both mocks and stubs. That's not because they are a 
perfect means of representing the behavior, but because they are just easy to use. Instead of having to implement
a whole interface of methods, values tracking the arguments they are called with, counters of method calls etc.,
I can just leverage a mocking framework that will set it up for me. I won't dive deeper into it, I just wanted to make
you aware of it as [In-memory adapters](#in-memory-adapters) I describe below is often a substitute for mocks/stubs.

### Parallelism

I have to admit that the codebases I usually work with commercially aren't parallelism-friendly. When it comes to unit
tests, teams usually succeed. When it comes to integration tests, or other layers of the testing pyramid, I usually see
sequential execution. That's a huge setback to the team velocity in a mature system. Lack of parallelism means that we 
have to wait longer for the tests to run locally and for the CI pipeline to finish. So, my principle is that I try to 
aim for enabling parallel execution, unfortunately with varying results. Sometimes the codebase is just not flexible
enough to achieve it without a major refactor. But worry not! Elimination of 
[Non-isolated shared resources](#non-isolated-shared-resources) described below will help you address that.

## Shared "given"

Let me start with an anti-pattern that actually motivated me to write this post. I've seen it far too many times, and it
has always been a hindrance. Shared "given" is a very simple thing - you just share the test data among tests within one
suite or among multiple suites. Here's an example of a shared "given" for a test suite that we'll use throughout this
section:

```scala
private val testMovieRatings = List(
  MovieRating(userId = "1", movieId = "111", rating = 5),
  MovieRating(userId = "2", movieId = "111", rating = 4),
  MovieRating(userId = "2", movieId = "222", rating = 2)
  // tens (or hundreds) of other instances ...
)

private val movieRatingService = new MovieRatingService

override protected def beforeAll(): Unit = {
  movieRatingService.addAll(testMovieRatings)
  super.beforeAll()
}
```

Shared "given" above has the following characteristics:
1. Shared test data in `testMovieRatings`.
1. `movieRatingService` that is a shared instance of the class under test.
1. `beforeAll` method that initializes the shared instance with the shared test data.

Of course, this anti-pattern often varies. The above structure is only one of many that I saw. Other common elements
I often see are:
- Separate classes only to store the shared test data
- Utility methods calls like `setupTestData()`, in either `beforeAll`, `beforeEach`, or on the test level
- Utility mixin classes, which do the setup for you, for example `class MyTest extends WithTestData`

Probably there's more to it, but you for sure get the idea - we share the test data between individual tests or whole 
test suites.

### Problem 1: Describing "given"

### Problem 2: Describing "then"

### Problem 3: Creating new test cases

### What to use instead?

[//]: # (TODO: Mention DRY)

## Shared "then"

## In-memory adapters

## Non-isolated shared resources

## Confusing math with science
