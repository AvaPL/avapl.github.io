---
title: LLM in Scala - Introduction
date: 2025-03-02 08:00:00 +0000
categories: [ Blog ]
tags: [ scala, scalapy, ai, llm in scala ]     # TAG names should always be lowercase
media_subpath: /assets/img/2025-03-02-llm-in-scala-introduction/
---

> The *LLM in Scala* series is designed to be viewed as Jupyter notebooks. The post you're reading is a non-interactive
> version of
> [this notebook](https://github.com/AvaPL/llm-in-scala-blog-series/blob/main/posts/1-introduction/1-introduction.ipynb).
> For the best experience, I highly recommend setting up your local environment using
> the [llm-in-scala-blog-series](https://github.com/AvaPL/llm-in-scala-blog-series) repository. It requires just a
> single Docker command and runs in a fully isolated environment!
{: .prompt-info }

Hi!

I'm not sure why, but when I'm writing in a Jupyter notebook, I feel more inclined to welcome you than when writing a
blog post. I'm really happy that you've decided to check out this series!

First, let me explain why we're in a Jupyter notebook. This isn't a native environment for a Scala program, is it? While
experimenting with various tools for implementing the code in [ScalaLLM](https://github.com/AvaPL/ScalaLLM), I wondered
what would be a straightforward way to present my code to others. In the Python world, people often use descriptive
Jupyter notebooks for this purpose. Without much hesitation, I checked what was available in our ecosystem and
discovered [Almond](https://almond.sh), which I hadn't known about before. After some trial and error, I finally decided
it would be a suitable tool for the job.

![Fluffy monster implementing neural network](fluffy_monster_neural_network.jpg){: w="350"}

## Scope of the series

You might be wondering what "LLM in Scala" really means. The idea for this project was sparked by the
book [Build a Large Language Model (From Scratch)](https://www.manning.com/books/build-a-large-language-model-from-scratch).
This book provides a comprehensive guide to implementing an LLM from the ground up using PyTorch. It also delves into
key concepts in detail, helping me see LLMs not as some abstract, otherworldly technology but as a natural extension of
deep neural networks.

The book’s code examples are written in Python. While there's nothing inherently wrong with Python, I wanted to take a
different approach, one that leverages Scala and its type safety to gain some help from the compiler. That’s how this
project came to life.

In this series, I’ll cover:

- Using Scala in Jupyter notebooks
- Interacting with Python and its libraries via ScalaPy
- The structure of basic LLMs, their components, and their roles in the model
- Implementing and training GPT-2 using the tools and knowledge gained along the way

This is likely to be a long journey, but for me, it was absolutely worth it. I learned a lot. I hope others will find
value in it as well, as I’ll be sharing insights that are often undocumented, lessons learned through trial and error,
and a few design patterns that emerged naturally throughout the process.

## How did we get here? - Jupyter, Almond, and Docker

Before diving in, let me first describe the setup that allows me to present this entire series in the form of Jupyter
notebooks. The key tool that made this possible is [Almond](https://almond.sh), a Jupyter kernel for Scala. While
Jupyter is often associated with Python, it actually supports many languages as long as there’s a dedicated kernel
available.

The downside is that not every widely used Jupyter-based service allows custom kernels. For example, you can run them
on [Binder](https://mybinder.org) (for free) and on [Deepnote](https://deepnote.com) (with paid plans that support
custom Docker images), but not on [Colab](https://colab.google). Although Google once allowed custom kernels, they’ve
since changed their approach, making it difficult, or in some cases, impossible, to run anything outside of their
predefined environment.

Given what happened with Colab, I wanted a setup that would remain reproducible for years to come, without relying on
external services that might change their policies. That’s why I chose [Docker](https://www.docker.com). While I could
have installed all the required tools directly on my local machine, Docker offers several advantages: it provides an
isolated environment that can be easily set up and torn down, and it allows precise control over resource allocation
which is crucial for neural networks, which tend to be resource-hungry.

With that in mind, I put together a simple Dockerfile and a small Docker Compose configuration. Let’s take a look at how
they work.

### Dockerfile

We start by selecting a base image. Since Jupyter is built on Python, we’ll use a lightweight Python slim image:

```dockerfile
ARG PYTHON_VERSION=3.12
FROM python:${PYTHON_VERSION}-slim
```  

Next, with Python available, we can install Jupyter. I opted for JupyterLab, which offers a more advanced interface:

```dockerfile
# Install JupyterLab
RUN pip install jupyterlab
```  

That covers the Python-related setup. Now, we need a Java runtime and the Scala Jupyter kernel. We’ll begin with a
headless JRE (which excludes GUI components) and add a few basic utilities that aren’t included in the base image. Here,
I’m using `curl` and `unzip`, but you can add more if needed:

```dockerfile
# Install Java Runtime and other utils
ARG JRE_VERSION=17
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      openjdk-${JRE_VERSION}-jre-headless \
      curl \
      unzip \
      && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```  

The last two lines ensure that unnecessary leftovers are removed, keeping the final image as small as possible.

Next, we install Almond, the Scala Jupyter kernel, using `coursier`:

```dockerfile
# Install Almond kernel with Scala
ARG ALMOND_VERSION=0.14.0-RC15
ARG SCALA_VERSION=2.13.14
RUN curl -Lo coursier https://git.io/coursier-cli && \
    chmod +x coursier && \
    ./coursier launch almond:${ALMOND_VERSION} --scala ${SCALA_VERSION} -- \
      --install --display-name "Scala ${SCALA_VERSION}" && \
    rm -f coursier
```  

Be sure to pick an Almond version that supports your chosen Scala version.

We’re almost done! To organize our notebooks and related files, we set up a dedicated workspace using `WORKDIR`:

```dockerfile
WORKDIR /posts  
```  

Finally, we define the command that runs JupyterLab on startup:

```dockerfile
# Start JupyterLab (note: allow-root is unsafe, don't use it in production - why would you anyway?)
EXPOSE 8888  
CMD ["jupyter", "lab", "--allow-root", "--ip=0.0.0.0"]  
```  

We expose the default JupyterLab port and bind it to `0.0.0.0` (
see [this explanation](https://stackoverflow.com/questions/20778771/what-is-the-difference-between-0-0-0-0-127-0-0-1-and-localhost)
for why).

> One caveat: The `--allow-root` flag grants JupyterLab root access within the container, allowing it to install Python
> libraries and manage dependencies without permission issues. While it’s possible to configure a separate user with
> appropriate privileges, doing so would require additional setup and significantly complicate the configuration. Since
> we're running inside Docker, the security risks are limited to the containerized environment. However, never use this
> flag on your local machine or a production server, as running JupyterLab as root outside a controlled container can
> expose your system to serious security risks.
{: .prompt-warning }

### Docker Compose

While we could manually build and run the Docker image, it's more convenient to have a small Compose file that defines
the build context and mounts a volume from the local filesystem. Here’s a minimal Docker Compose configuration that does
just that:

```yaml
services:
  llm-in-scala-blog-series:
    container_name: llm-in-scala-blog-series
    build:
      context: .
    ports:
      - 8888:8888
    volumes:
      - ./posts:/posts
```  

This setup creates a single container based on the Dockerfile in the current directory (`.`). It maps port `8888` inside
the container to port `8888` on the host, allowing access to JupyterLab. Additionally, it mounts the local `posts`
folder to the container’s working directory, ensuring that notebooks and related files persist across container
restarts.

Now, run the following command to build the image and start the container in the background:

```shell
docker compose up --build -d
```  

Once the container is running, you can find the JupyterLab access URL by checking the logs:

```shell
docker logs llm-in-scala-blog-series 2>&1 | grep '127.0.0.1'
```  

Since JupyterLab generates a new access token each time it starts, the URL will be different on every run. These tokens
are essential for security, preventing unauthorized execution of code on the server.

And that’s it! We now have a fully functional JupyterLab instance running in an isolated environment, ready to execute
Scala code:

```scala
import scala.util.Properties.versionString

println(s"Hello from Scala $versionString!")
```

> Hello from Scala version 2.13.14!

### Why not use an existing image?

You might be wondering why I didn’t use the [almondsh/almond](https://hub.docker.com/r/almondsh/almond) image or one of
the prebuilt images from [Jupyter’s DockerHub](https://hub.docker.com/u/jupyter) instead of creating a custom one. Well,
I actually tried but it didn’t go well.

Without getting into too many details, the main issue was interoperability with ScalaPy. The prebuilt images didn’t
properly expose the Python dynamic libraries needed for ScalaPy to function, leading to compatibility issues. Rather
than spending time debugging and patching those images, I opted for a custom setup that’s simpler, predictable, and
works out of the box.

## A simplified guide to Scala in Jupyter

With everything set up, what can we do with this environment?

Since Almond is built on top of [Ammonite](https://ammonite.io), you can use it just like the Ammonite REPL. As shown
earlier, you can write and execute Scala code as usual:

```scala
import java.time.LocalDate

case class Person(name: String, birthYear: Int)

val person = Person("Alice", 1990)
val age = LocalDate.now.getYear - person.birthYear

println(s"${person.name} is $age years old.")
```

> Alice is 35 years old.

Since we’re in a REPL environment, you can reuse previous values and even redefine them as needed:

```scala
println(s"Current person name: ${person.name}")
```

> Current person name: Alice

```scala
val person = Person("Bob", 1989)
println(s"New person name: ${person.name}")
```

> New person name: Bob

### User input... or not?

Unfortunately, reading user input doesn’t work for me with this configuration. While Almond can communicate with the
Jupyter API and display an input field, the process hangs after the input is provided. However, this might work for you,
so it’s worth trying.

But let's take one positive lesson from this: if anything hangs, you can press the ⏹ icon in the top bar to interrupt
the cell.

```scala
val name = Input("Enter your name: ").request()
println(s"Hello $name!")
```

><pre>
> Enter your name:  Pawel
> Interrupted!
>   jdk.internal.misc.Unsafe.park(Native Method)
>   ...
></pre>

### Using libraries

With all the power of Ammonite at our disposal, we can easily import libraries of our choice. To import a library, use
the following syntax:

```scala
import $ivy.`<groupId>::<artifactId>:<version>`
```

For example:

```scala
import $ivy.`com.lihaoyi::ujson:4.1.0`

val json = ujson.read(""" { "example": 123 } """)
println(s"Example value: ${json("example")}")
```

><pre>
> Downloading https://repo1.maven.org/maven2/com/lihaoyi/ujson_2.13/4.1.0/ujson_2.13-4.1.0.pom
> ...
> Example value: 123
></pre>

### Using common code

To extract common code, simply save it as a `.sc` file and import it into your notebook. For example, suppose we have a
file called `Commons.sc` with the following content:

```scala
case class Point(x: Int, y: Int)

def printPointSum(p: Point): Unit =
  println(s"Sum of coordinates: ${p.x + p.y}")
```

You can then use the defined method in your notebook like this:

```scala
import $file.Commons

val point = Commons.Point(1, 2)
Commons.printPointSum(point)
```

><pre>
> Compiling /posts/1-introduction/Commons.sc
> Sum of coordinates: 3
></pre>

The kernel runs relative to the folder where the notebook is placed. This means that if you have a `.sc` file in a
subfolder, you'll need to specify the folder name in the import. For example, to import `Other.sc` from a folder
called `inner`, you would use:

```scala
import $file.inner.Other
```

If the `.sc` file is in the parent folder, you can use the special `^` in the path. When `Other.sc` is placed in the
parent folder, use:

```scala
import $file.^.Other
```

Unfortunately, I haven’t found an easy way to import another Scala notebook. Please let me know if you discover one!

### File IO

Speaking of files, you can work with them as usual. To read and write files, simply use the regular IO utilities:

```scala
import scala.io.Source
import java.nio.file.{Files, Paths}
import java.nio.charset.StandardCharsets

val fileContents = Source.fromFile("lorem_ipsum.txt").mkString
val fileContentsUppercase = fileContents.toUpperCase
Files.write(Paths.get("lorem_ipsum_upper.txt"), fileContentsUppercase.getBytes(StandardCharsets.UTF_8))
```

### Interacting with the OS

Another powerful tool at our disposal is full access to [os-lib](https://github.com/com-lihaoyi/os-lib). For example,
you can call `curl` in a subprocess like this:

```scala
import os._

def runCommand(command: String*): Unit =
  print(os.proc(command).call(
    stdout = os.ProcessOutput.Readlines(println),
    stderr = os.ProcessOutput.Readlines(Console.err.println)
  ).out.text())

runCommand("curl", "example.com")
```

><pre>
>   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
>                                  Dload  Upload   Total   Spent    Left  Speed
> 
>   0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
>   0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
> 100  1256  100  1256    0     0   2100      0 --:--:-- --:--:-- --:--:--  2100
> <!doctype html>
> &lt;html>
> ...
></pre>

### Python interop

The final feature I want to highlight is interop with Python. We'll be using this extensively throughout the series, so
I won’t dive into all the details here. To use Python, simply import ScalaPy as a regular library:

```scala
import $ivy.`dev.scalapy::scalapy-core:0.5.3`

import me.shadaj.scalapy.py

val sys = py.module("sys")
println(s"You're using Python ${sys.version}.")
```

><pre>
> Downloading https://repo1.maven.org/maven2/dev/scalapy/scalapy-core_2.13/0.5.3/scalapy-core_2.13-0.5.3.pom
> ...
> You're using Python 3.12.9 (main, Feb 25 2025, 08:58:51) [GCC 12.2.0].
></pre>

## Summary

Okay, I think that's enough for an introduction post. In this post, we set up a Scala environment in Jupyter using
Almond, explored how to interact with Scala in a REPL setting, and looked into integrating various tools and libraries.
We covered importing and using libraries, accessing files, and interacting with the OS, while also touching on the power
of Python interop.

In future posts, we'll dive deeper into implementing a large language model in Scala, building upon the knowledge and
tools we've introduced here. Stay tuned!
