---
title: "Overriding PureConfig with environment variables"
date: 2024-11-24 12:00:00 +0000
categories: [ Blog ]
tags: [ scala, tips ]     # TAG names should always be lowercase
media_subpath: /assets/img/2024-11-24-overriding-pureconfig-with-environment-variables/
---

When I started this blog, one of my main goals was to create a personal cookbook — a place to document the things I tend
to forget. This post is a perfect example. I always seem to forget how to override PureConfig settings with environment
variables, so I’ve finally decided to write it down. While drafting this, I also stumbled upon a few alternative
approaches that I hadn’t used before.

Keep in mind, this isn’t an exhaustive list of all available options. I’m assuming you’re already at least somewhat
familiar with [PureConfig](https://pureconfig.github.io) and [Typesafe Config](https://github.com/lightbend/config).

[//]: # (TODO: Add image)
![Fluffy monsters configuring](fluffy_monsters_configuring.jpg){: w="400" }

## Configuration hierarchy

In most of the applications I’ve worked on, the configuration sources follow this hierarchy:

- **`reference.conf` file** – base configuration provided by libraries or shared modules
- **`application.conf` file** – service-specific settings, often with defaults for local development
- **environment variables** – overrides used in containerized deployments.

All the sources are managed by PureConfig. While the first two are fairly straightforward, managing environment
variable overrides has always been a bit tricky for me. This is especially true when you need to tweak the configuration
of a running application in production, where there’s little room for experimentation.

Interestingly, there are several ways to handle environment variable overrides, but not all of them are accessible
without modifying the application code. Let’s explore the options.

## Example configuration setup

Here’s a simple configuration that we’ll use throughout this post:

[//]: # (@formatter:off)

```scala
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

![Fluffy monster model](fluffy_monster_model.jpg){: w="310" .w-30 .right }

This setup includes several types of configuration values:

- **plain value** in `name`
- **list of values** in `pseudonyms`
- **nested configuration** in `favoritePet`
- **list of nested configurations** in `otherPets`
- **map** in `dictionary`

The configuration classes derive `ConfigReaders`, enabling PureConfig to read them.

Here’s an example configuration file for this setup:

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

Example code that loads the configuration is available
here: [Scastie Example](https://scastie.scala-lang.org/Ox2A25BhSxKPQkArCZlSAw)

> The example above loads the configuration from a string due to limitations in Scastie. In a real application, you
> would load it from an `application.conf` file. Additionally, the example
> uses [PPrint](https://com-lihaoyi.github.io/PPrint/) for better visualization of the loaded configuration.  
{: .prompt-info }

## Explicit environment variables in HOCON

The first method to override configuration is by using environment variables directly in the configuration file.

### Overriding a plain value

The simplest case involves overriding a plain value:

```hocon
name = Jeremy
name = ${?MY_ENV_NAME}
```

If the environment variable `MY_ENV_NAME` is set, it overrides the value of `name`. Otherwise, the value specified in
the configuration file is used. An interesting aspect of this approach is that you don’t need to define a special
`ConfigSource` for environment variables — they are automatically picked up by PureConfig.

### Overriding a list of values

Next, let’s see how to override a list of values:

```hocon
pseudonyms = [Jerry, J]
pseudonyms = ${?MY_ENV_PSEUDONYMS}
```

The environment variable definition follows the same pattern as in the previous example. But how should the values be
provided? Here’s the tricky part: each value in the list must be defined as a separate environment variable, with the
list index appended after a dot:

```shell
MY_ENV_PSEUDONYMS.0 = "Joker"
MY_ENV_PSEUDONYMS.1 = "Hehehenry"
```

Additionally, you can't just override a single value in the list — you must provide the entire list. You can, however,
specify more or fewer elements than in the original configuration.

> There is actually one additional issue: most shells don't allow defining environment variables with dots in the name.
> For example, Bash only supports
> [letters, numbers, and underscores](https://www.gnu.org/savannah-checkouts/gnu/bash/manual/bash.html#index-name).
> There are various shell-specific workarounds for this limitation. The good news is that if you're defining environment
> variables in a container (e.g., via Docker Compose or Kubernetes manifests), you won't encounter such constraints.  
{: .prompt-info }

### Overriding a nested configuration

Next, let's look at how to override a nested configuration:

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

There is one important limitation to note: you cannot override just a single field in a nested configuration. You must
provide values for all fields. While this isn't an issue for simple configurations like this one, it can become
problematic with more complex ones. This limitation applies to all approaches covered in this post.

### Overriding a list of nested configurations

We quite quickly got to the point where the environment variable contains not only the name we provided in the file, but
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

Quite a mess, isn't it?

### Overriding values in a map

Let's now look at how to override some values in a map:

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

We might be surprised by the resulting map:

```scala
dictionary = Map(
  "floo" -> "sheep",
  "floo-broo" -> "sandwich",
  "wheee" -> "shark",
  "HOO" -> "owl"
)
```

The value for `floo` was overridden, and new entries were added for `wheee` and `HOO`, but `floo-broo` was left
untouched. The environment variables were prioritized but then eventually merged with the original configuration. The
consequence of this behavior is that there is no way to remove a key from the default configuration map (at least, I
haven’t found a way — please let me know if I’m wrong!). Unfortunately, this limitation applies to all the approaches
presented in this post.

Example code that loads the configuration with the above overrides is available
here: [Scastie example](https://scastie.scala-lang.org/R7wmYgQJS2GXeVzxTZ0NCw).

> Again, due to limitations in Scastie, I've used additional workarounds. The application runs in a forked
> JVM (`fork := true`), with environment variables defined in the `envVars` sbt setting. To view these parameters,
> navigate to "Build settings" on the left.
{: .prompt-info }

One of the advantages of this approach is that you can use custom names for environment variables and place them
anywhere in the configuration file. For example, we could define an environment variable to override only the `name` of
the second pet in the `other-pets` list:

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
configuration file, as well as predict in advance which values might be overridden. Let's explore if there's a way to
make this more manageable.

## Using CONFIG_FORCE_ environment variables

Generally, the less boilerplate we need to write, the better. For this reason, PureConfig (or more specifically,
Typesafe Config, which is used under the hood) offers a way to override configuration with environment variables without
explicitly defining them in the configuration file. This is done by using `CONFIG_FORCE_` environment variables. To make
use of this feature, you need to:

1. Use a `ConfigSource` that supports environment variable overrides (`ConfigSource.default` is enough).
2. Set the JVM property `config.override_with_env_vars` to `true`.
3. Define environment variables with the `CONFIG_FORCE_` prefix.

To override the `name` value, you would set the following environment variable:

```shell
CONFIG_FORCE_name = "Henry"
```

But will this follow the same pattern as when overriding other values? Unfortunately, no. For example, to override the
elements of a list, we use environment variables like this:

```shell
CONFIG_FORCE_pseudonyms_0 = "Joker"
CONFIG_FORCE_pseudonyms_0 = "Hehehenry"
```

To override a nested configuration, the environment variables would look like this:

```shell
CONFIG_FORCE_favorite__pet_name = "Boo"
CONFIG_FORCE_favorite__pet_color = "blue"
```

Why is it different? In this case, the configuration keys are transformed as follows:

- the prefix `CONFIG_FORCE_` is stripped
- single underscores (`_`) are converted into dots (`.`)
- double underscores (`__`) are converted into dashes (`-`)
- triple underscores (`___`) are converted into a single underscore (`_`)

For example, the environment variable `CONFIG_FORCE_a_b__c___d` would override the configuration
key `a.b-c_d` ([source](https://github.com/lightbend/config/tree/v1.4.3?tab=readme-ov-file#optional-system-or-env-variable-overrides)).

The complete example code is available here: [Scastie example](https://scastie.scala-lang.org/tUDnKZr1TRqqfv4FCoza8g)

With this approach, we no longer need to remember to define all the environment variables directly in the configuration
file, which is great. Additionally, the variable names are shell-friendly and easier to define. The downside is that we
don't have the 1:1 mapping between configuration keys and environment variables. It's also easy to forget how many
underscores to use or just make a typo. Can we improve this further?

## Implicit environment variables

Well, it turns out we can. While I couldn't find any official documentation on this, it appears that we can easily
achieve a 1:1 mapping between the configuration keys and the environment variables. To do this, we can
use `ConfigFactory.systemEnvironment` from Typesafe Config and wrap it in `ConfigSource.fromConfig` as the primary
source for the configuration:

```scala
val fluffyMonster = ConfigSource
  .fromConfig {
    ConfigFactory.systemEnvironment().resolve()
  }
  .withFallback(ConfigSource.default)
  .loadOrThrow[FluffyMonster]
```

Now, we can override the configuration just as we would with one-liners in the configuration file:

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

Awesome, right? I highly recommend this approach because it's exceptionally easy to use. There's no need to explicitly
define environment variables in the configuration file, and we also don't have to worry about special naming
conventions.

You can find the complete example code for this approach
here: [Scastie example](https://scastie.scala-lang.org/wbMBfdJkR3Ca0pfO1Oqcgw)

## Summary

As we've seen, there's no single, canonical way to override configuration keys in PureConfig. I hope this post helps you
avoid the trial and error of figuring out why your environment variables aren't being picked up. I also hope you found
this post useful and that it will come in handy for your projects! Don't forget to bookmark it — you never know when you
might need it 😄
