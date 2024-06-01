---
title: "Traversing the Future"
date: 2024-05-18 12:00:00 +0000
categories: [ Blog ]
tags: [ scala, functional programming, cats ]     # TAG names should always be lowercase
img_path: /assets/img/2024-05-18-traversing-the-future/
---

Although the community seems to be turning toward [direct style Scala](https://www.youtube.com/watch?v=0Fm0y4K4YO8)
nowadays, I have to admit that I kinda like the functional constructs. Higher-kinded types, monads, `IO`, etc., are
things that, in my opinion, make the code more explicit. Often, they highlight two important aspects of a robust
application: calls that are asynchronous and calls that might fail.

Of course, you have to find a balance between the functional gibberish and the real logic. As always in programming,
it's a kind of art. I've never worked commercially on a project that is written in a purely functional way. All of them
contained some Akka code or a Java library with side effects. The right word to describe these projects is "pragmatic" -
we used tools to get things done, not to blindly follow some set of paradigm rules.

I blend pure and impure code on daily basis, which presents a unique set of challenges. In this post, I'll delve into
one of those challenges.

![Fluffy monsters flying](fluffy_monsters_flying.jpg){: w="400"}

## The context

I might be wrong, but I think that the impure class I use most frequently is the `Future`. It's one of the basic
building blocks of Akka and a natural wrapper around Java `Future`s. Scala provides various ways to manipulate
`Future`s, including methods defined in the `Future` itself and for-comprehensions. However, when `Option`s and
`Either`s come into play, things get more complicated. Manipulating `Future[Either[DomainError, Result]]` is not as
straightforward as operating on `Future[Result]`.

For this reason, I frequently use [cats](https://typelevel.org/cats/) to simplify things. Although cats is designed to
work with purely functional code, it also has interop with `Future`s. This interop allows us, for instance, to:

- Use `EitherT[Future, DomainError, Result]` instead of `Future[Either[DomainError, Result]]`
- Use `.void` instead of `.map(_ => ())`
- Use `foo |+| bar` instead of `foo.flatMap(foo => bar.map(bar => foo + bar))`

Of course, there is a whole host of other use cases. This post will be about `.traverse` and also a bit
about `.sequence`.

> This post aims to explain one of the reasons why using `Future`s with cats is generally discouraged. While it often
> fits into "pragmatic" code, it might lead to unexpected behavior if used without extra caution.
{: .prompt-warning }

### Why is Future not pure?

Long story short, the problem with `Future` is that it's evaluated eagerly. The moment a `Future` is defined, it
immediately starts the computation. You might wonder if that's a problem at all - it turns out it might be. Consider the
following two snippets:

```scala
for {
  _ <- Future(println("hello"))
  _ <- Future(println("hello"))
} yield ()

// Output:
// hello
// hello
```

```scala
val hello = Future(println("hello"))

for {
  _ <- hello
  _ <- hello
} yield ()

// Output:
// hello
```

If `Future` was pure, the results would be the same. However, because of its eager evaluation, which breaks referential
transparency, the results are different. If we used `IO` here instead, we would receive the same output.

> Note that the problem isn't about using `println`, which has a side effect. Side effects, when correctly enclosed in
> an `IO` (or similar abstraction), are pure.
{: .prompt-info }

> If you have to use a `Future`, there is a way to wrap it with an `IO` to make it pure.
> See
> [IO#fromFuture](https://typelevel.org/cats-effect/api/3.x/cats/effect/IO$.html#fromFuture[A](fut:cats.effect.IO[scala.concurrent.Future[A]]):cats.effect.IO[A])
> for more details.
{: .prompt-tip }

### What is a functional traversal?

Now, quickly about `.traverse`. To simplify the explanation, we'll start with `.sequence` instead.

#### .sequence

Imagine that we have an element of type `A`. Let's first wrap it with a `G[_]`. Then, wrap the result in an `F[_]`.
Having `F[G[A]]`, we want to flip the wrappers, receiving `G[F[A]]`. That's exactly what `.sequence` does.

For instance, when our `F[_]` is a `List`, `G[_]` is an `Option`, and `A` is an `Int`:

```scala
import cats.implicits._

val foo = List(Some(1))
println(foo.sequence) // Some(List(1))
val bar = List(None)
println(bar.sequence) // None
```

![.sequence](sequence.png){: w="500"}
_.sequence applied to a List[Option[Int]]_

Note that this conversion is not always possible. Let's say we have a `Future` and a `List` of `Int`s. We can sequence
the operations one way:

```scala
List(Future(1), Future(2), Future(3)).sequence // yields a `Future[List[Int]]`
```

but not the other:

```scala
Future(List(1, 2, 3)).sequence // compilation error
```

This makes sense because if the second `.sequence` worked, we would be able, for instance, to tell the size of the list
before the computation inside the `Future` was finished.

#### .traverse

Traversal goes one step further than `.sequence` by additionally converting `A` to `B`. Imagine that instead of
having `F[G[A]]`, we have an `F[A]` and a function `A => G[B]`. After we apply that function to `A`, we `.sequence` the
result. That's what `.traverse` does.

Moreover, `.traverse` can be implemented via `.sequence`. `.traverse(f)` is the same thing as `.map(f).sequence`.

Let's see an example, analogous to the previous one:

```scala
import cats.implicits._

val foo = List("1")
println(foo.traverse(_.toIntOption)) // Some(List(1))
val bar = List("not an Int")
println(bar.traverse(_.toIntOption)) // None
```

![.traverse](traverse.png){: w="500"}
_.traverse applied to a List[String] using a String => Option[Int] function_

## The challenge

I surprisingly find myself quite often in situations where I have to execute some asynchronous operation on every
element of a collection. One example is legacy Java libraries that don't execute operations in batches; another might be
an external API that doesn't expose endpoints for batch edits.

Here is the pattern:

* I have a collection of elements like `List[Entity]`.
* I also have a function that executes an asynchronous operation using `Entity` (`Entity => Future[Result]`).
* I want to execute that operation on each `Entity` instance, getting `List[Future[Result]]` or something similar.
* Then, to simplify things, I want to convert it into a `Future[List[Result]]` or even a single `Future[BatchResult]` if
  I can combine `Result`s somehow.

As you see, this perfectly fits what `.traverse` does. Let's implement it then:

```scala
type Entity = String
type Result = String

val entities = List("humongous", "blue", "fluffy", "monster") // 1

def heavyComputation(entity: Entity): Future[Result] = // 2
  Future {
    println(s"Converting [$entity]...")
    val result = entity.toUpperCase
    Thread.sleep(100 * entity.size) // Pretend it's really heavy
    println(s"Converted [$entity] into [$result]")
    result
  }

def runCatsTraverse() = {
  println("\ncats .traverse(f)")
  val results = entities.traverse(heavyComputation) // 3
  println(s"results = ${Await.result(results, 15.seconds)}")
}
```

> Please forgive me for all the side effects and impurity of the code in the examples. I wanted to keep the examples as
> simple as possible to reduce the noise irrelevant to the actual problem.
{: .prompt-info }

In the snippet above, we have a list of entities (1) and a simulated heavy computation (2). Now, for we traverse each
element in the list, getting a list of results (3).

Speaking in terms of types, we have a `List[Entity]` (1) and a function `Entity => Future[Result]` (2) that we use for
traversal to ultimately receive `Future[List[Result]]` (3).

We are passing the list `[humongous, blue, fluffy, monster]` as the input and expect a parallel conversion to uppercase,
resulting in `[HUMONGOUS, BLUE, FLUFFY, MONSTER]` as the output.

Great! Let's run it. Here is an example output:

```
cats .traverse(f)
Converting [humongous]...
Converted [humongous] into [HUMONGOUS]
Converting [blue]...
Converted [blue] into [BLUE]
Converting [fluffy]...
Converted [fluffy] into [FLUFFY]
Converting [monster]...
Converted [monster] into [MONSTER]
results = List(HUMONGOUS, BLUE, FLUFFY, MONSTER)
```

It works well and the result is correct, but the order of operations is counterintuitive. Because we are using
`Future`s, we expect that we'll make use of the CPU and run the heavy computations in parallel. This didn't happen here.
Instead, each operation was run sequentially.

Before explaining why, let's take one more step. Remember this?
> Moreover, `.traverse` can be implemented via `.sequence`. `.traverse(f)` is the same thing as `.map(f).sequence`.

Let's use this property. Here's the modified code:

```scala
def runCatsMapSequence() = {
  println("\ncats .map(f).sequence")
  val results = entities.map(heavyComputation).sequence
  println(s"results = ${Await.result(results, 15.seconds)}")
}
```

Running it should yield the same result, shouldn't it? Well, here's the kicker:

```
cats .map(f).sequence
Converting [humongous]...
Converting [blue]...
Converting [fluffy]...
Converting [monster]...
Converted [blue] into [BLUE]
Converted [fluffy] into [FLUFFY]
Converted [monster] into [MONSTER]
Converted [humongous] into [HUMONGOUS]
results = List(HUMONGOUS, BLUE, FLUFFY, MONSTER)
```

It turns out that this time the computations were run in parallel.

### Where's the catch?

There are 2 main contributors to the behavior we observe above:

1. `.traverse` doesn't guarantee the execution order.
2. The `Future` is evaluated eagerly.

Because of the first point, we cannot be sure when the `Future`s will be executed. They could be run in parallel, but
unfortunately for us, they were evaluated sequentially.

Due to the second point, we observe parallel execution when `.traverse` is replaced with `.map(f).sequence`.
As we see in the outputs, cats' `.traverse` isn't implemented as `.map(f).sequence` because the outputs of the snippets
above are different.

> In fact, the behavior of `.traverse` for `Future`s was
> once [changed in a minor cats version](https://github.com/typelevel/cats/issues/4176). After that, `Future` usages
> with cats [were deprecated](https://github.com/typelevel/cats/pull/4230) due to being error-prone, but there are no
> plans for removal.
{: .prompt-info }

Now, we learned the hard way why using `Future`s with Cats is generally discouraged. The result was unexpected and
nothing warned us about it (well, except for the documentation in the source code that we didn't read). What’s more, we
discovered it only because we went the extra mile to investigate it in depth. Since the outputs are the same, this issue
could have been running in our codebase for months or even years unnoticed (make sure to check your codebase!),
degrading the performance of our system.

The question is: _can we do better?_

[//]: # (TODO: Scastie: https://scastie.scala-lang.org/pnwqiSLsTxC1I4Xg2YuYvg)

### Standard library to the rescue

[//]: # (TODO: Add a note about Future.sequence too)

### Would IO help?

[//]: # (TODO: Add a note about the result ordering for both Future and IO)

[//]: # (TODO: Describe that we cannot use parTraverse nor parSequence on a Future)
