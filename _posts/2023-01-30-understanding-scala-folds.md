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

## What is a fold?
