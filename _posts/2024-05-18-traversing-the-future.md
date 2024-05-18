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
