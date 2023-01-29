---
title: Understanding Scala folds
date: 2023-01-30 12:00:00 +0000
categories: [Blog]
tags: [scala, functional programming]     # TAG names should always be lowercase
img_path: /assets/img/2023-01-30-understanding-scala-folds/
---

One of the first things that you encounter in the functional
programming world are functions that operate on collections.
In this post I will focus on the *fold* operation. It might
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
I would like to describe it in the simplest way possible. Fold is
an operation that takes an init element and a collection and then
aggregates (folds) it into a single value. This single value can
be anything - a sum, a concatenation or even a new collection.
Simple, isn't it?

![Fold definition example](fold_definition_example.png)

### What is associativity and commutativity?

When an operation is associative, it means that no matter how we put
parentheses in it, the result will be the same. For example, addition is
associative, whereas subtraction is not.

![Associativity example](associativity.png)

Commutativity property means that the order of operands does not matter
in the operation. We can observe it again on addition, which is commutative
and on subtraction, which is not.

![Commutativity example](commutativity.png)

Quite often these properties come together, but that's not always the case.
For instance, string concatenation is associative but not commutative.

![Associative but not commutative example](associative_not_commutative.png)

Why are we talking about this? Because both of these properties matter for
folds. Let's see how our results may be affected by that.
