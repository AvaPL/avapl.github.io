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
- save each part in a dedicated repository which writes to Postgres tables
under the hood

At first, it seemed quite easy. Then we've found out that we need to add SQL
transactions to the repositories to avoid concurrency issues. That turned out
to be a pretty complex task. Not because we needed to add transactions,
but because we wanted to have code that is easy to maintain and doesn't depend
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
_Table schema_

> `user` table is not that relevant here, let's assume that user table is
defined somewhere and contains all fields that a user entity may require.
{: .prompt-info }

The components that we want to have are:
- abstract repositories for **account** and **points**
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

## It works, doesn't it?

Let's start with the definition of our repositories. The account repository looks
like this:

```scala
trait AccountRepository {
  def getBalance(userId: UUID): IO[Option[Int]]
  def addFunds(userId: UUID, amountToAdd: Int): IO[Unit]
}
```

I've used `IO` from [Cats Effect](https://typelevel.org/cats-effect/) as an effect.
The points repository is quite similar:

```scala
trait PointsRepository {
  def getPoints(userId: UUID): IO[Option[Int]]
  def incrementPoints(userId: UUID): IO[Unit]
}
```

The next thing is a service that exposes a method for adding funds which both
increases the balance and increments the loyalty points.

```scala
class AccountManagementService(
  accountRepository: AccountRepository,
  pointsRepository: PointsRepository
) {
  def addFunds(userId: UUID, amountToAdd: Int): IO[Unit] =
    for {
      _ <- accountRepository.addFunds(userId, amountToAdd)
      _ <- pointsRepository.incrementPoints(userId)
    } yield ()
}
```

Last missing piece is the implementation of our repositories. I'll use a
[PostgreSQL](https://www.postgresql.org/) database and queries written in
[doobie](https://tpolecat.github.io/doobie/). The first Postgres repository
is the one for accounts:

```scala
class PostgresAccountRepository(xa: Transactor[IO]) extends AccountRepository {

  override def getBalance(userId: UUID): IO[Option[Int]] =
    // (...).transact(xa)

  override def addFunds(userId: UUID, amountToAdd: Int): IO[Unit] =
    for {
      currentBalance <- getBalance(userId)
      newBalance = currentBalance.getOrElse(0) + amountToAdd
      _ <- setBalance(userId, newBalance)
    } yield ()

  private def setBalance(userId: UUID, balance: Int): IO[Int] =
    // (...).transact(xa)
}
```

Here I've used a `Transactor[IO]` from doobie that allows to convert a
[`ConnectionIO`](https://tpolecat.github.io/doobie/docs/03-Connecting.html)
produced by doobie SQL queries to an `IO`. The repository for points is
analogous:

```scala
class PostgresPointsRepository(xa: Transactor[IO]) extends PointsRepository {

  override def getPoints(userId: UUID): IO[Option[Int]] =
    // (...).transact(xa)

  override def incrementPoints(userId: UUID): IO[Unit] =
    for {
      currentPoints <- getPoints(userId)
      newPoints = currentPoints.getOrElse(0) + 1
      _ <- setPoints(userId, newPoints)
    } yield ()

  private def setPoints(userId: UUID, points: Int): IO[Int] =
    // (...).transact(xa)
}
```

Now as we have all elements needed we can construct our service:

```scala
val xa = Transactor.fromDriverManager[IO](
  driver = "org.postgresql.Driver",
  url = "jdbc:postgresql://localhost:5432/postgres",
  user = "postgres",
  pass = "example"
)
val accountManagementService = AccountManagementService(
  accountRepository = PostgresAccountRepository(xa),
  pointsRepository = PostgresPointsRepository(xa)
)
```

The service can be used to add some funds to a user account. This can be written
as simple as:

```scala
val userId = // uuid
IO
  .parReplicateAN(
    n = 4 // set parallelism degree to 4
  )(
    1000, // repeat 1000 times
    accountManagementService.addFunds(userId, 10)
  )
```

The code above adds 10 to the user balance 1000 times with the parallelism set
to 4. Assuming that we are starting with 0 funds and 0 points we should end up
with 10 000 account balance and 1000 points after this operation. There is just
one issue - we don't. Running the above `IO` results in some random value put
in the table rows. For instance, here is one of the result that I've received:

| user_id                              | balance  | points |
|:-------------------------------------|:---------|:-------|
| 50e91efe-ef47-407b-a1be-5a89e105b0af | 4760     | 443    |

These are definitely not the values that we expected.

### What happened?
