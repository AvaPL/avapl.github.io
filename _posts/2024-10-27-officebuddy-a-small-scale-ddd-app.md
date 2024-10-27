---
title: "OfficeBuddy — a small-scale DDD app"
date: 2024-10-27 12:00:00 +0000
categories: [ Blog ]
tags: [ scala, functional programming, cats, ddd, domain-driven design, projects ]     # TAG names should always be lowercase
img_path: /assets/img/2024-10-27-officebuddy-a-small-scale-ddd-app/
---

Today, I published [OfficeBuddy](https://github.com/AvaPL/OfficeBuddy) — my pet project which I've been working on for
the past year in my spare time. It's a simple office management application, which I created to expand my knowledge
about some new areas and reinforce the knowledge of the ones I already know.

![Fluffy monsters office](fluffy_monsters_office.jpg){: w="400"}

Things you'll find in this project:

- small-scale DDD approach
- purely functional backend written in Scala with [Cats Effect](https://typelevel.org/cats-effect/)
- tests written using [Weaver](https://disneystreaming.github.io/weaver-test/)
  and [Mockito Scala](https://github.com/mockito/mockito-scala)
- mobile-first frontend written in Angular using mostly GitHub Copilot (it turns out it's quite good!)
- ...and probably more to come in the future if I continue the development 😄

If you want to check how a small-scale DDD approach can be implemented in Scala (~23K lines of code at the moment of
writing this post), I encourage you to take a look at the repository. I enjoyed every minute of working on this project,
and I hope you'll find it interesting as well!
