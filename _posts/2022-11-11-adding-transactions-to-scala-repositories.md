---
title: Adding transactions to Scala repositories
date: 2022-11-11 12:00:00 +0000
categories: [Blog]
tags: [scala, sql, domain-driven design, ddd, tips]     # TAG names should always be lowercase
img_path: /assets/img/2022-11-11-adding-transactions-to-scala-repositories/
---

Not that long ago, I had to review a piece of code which worked like this:
- receive a HTTP request
- split the body into 5 parts
- write each part to a dedicated repository which writes to Postgres tables
under the hood

At first, it seemed quite easy. Then we've found out that we need to add SQL
transactions to the repositories to avoid concurrency issues. That turned out
to be a pretty complex task. Not because we needed to add transactions,
but because we wanted to have code that is easy to maintain, doesn't depend
on the fact that we are using SQL tables beneath.

In this post, I want to  and how to deal with transactions across the repositories
and highlight what problems you may encounter. I'll present my approach to
the solution from a faulty design to a more robust solution step by step.

![Fluffy monsters with money](fluffy_monsters_with_money.png){: w="500"}

## Example context

Suppose we wanted to build a part of a bank account management system. Our users
can check their balance and add funds. Additionally, each time they deposit some
money they get a kind of loyalty points. Of course this is just a minimal
fragment of the functionalities we could provide in such a system.

Here is our table schema:

![Tables](tables.png){: w="500"}

`user` table is not that relevant here, let's assume that user table is
defined somewhere and contains all fields that a user entity may require.

The components that we want to have are:
- abstract repositories for accounts and points
- concrete implementations of our repositories
- a service that encloses the main logic of adding funds and awarding loyalty
points

### Main goals of the final solution

The requirements that we want our code to fulfill are:
- safe concurrent execution of operations
- the services determine the boundaries of the transactions they execute
- the services are easy to test with repository mocks
- repositories don't have a predefined effect (eg. `IO`, `Future`) in method
return types
- abstractions do not depend on the implementations, ie. repositories don't
  use types that are specific to the classes that implement them - this will
  become clear later
