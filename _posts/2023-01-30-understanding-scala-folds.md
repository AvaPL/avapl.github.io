---
title: Understanding Scala folds
date: 2023-01-30 12:00:00 +0000
categories: [Blog]
tags: [scala, functional programming]     # TAG names should always be lowercase
img_path: /assets/img/2023-01-30-understanding-scala-folds/
---

One of the first things that you encounter in the functional
programming world are functions that operate on collections.
In this post I will focus on the **fold** operation. It might
sometimes be as confusing as it is common in our programs,
especially when used with non-associative or non-commutative
operators. Don't worry if you don't know what it means, more
on that below!

> This post was inspired by my deep dive into Haskell after reading
[Learn You a Haskell for Great Good!](http://learnyouahaskell.com/)
book. I highly recommend reading it!
{: .prompt-info }

![Fluffy monster on carpet](fluffy_monster_on_carpet.png){: w="350"}

## Some explanation first

### What is a fold?

Instead of a direct definition or quoting the documentation,
I would like to describe it in the simplest way possible. **Fold** is
an operation that takes an init element and a collection and then
aggregates (folds) it into a single value. This single value can
be anything - a sum, a concatenation or even a new collection.
Simple, isn't it?

![Fold definition example](fold_definition_example.png)

### What is associativity and commutativity?

When an operation is **associative**, it means that no matter how we put
parentheses in it, the result will be the same. For example, addition is
associative, whereas subtraction is not.

![Associativity example](associativity.png)

**Commutativity** property means that the order of operands does not matter
in the operation. We can observe it again on addition, which is commutative
and on subtraction, which is not.

![Commutativity example](commutativity.png)

Quite often these properties come together, but that's not always the case.
For instance, string concatenation is associative but not commutative.

![Associative but not commutative example](associative_not_commutative.png)

Why are we talking about this? Because both of these properties matter for
folds. You'll see how our results may be affected by that in the next sections.

## Types of folds

There are 3 methods that allow folding in Scala:
[`foldLeft`](https://scala-lang.org/api/3.x/scala/collection/IterableOnceOps.html#foldLeft-fffff9e1),
[`foldRight`](https://scala-lang.org/api/3.x/scala/collection/IterableOnceOps.html#foldRight-fffff9e1)
and
[`fold`](https://scala-lang.org/api/3.x/scala/collection/IterableOnceOps.html#fold-fffff9e1).
The first two have associativity (or if you prefer, direction) specified in
their names, and the third one is a fold that doesn't have a predefined order
of operations. `fold` defaults to `foldLeft` and can be overridden if the
collection supports a more efficient way of unordered folding. It won't be
covered in this post in detail.

### foldLeft

Let's take a look at the most common of the folds, `foldLeft`. According to
the documentation:

> Applies a binary operator to a start value and all elements of this collection, going left to right.

In my opinion, it's better to think about this operation as a fold that
*associates* to the left. Why? Because it doesn't really go from left to
right, let me explain.

Let's assume that we have a `List(4, 2, 7)`, an init value of `0` and a binary
operation `f`. In the code it can be described as:

```scala
List(4, 2, 7).foldLeft(0)(f)
```

`foldLeft` will construct the following expression:

![foldLeft](foldleft.png){: w="500"}

We start with initial value of `0`. Then `f` is applied on `0` and `4`.
Then, `f` is applied to the result of the previous operation and `2`. The same
thing happens for `7`. Did we go from left to right? Kind of, we went through
the list from left to right to construct a whole expression which then was
evaluated as the final result.

> The important thing to note is that the final result of `foldLeft` is `f`
of accumulated value and the last element of the list. I'll refer to that
fact later.
{: .prompt-tip }

#### Implementation

We can implement `foldLeft` for a `List[A]` using tail recursion:

```scala
@tailrec
final def foldLeft[B](z: B)(f: (B, A) => B): B =
  list match {
    case head :: tail => tail.foldLeft(f(z, head))(f)
    case Nil          => z
  }
```

This means that `foldLeft` is stack safe, and you won't see
`StackOverflowError`s. Note that the evaluation terminates on the very last
element so the whole list is always traversed.

#### Examples

Let's assume that `f` is addition:

```scala
List(4, 2, 7).foldLeft(0)((acc, int) => acc + int) // 13
```

Note that the accumulator in `foldLeft` is on the left. This means that we can
simplify the above to:

```scala
List(4, 2, 7).foldLeft(0)(_ + _) // 13
```

What would happen though if we defined it like this?

```scala
List(4, 2, 7).foldLeft(0)((acc, int) => int + acc) // 13
```

Well, nothing. The result will be the same.

Okay, that's boring. We just went from left to right and applied added both
operands in both cases. Let's take a look at another `f`, subtraction this
time.

```scala
List(4, 2, 7).foldLeft(0)(_ - _) // -13
```

We received the same number but with a minus, seems reasonable. Now, we'll
swap the operands again:

```scala
List(4, 2, 7).foldLeft(0)((acc, int) => int - acc) // 9
```

And we get `9`. `9`? Why `9`? Weren't we just applying subtraction from left
to right in this case too?

![foldLeft subtraction commutativity](foldleft_subtraction_commutativity.png){: w="500"}

Well, we were but the `f` in this case swapped the order of operands. This means
that `f` affects the result of operations that are non-commutative, like
subtraction or string concatenation. Even when we know that we cannot freely
swap the operands here, it's hard to predict the result without analyzing the
expression tree. There we have it, our first caveat of folds!

### foldRight

You might have guessed that `foldRight` associates the operations to the right.

Using the same `List(4, 2, 7)`, we'll jump straight into the expression formed
by:

```scala
List(4, 2, 7).foldRight(0)(f)
```

It gives us:

![foldRight](foldright.png){: w="500"}

Note that the initial `0` is now on the right side of the list. If you
understood how `foldLeft` works, it should be easy to understand `foldRight`.
But this is where the fun begins!

#### Implementation

The most straightforward implementation of `foldRight` for `List[A]` is via
recursion:

```scala
def foldRight[B](z: B)(f: (A, B) => B): B =
  list match {
    case head :: tail => f(head, tail.foldRight(z)(f))
    case Nil => z
  }
```

We have to begin the evaluation from the last element of the list, so we build
a chain of recursive calls until we reach the end. This has a major
disadvantage though. We'll sooner or later run out of stack frames because
of stack overflows.

We can definitely do better. If we have to start from the end of the list,
why don't we just put the last element at the beginning by reversing the list?
This is a thing that we can do in a stack safe way. How? Via `foldLeft`!

```scala
def reverse: List[A] =
  list.foldLeft(List.empty[A])((acc, a) => a :: acc)
```

Here, we are using fold to transform one collection into another one. So now,
how can we perform a `foldRight` on a reversed list? Again, via `foldLeft`.
Surprising, isn't it?

```scala
def foldRight[B](z: B)(f: (A, B) => B): B =
  list.reverse.foldLeft(z)((acc, a) => f(a, acc))
```

We have a stack safe implementation of `foldRight`, but it comes at a cost.
By using `foldLeft` twice, we are also traversing the list twice. This is
a thing to consider when we want to use `foldRight`.

> Scala standard library implements `foldRight` exactly this way.
{: .prompt-tip }

#### Examples

If you understand `foldLeft`, understanding the examples for `foldRight`
should be easy.

```scala
// Equivalent of:
//   List(4, 2, 7).foldLeft(0)(_ + _)
// as addition is associative and commutative.
List(4, 2, 7).foldRight(0)(_ + _) // 13
```

```scala
// Equivalent of:
//   List(4, 2, 7).foldLeft(0)(_ - _)
List(4, 2, 7).foldRight(0)((int, acc) => acc - int) // -13
```

```scala
// Equivalent of:
//   List(4, 2, 7).foldLeft(0)((acc, int) => int - acc)
List(4, 2, 7).foldRight(0)(_ - _) // 9
```

Note that by convention, the accumulator for `foldRight` is the right argument
of `f`. This is a thing that may help you or confuse you more, especially
when using wildcards.

The last interesting example of `foldRight` use case is appending to a list.
If we are starting the evaluation from the end, we can add a single element or
a whole list to the end effortlessly:

```scala
List(4, 2, 7).foldRight(List(0))(_ :: _) // List(4, 2, 7, 0)
```

#### To infinity, and beyond!

## Summary
