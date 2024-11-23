---
title: "Overriding PureConfig with environment variables"
date: 2024-11-24 12:00:00 +0000
categories: [ Blog ]
tags: [ scala, tips ]     # TAG names should always be lowercase
media_subpath: /assets/img/2024-11-24-overriding-pureconfig-with-environment-variables/
---

[//]: # (TODO: Review the post)

When I created this blog, one of my goals was to have a personal cookbook of the things that I tend to forget. This is
one of those things. I always forget how to override PureConfig settings with environment variables, so I decided to
finally write it down. Along the way of writing this post, I also discovered that there are some other ways than I
regularly use.

Please have in mind that this post is not an exhaustive list of all the options available. I also assume that you are
already at least partially familiar with [PureConfig](https://pureconfig.github.io)
and [Typesafe Config](https://github.com/lightbend/config).

[//]: # (TODO: Add image)
![Fluffy monsters configuring](fluffy_monsters_configuring.jpg){: w="400"}

## Configuration hierarchy

Most of the applications I worked with have the following hierarchy of configuration sources:

- `reference.conf` file - the config provided by libraries or shared modules
- `application.conf` file - service-specific configuration, with defaults for local development
- environment variables - overrides for deployments in contenerized environments

All the sources are managed by PureConfig. Whereas the first two are straightforward to manage, the last one always
confuses me. Sometimes there is a need to change the configuration of an already running application (especially on
production) and there's not much room for experimenting. There are actually multiple ways to do this. What is more, some
of them might not be available without modifying the application code. Let's see how we can do it.

## Example configuration setup

The following example is a simple configuration that we will use throughout this post:

[//]: # (@formatter:off)

```scala 3
case class FluffyMonster(
  name: String,
  pseudonyms: List[String],
  favoritePet: FluffyPet,
  otherPets: List[FluffyPet],
  dictionary: Map[String, String]
) derives ConfigReader

case class FluffyPet(
  name: String,
  color: String
) derives ConfigReader
```

[//]: # (@formatter:on)

[//]: # (TODO: Add image)
![Fluffy monster model](fluffy_monster_model.jpg){: w="400" .w-50 .right }

It contains the following types of configuration values:

- plain value in `name`
- list of values in `pseudonyms`
- nested configuration in `favoritePet`
- list of nested configurations in `otherPets`
- map in `dictionary`

Configuration classes derive `ConfigReader`s to allow PureConfig to read them.

A configuration file for this setup might look like this:

```hocon
{
  name = Jeremy
  pseudonyms = [Jerry, J]
  favorite-pet = {
    name = Misha
    color = gray
  }
  other-pets = [
    {
      name = Lolo
      color = blue
    },
    {
      name = Flippo
      color = pink
    }
  ]
  dictionary = {
    floo = sand
    floo-broo = sandwich
  }
}
```

Example code that loads the configuration is available here: https://scastie.scala-lang.org/Ox2A25BhSxKPQkArCZlSAw

> The example above loads the configuration from a string due to limitations in Scastie. In a real application, you
> would load it from `application.conf` file. Additionally, the example
> uses [PPrint](https://com-lihaoyi.github.io/PPrint/) for better visualization of the loaded configuration.
{: .prompt-info }

## Explicit environment variables in HOCON

The first way to override the configuration is to use environment variables directly in the configuration file.

The simplest case is overriding a plain value, which can be done like this:

```hocon
name = Jeremy
name = ${?MY_ENV_NAME}
```

If the environment variable `MY_ENV_NAME` is set, the value of `name` will be overridden. Otherwise, the value from the
configuration file will be used. What is interesting, it that we don't have to define a special `ConfigSource` to use
environment variables as overrides. They'll be automatically picked up by PureConfig.

Now, let's see how to override a list of values:

```hocon
pseudonyms = [Jerry, J]
pseudonyms = ${?MY_ENV_PSEUDONYMS}
```

The definition of the environment variable is analogous to the previous example. How should the values be provided
though? Here comes the first tricky part. Every value has to be provided as a separate environment variable with list
index after a dot:

```shell
MY_ENV_PSEUDONYMS.0 = "Joker"
MY_ENV_PSEUDONYMS.1 = "Hehehenry"
```

What is more, you can't just override a single value, you have to provide the whole list. Of course, you can provide
fewer or more elements in the list than in the original configuration.

> There is actually one additional issue - most of the shells don't allow defining environment variables with dots in
> the name. For instance, Bash allows
> only [letters, numbers, and underscores](https://www.gnu.org/savannah-checkouts/gnu/bash/manual/bash.html#index-name).
> There are various workarounds for this, each of them being shell-specific. The good news is that if you are defining
> the environment variables in a container, for example via Docker Compose or Kubernetes manifest, you won't encounter
> such constraints.
{: .prompt-info }

The next example shows how to override a nested configuration:

```hocon
favorite-pet = {
  name = Misha
  color = gray
}
favorite-pet = ${?MY_ENV_FAVORITE_PET}
```

Based on the previous example, you can probably guess that we'll need to provide the components after a dot:

```shell
MY_ENV_FAVORITE_PET.name = "Boo"
MY_ENV_FAVORITE_PET.color = "blue"
```

There is one important limitation here - you can't override only a single field of the nested configuration. You have to
provide all of them. It's not a big deal for a simple configuration like this one, but it might be a problem for more
complex ones. This applies to all approaches presented in this post.

We quite quickly got to the point where the environment variable contain not only the name we provided in the file, but
also dots and lower-case fields. It gets even further when we override a list of nested configurations:

```hocon
other-pets = [
  {
    name = Lolo
    color = blue
  },
  {
    name = Flippo
    color = pink
  }
]
other-pets = ${?MY_ENV_OTHER_PETS}
```

Overrides could look like this:

```shell
MY_ENV_OTHER_PETS.0.name = "Bean"
MY_ENV_OTHER_PETS.0.color = "green"
```

Quite a mess, isn't it? Let's now override a map:

```hocon
dictionary = {
  floo = sand
  floo-broo = sandwich
}
dictionary = ${?MY_ENV_DICTIONARY}
```

If we provide the following environment variables:

```shell
MY_ENV_DICTIONARY.floo = "sheep"
MY_ENV_DICTIONARY.wheee = "shark"
MY_ENV_DICTIONARY.HOO = "owl"
```

We might get surprised by the result map:

```scala
dictionary = Map(
  "floo" -> "sheep",
  "floo-broo" -> "sandwich",
  "wheee" -> "shark",
  "HOO" -> "owl"
)
```

Value for `floo` was overridden, we added entries for `wheee` and `HOO`, but `floo-broo` was left untouched. The
environment variables were prioritized but then eventually merged with the original configuration. The consequence is
that there is no way to remove a key from the default configuration map (or at least I didn't find it, please let me
know if I'm wrong!). Unfortunately, it applies to all approaches presented in this post.

Example code that loads the configuration with the above overrides is available
here: https://scastie.scala-lang.org/R7wmYgQJS2GXeVzxTZ0NCw

> Again, due to limitations of Scastie, I used additional workarounds. The application is run in a forked
> JVM (`fork := true`) with environment variables defined in `envVars` sbt setting. Navigate to "Build settings" on
> the left to see these parameters.
{: .prompt-info }

One of the advantages of this approach is that you can use custom names for the environment variables and put them
anywhere in the configuration file. For instance, we could define an environment variable that allows us to override
only the `name` of the second pet in `other-pets`:

```hocon
other-pets = [
  {
    name = Lolo
    color = blue
  },
  {
    name = Flippo
    name = ${?MY_ENV_FLIPPO_NAME}
    color = pink
  }
]
```

```shell
MY_ENV_FLIPPO_NAME = "Flippotron 9000"
```

The obvious downside is that you have to remember to provide all the necessary environment variables in the
configuration file and predict in advance which values might be overridden. Let's see whether we can improve that.

## Using CONFIG_FORCE_ environment variables

Usually, the less boilerplate we have to write, the better. For this reason, PureConfig (well, actually Typesafe Config
which lays beneath) provides a way to override the configuration with environment variables without explicitly defining
them in the configuration file. This is done by setting `CONFIG_FORCE_` environment variables. To use this feature, you
have to:

1. Use a `ConfigSource` that supports environment variable overrides (`ConfigSource.default` is enough).
2. Set `config.override_with_env_vars` JVM property to `true`.
3. Define environment variables prefixed with `CONFIG_FORCE_`.

The code that loads the configuration with can be as simple as:

```scala
val fluffyMonster = ConfigSource
  .default
  .loadOrThrow[FluffyMonster]
```

To override the `name` value, you can use the following environment variable:

```shell
CONFIG_FORCE_name = "Henry"
```

Will it follow the same pattern as before when overriding other types of values? Unfortunately, no. For example, to
override the elements of the list, we would use the following environment variables:

```shell
CONFIG_FORCE_pseudonyms_0 = "Joker"
CONFIG_FORCE_pseudonyms_0 = "Hehehenry"
```

To override the nested configuration, we would use:

```shell
CONFIG_FORCE_favorite__pet_name = "Boo"
CONFIG_FORCE_favorite__pet_color = "blue"
```

Why is it different? In this case, the configuration keys are mangled as follows:

- the prefix `CONFIG_FORCE_` is stripped
- single underscore (`_`) is converted into a dot (`.`)
- double underscore (`__`) is converted into a dash (`-`)
- triple underscore (`___`) is converted into a single underscore (`_`)

For example, the environment variable `CONFIG_FORCE_a_b__c___d` would override the configuration
key `a.b-c_d` ([source](https://github.com/lightbend/config/tree/v1.4.3?tab=readme-ov-file#optional-system-or-env-variable-overrides)).

The complete example code is available here: https://scastie.scala-lang.org/tUDnKZr1TRqqfv4FCoza8g

Now, we don't have to remember to define all the environment variables in the configuration file, which is great.
Additionally, the names are shell-friendly and we can easily define them. The disadvantage is that we don't have a 1:1
mapping between the configuration keys and the environment variables. It's also easy to forget how many underscores to
put in each case or just make a typo. Can we do better?

## Implicit environment variables

Well, it turns out we can. I didn't find any official part in the documentation about this, but it seems that we can
easily achieve 1:1 mapping between the configuration keys and the environment variables. To do this, we have to
use `ConfigFactory.systemEnvironment` from Typesafe Config and wrap it in `ConfigSource.fromConfig` as the primary
source of the configuration:

```scala
val fluffyMonster = ConfigSource
  .fromConfig {
    ConfigFactory.systemEnvironment().resolve()
  }
  .withFallback(ConfigSource.default)
  .loadOrThrow[FluffyMonster]
```

Now, we can override the configuration the same way we would do if we defined one-liners in the configuration file:

```shell
name = "Henry"
pseudonyms.0 = "Joker"
pseudonyms.1 = "Hehehenry"
favorite-pet.name = "Boo"
favorite-pet.color = "blue"
other-pets.0.name = "Bean"
other-pets.0.color = "green"
dictionary.floo = "sheep"
dictionary.wheee = "shark"
dictionary.HOO = "owl"
```

Awesome, isn't it? I highly recommend this approach as it's exceptionally easy to use. We don't have to remember about
explicitly defining environment variables in the configuration file, and we don't have to worry about any special naming
conventions.

The complete example code for this one is available here: https://scastie.scala-lang.org/wbMBfdJkR3Ca0pfO1Oqcgw

## Summary

As you see, there is no single canonical way to override configuration keys with in PureConfig. I hope that having this
post, I won't have to again go through trial and error to find out why my environment variables are not being picked up.
I also hope that you found this post useful and that you'll be able to use it in your projects! Remember to bookmark it,
you never know when you might need it 😄.
