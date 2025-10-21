---
title: A few testing anti-patterns - part 1
date: 2025-10-20 12:00:00 +0000
categories: [ Blog ]
tags: [ clean code, testing, tips ]     # TAG names should always be lowercase
media_subpath: /assets/img/2025-10-20-a-few-testing-anti-patterns-part-1/
---

Let’s admit it — most of us don’t enjoy writing tests. Adding a new feature is often demanding enough that we rarely
have the energy left to write proper tests for it. We tend to treat them as an afterthought. This is especially true
when the feature is complex and the flow can take multiple paths depending on inputs, feature flags, or errors
encountered along the way.

Some might argue that embracing TDD would solve this problem. From my point of view, that’s an idyllic vision that falls
apart in practice. When we write code, we usually start with a general idea of the requirement in mind. We can describe
the high-level behavior we want to implement, but it’s difficult to immediately attach that to a specific class. In the
early stages of development, there are no interfaces or classes yet — we have to design them and usually refactor them
as new structures emerge. This makes TDD somewhat illusory.

> While I find TDD impractical for developing new features, it truly shines when fixing bugs. If you can reproduce a bug
> and isolate it in a unit test, that test serves as perfect verification that the fix actually works. It also makes
> code
> reviews much easier — you already have proof that your solution is valid.
> {: .prompt-tip }

[//]: # (TODO: Add an image)

How many times have you encountered a situation where you introduced a simple change and a whole host of tests fell
apart? If you have, that might actually be a good sign. The code you’re modifying has test coverage, which means you can
verify — at least to some extent — whether your changes broke the intended behavior. In this case, tests serve as a form
of documentation that also acts as a safety net when you’re not fully aware of every detail of the logic a piece of code
implements.
For that reason, failing tests shouldn’t generally annoy you. What might be worth complaining about is **why** they’re
failing. If the failures occur because you’ve changed the actual logic, that’s perfectly fine. However, if they’re
caused only by the test’s own implementation details, it’s a sign that the test wasn’t written properly.

> An example of such an implementation detail could be an assertion that relies on the order of elements in the result,
> even though the order doesn’t actually matter.
> {: .prompt-info }

That brings us to another aspect of testing: editing the tests. It’s not always easy to make them work again after even
a simple change is introduced. Sometimes the change is just a one-liner, yet it requires modifying dozens or even
hundreds of tests that rely on it. For this reason, we should aim to design our tests not only to verify the current
logic but also to make them easier to update in the future.

This post is the first in a series that covers some anti-patterns you might want to avoid in your codebase to make
working with tests more seamless for you and your teammates. If you’ve already adopted some of these concepts, it might
be difficult to move away from them. In that case, you can either avoid them from the start or gradually migrate your
tests as your codebase evolves.

## A few principles of my testing style

Before we dive into the main content of this post, let me briefly introduce my testing style. I realize that every
person or team has their own conventions, and I’d like to make sure we’re on the same page.

### given/when/then

First of all, I always try to structure my tests so that they follow the given/when/then pattern. That doesn’t
necessarily mean I add explicit comments to separate the sections, but I usually leave at least a blank line between
them. The simplest example would look like this:

[//]: # (@formatter:off)

```scala
val square = Square(width = 5)

val area = calculateArea(square)

assert(area === 25)
```

[//]: # (@formatter:on)

> The examples in this post are written in Scala, but the content itself is language-agnostic. The concepts discussed
> here are universal and can be applied in any language or framework, unless there’s a technical limitation that
> prevents it.
> {: .prompt-info }

Sometimes I also split the sections into smaller parts, especially when stubs are involved. For example:

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

Sometimes I move the stubs' behavior into the "when" section:

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

Placing the behavior in the “when” section also makes sense if you use mocks instead — the expectations are then
separated from the “given” section, which makes the tests easier to read. The downside of using mocks is that you
usually lose part (or sometimes all) of the “then” section, since the expectations are already expressed in the “when”.
I tend to decide which approach to use on a case-by-case basis.

> See [this section](https://martinfowler.com/articles/mocksArentStubs.html#TheDifferenceBetweenMocksAndStubs) of Martin
> Fowler's "Mocks Aren't Stubs" to learn more about the differences between stubs and mocks.
> {: .prompt-tip }

I follow the same convention in the test descriptions. Depending on the framework, my test might look like this:

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

In test descriptions, I tend to omit technical details if they’re irrelevant to the test itself. In the example above, I
didn’t mention that we need 3 services to perform this task and instead focused on the behavior.

I often see tests with descriptions like “happy path” or “should return a correct result.” In my opinion, such
descriptions provide no meaningful information about the behavior being tested. The description is the most flexible
part of a test — it lets us explain our intention in plain, human language. Make use of that! You can treat overly brief
descriptions as the first anti-pattern 😁

### Scope of the tests

The next thing I’d like to explain is the scope of tests. I usually try to limit the scope of each test suite to a
single class. There are two main reasons for this:

1. Smaller test suites are easier to reason about.
1. A 1:1 mapping between a class and its test suite is easier to maintain. I prefer having a single `FooTest` for `Foo`
   rather than a collection of suites combining multiple classes — ones I’ll eventually forget about or struggle to find
   because they’re spread across different packages.

I generally follow the same convention for both unit and integration tests.

> By "integration tests", I mean tests that interact with a single external component. For example, that could be a test
> of a repository that integrates with a database. See:
> [The Confusion About Testing Terminology](https://martinfowler.com/articles/practical-test-pyramid.html#TheConfusionAboutTestingTerminology)
> for more details.
> {: prompt-tip }

Because I usually use interfaces as class dependencies, I rarely need to include non-stubbed versions of other classes
in my test suites. This makes testing much easier, as I can focus on the behavior of the class under test while mocking
or stubbing the others. It also significantly reduces the scope of the tests.

### Mocks and stubs

This part might be controversial, but I frequently use both mocks and stubs. That’s not because they perfectly represent
behavior, but because they’re simply easy to use. Instead of implementing an entire interface with methods, argument
trackers, call counters, and so on, I can leverage a mocking framework to handle all of that for me. I won’t dive deeper
into it here. I just wanted to make you aware that, as you’ll see in
[Simulating third-party services](#simulating-third-party-services) below, in-memory adapters often serve as a
substitute for mocks and stubs.

### Parallelism

I have to admit that the codebases I usually work with commercially aren’t very parallelism-friendly. For unit tests,
teams usually manage well, but for integration tests or other layers of the testing pyramid, I often see sequential
execution. This is a significant setback for team velocity in a mature system. Lack of parallelism means we have to wait
longer for tests to run locally and for the CI pipeline to finish.

My principle is to aim for enabling parallel execution, unfortunately with varying results. Sometimes the codebase
simply isn’t flexible enough to allow it without a major refactor.

## Shared "given"

Let me start with an anti-pattern that actually motivated me to write this post. I've seen it far too many times, and it
has always been a hindrance. Shared "given" is a very simple thing - you just share the test data among tests within one
suite or among multiple suites. Here's an example of a shared "given" for a test suite that we'll use throughout this
section:

[//]: # (@formatter:off)

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

[//]: # (@formatter:on)

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

Why do people use it? At first, it just looks convenient. You don't have to repeat the same test data over and over
again. What is more, if your class has a lot of parameters, there's no need to define it in every test. However, over
time, the downsides start showing up.

### Problem 1: Describing "given"

The first issue appears quite quickly, when you have to describe the tests. If you share the test data, you cannot
easily describe what exactly each test is fed with. You may try to use something like "GIVEN a list of movie ratings",
but that's quite shallow description. Usually, we want to explain every test clearly, e.g. "GIVEN movie ratings from
multiple users" or "GIVEN multiple positive (>= 4) ratings for a single movie".

Of course, you can try to just assume that the shared data is so diversified that will fulfill the requirements of
every test, but that might quickly turn out to be either hard or impossible. We have to remember that the codebases
evolve constantly. If multiple tests depend on shared test data, you cannot really guarantee that even a small change
won't break their descriptions. Additionally, the tests may verify contradictory cases.

### Problem 2: Describing "then"

Another consequence of shared "given" is that it actually also affects "then". How can you define the assertions if
you cannot be sure what the input is? The workaround I often see is that the tests often rely on the fact that the
method under test filters the data. For example:

[//]: # (@formatter:off)

```scala
test(
  """GIVEN a list of movie ratings
    | WHEN average rating for one movie is calculated
    | THEN correct average rating should be returned
    |""".stripMargin
) {
  val movieId = "111"

  val averageRating = movieRatingService.averageRatingForMovie(movieId)

  assert(averageRating === 4.5)
}
```

[//]: # (@formatter:on)

There are numerous issues with the above test. Let's highlight two of them:

1. As described above, the "then" is shallow, because we cannot assume anything about the input. Thus, we end up with
   an assertion on "correct average rating", which is unclear. We always want to return correct results, don't we?
2. To understand where the final `4.5` comes from, we have to jump to the shared test data. What is more, we have to
   analyze it thoroughly to find all the ratings for movie with ID `111`, as they are not defined on the test level.

### Problem 3: Creating new test cases

The thing that will eventually drive your team up the wall is the ability to create new test cases.

Let's say that your team is developing a new feature, which will mark some ratings as originating from
[review bombing](https://en.wikipedia.org/wiki/Review_bomb). For simplicity, let's assume that you just had to add
a simple boolean flag `isReviewBomb` to the `MovieRating` model, which is `false` by default. Of course, to properly
verify that business logic depending on it, you have to add new test cases, which also means new test data. Because
a shared "given" is used, you may either edit existing instances, or add new ones to the list. Now, this is the moment
when the frustration begins. Imagine the following scenario:

1. You add a new `MovieRating` instance to the list and write a new test, and you run the whole test suite.

   [//]: # (@formatter:off)
    
    ```scala
    private val testMovieRatings = List(
      // other instances
      MovieRating(userId = "3", movieId = "222", rating = 1, isReviewBomb = true) // <- new instance
    )
    ```

    [//]: # (@formatter:on)

1. It turns out that the new instance you've added affects the average rating in another test, making it fail.

   [//]: # (@formatter:off)

    ```scala
    val averageRating = movieRatingService.averageRatingForMovie("222")

    assert(averageRating === 1)
    ```

   [//]: # (@formatter:on)
1. You fix the issue by altering the properties of the `MovieRating` you've added.

   [//]: # (@formatter:off)

    ```scala
    MovieRating(..., movieId = "not-used-anywhere-else", ...) // <- new instance
    ```

   [//]: # (@formatter:on)
1. Now, your test fails because the properties were altered, you have to adjust it.
1. It turns out that another test relied on the total number of movie ratings. You have to either remove one of the test
   instances, or change the definition of the failing test.

   [//]: # (@formatter:off)
    
    ```scala
    // ...
    val totalRatings = movieRatingService.countAll()
    
    assert(totalRatings === 15)
    // ...
    ```
    
    [//]: # (@formatter:on)

1. And so on...

It happened to me far too many times. Because the tests are interdependent on the shared test data, it makes them very
brittle. It's very frustrating when a simple change is introduced, but the test suites are written in a way that you
cannot really alter the shared test data without breaking other test suites.

### What to use instead?

For me, the most important rule is not to [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) the "given". Even
when you repeat the same instances over and over again, they serve their purpose - the tests become independent, and
contain the only data they need to verify the described behavior. They are also much easier to read as a form of
documentation, and equally easy to modify. It's just that simple.

But what if the instances you need for testing have numerous parameters, and you need a few instances per test? Your
"given" sections may quickly become gigantic blocks with a lot of irrelevant noise. Luckily, we can easily address that.
All we have to do is to just create small factory methods in the test suite that will initialize parameters to
reasonable defaults. For example, a factory method for our `MovieRating` could look like this:

[//]: # (@formatter:off)

```scala
private def anyMovieRating(
  userId: String = anyUserId,
  movieId: String = anyMovieId,
  rating: Int = 4,
  isReviewBomb: Boolean = false
) =
  MovieRating(userId, movieId, rating, isReviewBomb)

private lazy val anyUserId = "111"
private lazy val anyMovieId = "111"
```

[//]: # (@formatter:on)

As you see, I also introduced zero-args methods for the IDs. This is especially convenient when the values have to
adhere to a concrete pattern. I tend to name those `any...` to highlight that their properties may contain anything.
Now, you can use the above method to construct instances for the test purposes.

> I recommend avoiding random values in those methods. Non-deterministic values like `UUID.randomUUID()` or
> `Instant.now()` is often a source of confusion when it comes to debugging as the values constantly change. What is
> more, it might turn out that the logic in some edge cases actually depends on the exact values used, making the tests
> flaky.
> { :prompt-tip }

Let's fix the test which verifies average rating calculation:

[//]: # (@formatter:off)

```scala
test(
  """GIVEN ratings 4 and 5 for a movie
    | WHEN average rating for that movie is calculated
    | THEN the calculated average rating is 4.5
    |""".stripMargin
) {
  val movieId = anyMovieId
  val ratings = List(
    anyMovieRating(movieId = movieId, rating = 4),
    anyMovieRating(movieId = movieId, rating = 5)
  )
  val movieRatingService = new MovieRatingService
  movieRatingService.addAll(ratings)

  val averageRating = movieRatingService.averageRatingForMovie(movieId)

  assert(averageRating === 4.5)
}
```

[//]: # (@formatter:on)

The test is now not only accurately described, but also the whole "given" is represented in the code. There is no
implicit dependency on values defined elsewhere (especially, notice `movieId` was specified explicitly despite being
equal to the default value). We also conveniently skipped specifying the `userId`, as it's irrelevant in this case.

Sometimes, it might also make sense to create more domain-specific factory methods like `anyPositiveMovieRating` or
`anyReviewBombMovieRating`. As long as you are able to fully describe the properties of the created instance via the
method name and its parameters, you can be sure that your test will be understood by others. Try to create methods in
such a way that won't make the reader have to jump to their definition to understand your tests.

## Simulating third-party services

"Mock only yourself, because mocking others is rude" - this short maxim, although a bit silly, captures the essential
rule of thumb I want to convey here. The adapters in our code more often than not interact with the external world
through some kind of clients, JDBC drivers, messaging protocols, and so on. Most of the interfaces for those complicated
integrations are non-trivial, which makes people try to apply various shenanigans to be able to unit test the classes
using them. Some succeed by creating their own in-memory implementations, others eventually find open-source projects
that provide them. You can probably imagine how hard replicating the actual behavior of a constantly-changing project
such as Kafka or S3 is, yet people still do it
[[1](https://github.com/mguenther/kafka-junit)][[2](https://github.com/findify/s3mock)].

There is another very common subset of test classes that I'd like to also include here - in-memory databases. The most
common example that comes to mind is H2. Another is a plethora of in-memory databases written by hand by implementing
an interface using a built-in collection like a map. For example:

[//]: # (@formatter:off)

```scala
trait UserRepository {

  def readUser(userId: UserId): Option[User]

  def addUser(user: User): Unit

  // ...
}

class InMemoryUserRepository extends UserRepository {

  private val users = mutable.Map.empty[UserId, User]

  override def readUser(userId: UserId): Option[User] =
    users.get(userId)

  override def addUser(user: User): Unit =
    if (users.contains(user.id))
      throw new IllegalArgumentException("User already exists")
    else
      users.addOne(user.id -> user)

  // ...
}
```

[//]: # (@formatter:on)

This might be surprising for you, as many teams are using those. There's one fundamental issue with this approach -
in-memory implementation will never fully reflect the behavior of the real thing. Let's get into more details.

### Problem 1: Illusory test coverage

Let's start with an example service that will use the above repository:

[//]: # (@formatter:off)

```scala
class UserManagementService(userRepository: UserRepository) {

  private val logger = Logger[UserManagementService]

  def registerUser(user: User): Unit =
    Try {
      logger.info(s"Registering new user: ${user.id}")
      userRepository.addUser(user)
    } match {
      case Success(_) =>
        logger.info(s"User ${user.id} successfully registered!")
      case Failure(exception) =>
        logger.error(s"Registration of user ${user.id} failed", exception)
    }

  // ...
}
```

[//]: # (@formatter:on)

Then, a simple test for `registerUser` would look like this:

[//]: # (@formatter:off)

```scala
test(
  """GIVEN a user to register
    | WHEN a user with the same ID already exists
    | THEN the operation should be a no-op
    |""".stripMargin
) {
  // GIVEN
  val existingUser = User("existing-user-id")
  val duplicateUser = User("duplicate-user-id")

  val userRepository = new InMemoryUserRepository
  val userManagementService = new UserManagementService(userRepository)

  userManagementService.registerUser(existingUser)

  // WHEN
  userManagementService.registerUser(duplicateUser)

  // THEN
  assert(userRepository.readUser(existingUser.id).contains(existingUser))
}
```

[//]: # (@formatter:on)

Great! We were able to test the behavior, so where is the problem? The problem is that we're not guaranteed that the
repository we've defined behaves the same way as the "real" repository. Duplicate checks is a behavior defined outside
the test in `InMemoryUserRepository`, which is used only for testing. Effectively, this test verifies whether our
in-memory implementation works correctly, not that service logic is valid.

The same applies to third party libraries providing in-memory implementations. Let's say that we have a thin wrapper
around a Kafka client that sends `UserAdded` events to a Kafka topic:

[//]: # (@formatter:off)

```scala
class KafkaEventSender(
  kafkaClient: KafkaClient,
  eventsTopic: String
) {

  def sendEvent(userAdded: UserAdded): Unit =
    kafkaClient.send(eventsTopic, userAdded.serialize)
}
```

[//]: # (@formatter:on)

> For the purpose of the example, I'm simplifying a lot here. The real Kafka client interface is more complex.
> { :prompt-info }

Then, we use one of third-party libraries that provide in-memory Kafka server implementation to write a test for this
adapter:

[//]: # (@formatter:off)

```scala
test(
  """GIVEN a Kafka client and a UserAdded event
    | WHEN sendEvent is called with the event
    | THEN the event should be sent to the events topic
    |""".stripMargin
) {
  val kafkaClient = new ThirdPartyInMemoryKafkaClient()
  val eventsTopic = "events"
  val kafkaEventSender = new KafkaEventSender(kafkaClient, eventsTopic)
  val userAdded = UserAdded(User("user-id"))

  kafkaEventSender.sendEvent(userAdded)

  assert(kafkaClient.getMessages[UserAdded](eventsTopic) == List(userAdded))
}
```

[//]: # (@formatter:on)

You might have already guessed the issue here. We again verify the behavior of an in-memory implementation, not the
behavior of the adapter. This case is even worse than the previous one because the in-memory implementation is outside
our codebase, and probably quite complex. I hope you now understand what I mean by "illusory coverage" - we added some
tests, but they are effectively almost meaningless for the production code, as the actual behavior of the adapters might
be different.

There are also numerous other aspects worth considering here. Do in-memory implementations provide the same concurrency
guarantees? Are the operations atomic? Are we able to add transactions to an in-memory database? Of course, I'm not
saying that our test environment should always fully replicate the production. However, our goal should be to be
prepared to handle common behaviors of a live system, which might be omitted if we focus on replicating the logic
in-memory.

The last remark I have is that some of the properties of this anti-pattern may be already familiar to you. In-memory
implementation of an adapter is a single class, which is shared among the tests that we have to adjust to fit every
behavior under test. Does it ring a bell? Yup, it's another example of a shared "given", although not on the data level,
but on the logic level.

### Problem 2: You need integration tests anyway

As mentioned in the previous section, no matter how hard you try, a simulation of a third-party service will never
replace a real service. This means that to be certain that your implementation is correct, you have to write an
integration test anyway.

A concrete evidence for that might be using the Postgres flavor of H2 database to simulate the real Postgres. You'll
quickly find out that a lot of features, even basic ones, are simply not supported. The first thing that comes to my
mind are triggers. If your SQL defines a trigger at some point, you cannot use H2 anymore for testing. The same
limitations pop up at some point during testing of other kinds of adapters, not only those for databases.

If you have an integration test that verifies the same logic as the one with an in-memory adapter, then the latter
becomes an irrelevant duplication. There's no reason to maintain both if the former one integrates with the real service
and verifies the behavior against it.

### What to use instead?

There are two separate areas that I'd recommend to approach from different angles - adapters and services.

Let's start with testing the adapters, like `KafkaEventSender` described above or an actual implementation of
`UserRepository` that integrates with a real database. No matter whether those will be just a thin wrappers around a
third-party client, or actual logic like SQL queries, you have to test them against a real third-party service to be
sure that it works as expected. So, my overall recommendation is to spin up some Docker containers and write integration
tests for those.

Of course, it's not always that easy. Sometimes the containers are not available, especially when we're integrating with
some dedicated cloud service like S3. Amazon doesn't open-source S3, thus there's no Docker container for it. However,
S3 is a good example, as we have several approaches to tackle that:

1. Instead of using Docker, we can just create a dev environment in AWS and run our tests against it. This might incur
   costs, but you can be sure that your code is able to talk to the real thing.
2. We can use an S3-compatible Docker container that allows us to talk to it via the actual S3 client. An example of
   such a project is [Minio](https://www.min.io). You have to keep in mind that although you can use the actual S3
   client here, the behaviour of the underlying services might differ from the real S3.

There are also projects like [S3Mock](https://github.com/adobe/S3Mock) whose aim is to mimic the behavior of S3.
However, these have similar disadvantages as using H2 for simulating databases - at some point, you might discover that
not every feature is implemented, and you cannot use it for testing anymore.

The second area are the services. Here, my recommendation is quite simple - use stubs or mocks that reflect the behavior
you expect from the adapters, including the happy paths and errors. This way you will test the behavior of you service
rather than the adapters. Let's recall a test that was using `InMemoryUserRepository`:

[//]: # (@formatter:off)

```scala
test(
  """GIVEN a user to register
    | WHEN a user with the same ID already exists
    | THEN the operation should be a no-op
    |""".stripMargin
) {
  // GIVEN
  val existingUser = User("existing-user-id")
  val duplicateUser = User("duplicate-user-id")

  val userRepository = new InMemoryUserRepository
  val userManagementService = new UserManagementService(userRepository)

  userManagementService.registerUser(existingUser)

  // WHEN
  userManagementService.registerUser(duplicateUser)

  // THEN
  assert(userRepository.readUser(existingUser.id).contains(existingUser))
}
```

[//]: # (@formatter:on)

How can we improve it? Here, we want to test that our service recovers when the repository throws an exception. This can
be achieved with a simple stub:

[//]: # (@formatter:off)

```scala
test(
  """GIVEN a user to register
    | WHEN a DuplicateUserError is thrown by the repository
    | THEN the service should recover and no exception should be propagated
    |""".stripMargin
) {
  // GIVEN
  val user = User("user-id")

  val userRepository = stub[UserRepository]
  val userManagementService = new UserManagementService(userRepository)

  // WHEN
  (userRepository.addUser _).returnsWith(throw new DuplicateUserError(user.id))
  val result = userManagementService.registerUser(user)

  // THEN
  assert(result == ())
}
```

[//]: # (@formatter:on)

This way, the only class under test is our `UserManagementService`. We don't depend on the implementation of any
in-memory repository with its own logic.

## Summary

This was originally supposed to be a one-off post, but I see that it's already got quite long. That's why I've decided
to split it into more parts. I hope that you learned something useful. I also fully understand that the concepts
presented here might be quite controversial. Described anti-patterns are quite widely used and I saw them in all
commercial codebases I worked with. In each case, they were a hindrance rather than an aid. Although it's not easy to
get rid of those, maybe you'll find it easy not to start using them at all.

In the following post(s), I'll cover:

- Shared "then"
- Non-isolated shared resources
- Confusing math with science (yes, that's a testing anti-pattern!)

or maybe even more if I come up with other ideas. Stay tuned!
