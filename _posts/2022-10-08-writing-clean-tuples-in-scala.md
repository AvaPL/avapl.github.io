---
title: Writing clean tuples in Scala
date: 2022-10-08 12:00:00 +0000
categories: [Blog]
tags: [scala, clean code, tips]     # TAG names should always be lowercase
media_subpath: /assets/img/2022-10-08-writing-clean-tuples-in-scala/
---

How many times have you wondered what is hidden under `._1` in a line of code?
How many `filter` conditions were not clear because they relied on tuple
accessors? Using tuples is quite easy, although it creates many
opportunities to write something nasty. This post contains tips that will
help you improve the readability of your tuples.

![Three fluffy monsters](three_fluffy_monsters.png){: w="400"}

## It's all about naming

Let's start with something simple. You have a list of integers and want to
divide them into a group of the ones that are smaller than 0 and ones
that are greater or equal to 0. The simplest solution would be something
like this:

```scala
val ints = List(-4, 1, -2, 0, 5)
val partitionedInts = ints.partition(_ < 0)
```

We are using
[`List#partition`](https://www.scala-lang.org/api/current/scala/collection/immutable/List.html#partition(p:A=%3EBoolean):(List[A],List[A]))
here, which returns a tuple with elements that satisfy the given condition,
and with those that don't. But how can we tell which group we will get
by using `partitionedInts._1`? We have to look into the documentation or the method definition. It is
not clear just by looking at the code. Sometimes it's not even obvious
that a particular line returns a tuple, and even if it does, we may not
know the arity of the tuple.

A cleaner way to write this is to name each part of the result:

```scala
val (lessThan0, greaterOrEqual0) = ints.partition(_ < 0)
```

This way we see at first glance what we get in the tuple, and we
know its arity. We don't have to look at the method definition at all!
What is more, we don't have to deal with the tuple accessors (`_1` and
`_2`) in the following lines.

## Ignoring parts of the result

Sometimes, we may want to ignore a part of the result. Let's say that
we call some external library that returns an HTTP response that is
represented as a tuple of a status code, headers, and the response entity.
Then, we want to decode the response entity to JSON.

```scala
val httpResponse = sendHttpRequest(httpRequest)
val json = decode(httpResponse._3)
```

How do we know that `httpResponse._3` is the encoded JSON body that we
care about? We may guess that only by `json` used as the variable
name. We can do it better:

```scala
val (_, _, responseEntity) = sendHttpRequest(httpRequest)
val json = decode(responseEntity)
```

Now it is clear what we are decoding. The code is also not cluttered
with the information that the method returns additional information.
We use `_` placeholders here that let us ignore the status code
and the headers that we are not using.

### When the tuple is bigger

When the arity of the tuple is high, we may not want to write a long
list of underscores to ignore the majority of elements. In such case,
we can use a tuple accessor and assign the value to a new variable
that has a meaningful name:

```scala
val httpResponse = sendHttpRequest(httpRequest)
val responseEntity = httpResponse._3
val json = decode(responseEntity)
```

## Meaningful method names and type aliases

Deconstruction of the result is only one side where the readability
can be improved. The other one is of course the method itself. Let's
look at three levels of improving the methods' readability.

### The starting point

```scala
def readContactInfo(userId: UUID): (String, String) =
  ("john@doe.com", "123 456 789")
```

We have a method that returns contact info
(that most likely is fetched from a database or an API) in a form of
a tuple of an email and a phone number. To check the meaning of the
returned `(String, String)` tuple, we have to look into the method's
body. How can that be improved?

### A meaningful method name

```scala
def readEmailAndPhoneNumber(userId: UUID): (String, String) =
  ("john@doe.com", "123 456 789")
```

Of course, the easiest way to improve the method is to use a better
name. We now have a hint on the result type.

### Type aliases

```scala
type Email = String
type PhoneNumber = String

def readEmailAndPhoneNumber(userId: UUID): (Email, PhoneNumber) =
  ("john@doe.com", "123 456 789")
```

To make it even cleaner, we can introduce type aliases. Here, the
returned values are obvious just by looking at the return type.
We've hidden that the returned values are `String`s though, so
decide wisely if it fits your design.

## Pattern matching and for comprehensions

Often, tuples are just an intermediate structure that we are
working on in a chain of operations. Again, naming to the rescue!

Let's say that we have a list of contact
information tuples that were given to us, and we cannot modify
them.

```scala
val contactInfos = userIds.map(readEmailAndPhoneNumber)
```

Then we want to get only the tuples where the phone number starts
with `48`. This can be achieved with a simple one-liner:

```scala
contactInfos.filter(_._2.startsWith("48"))
```

Not that clear, huh? You may already know what we are going to simplify there - the
filter condition as it contains a mystery `_2`. The easiest way
is to just use a `case` with variable names:

```scala
contactInfos.filter {
  case (email, phoneNumber) => phoneNumber.startsWith("48")
}
```

If our use case also requires some mapping, like dropping the
first 2 digits of the number, we can go further with either
`collect` or a `for` comprehension:

```scala
contactInfos.collect {
  case (email, phoneNumber) if phoneNumber.startsWith("48") =>
    (email, phoneNumber.drop(2))
}
```

```scala
for {
  (email, phoneNumber) <- contactInfos if phoneNumber.startsWith("48")
} yield (email, phoneNumber.drop(2))
```

This gives us more flexibility and cleaner code at the same time.

> In Scala 3, you can omit `case` in `filter` and `collect` lambdas.
{: .prompt-tip }

## Keep your tuples structure simple

Simplicity is as important as proper naming. Flat tuples are in
general easier to reason about than nested ones. For example,
to access the `2.5` element of:

```scala
val myTuple = (1, (2.5, "abc"))
```

you would have to use `myTuple._2._1`. Each nesting level
introduces additional complexity so try to keep your tuples simple.

## -> for Tuple2

Scala has a convenient way of defining `Tuple2` via the `->` operator.
This allows us to write quite clean mappings:

```scala
val colors = Map(
  "red"   -> 0xff0000,
  "green" -> 0x00ff00,
  "blue"  -> 0x0000ff
)
```

The thing is, this operator looks like a mapping by itself.
It's an arrow from one value to another one. When it is used
just to create a `Tuple2` that is not a mapping, it just looks
weird. For instance, what is `"john@doe.com" -> "123 456 789"`?
A mapping from email to a phone number? Or just a couple that
represents the contact information?

My advice is to use `->` only when the meaning of the tuple is
really a mapping. This will help to avoid confusion.

## Do you need a tuple at all?

As you see, writing clean code using tuples can be challenging.
The most straightforward way to avoid all kinds of readability
issues is to stay away from tuples. That is easier said than done
though. My rule of thumb is to use tuples only when a separation
could decrease performance. Tuples are especially useful when
the data is fetched in one go from a
database or an external service. This requires only one call
and is the most efficient.
[`List#partition`](https://www.scala-lang.org/api/current/scala/collection/immutable/List.html#partition(p:A=%3EBoolean):(List[A],List[A]))
mentioned at the beginning is also an example of such an optimization -
thanks to the use of a tuple, the operation requires only one
list traversal.

Also, use cases like
[context propagation](https://doc.akka.io/docs/akka/current/stream/stream-context.html)
in Akka Streams seem to be a natural fit for tuples. But this
particular one encouraged me to wonder if I need a tuple here
at all. Remember that some complex tuples can be easily replaced by a local
case class. If the elements form something coherent (like a context in
Akka Streams), a case class is the language structure to put
them in.

I hope that this post will make you more aware of the potential
consequences of using such a simple structure as a tuple.
