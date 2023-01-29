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

Well, nothing, the result will be the same. Okay, that's boring. We just went
from left to right and applied added both operands in both cases. Let's take
a look at another `f`, subtraction this time.

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
subtraction or string concatenation. There we have it, our first caveat of
folds!

### foldRight
