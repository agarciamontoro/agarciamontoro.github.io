---
layout: post
title: Google Summer of Code, First Impressions
lang: en
tags: [GSoC, ROOT, VecCore]
---

The first two weeks of coding are already gone, so this seems like a good moment to write a quick update of what I have been doing lately. SPOILER: I have been writing random code until something worked, as everyone of us developers do every day.

Before digging into the details of these two weeks work, and as my latest post was more a hello than a technical post, let me talk first about how to have a functional development environment for ROOT.

## Environment Set Up

Remember the goal of this GSoC project: to make ROOT ---its maths library--- faster. Ok, nice, but how do we even start with this? What do we need to do just to start writing code?

First of all, we have to build the latest version of ROOT. As with every libre software project, we have to locate its [repository](https://github.com/root-project/root), clone it to our machine and take a look at the [building instructions](https://root.cern.ch/building-root).

TL;DR: open a terminal, go to any directory and type the following:

{% highlight bash %}
git clone git@github.com:root-project/root.git
mkdir root-build
cd root-build
cmake -Dveccore=ON -Dbuiltin_veccore=ON \\
      -Dvc=ON -Dbuiltin_vc=ON \\
      -Dimt=ON -Droottest=ON -Dtesting=ON \\
      ../root-fork/
make
{% endhighlight %}

Ok, first three lines are quite self-explanatory, but the fourth one requires some comments. As you can see, ROOT is configured through the obscure but wonderful `CMake` tool, and the common syntax to prepare the build of a project like this one is something like `cmake <list_of_options> <source_path>`, where each of the options follow the syntax `-D<option_key>=<option_value>`.

Let's see what each option does:

* `-Dveccore=ON -Dbuiltin_veccore=ON`: These two options tell ROOT to be built with support of the `VecCore` library, and to use the version shipped within ROOT, not the one we could have already installed in our system.
* `-Dvc=ON -Dbuiltin_vc=ON`: As before, these ones specify that we want support for the `Vc` library shipped within ROOT.
* `-Dimt=ON`: Let ROOT enable its multithreading capabilities.
* `-Droottest=ON -Dtesting=ON`: Finally, tell ROOT to build all the tests for us, we will surely need them.

After the configuration is finished ---maybe you are prompted to install some required dependencies---, we can call `make` and... *Voilà*! Project built!

Now we should have a functional development environment in our machine, with support for vectorization and parallelization. But what are those `VecCore` and `Vc` libraries exactly? Why do we need them?

### Vectorization: Our New Best Friend

When we parallelize a code, we intend to run independent chunks of code at the **same** time, maybe in different CPUs, in different processors inside a GPU or even in completely different computers.

But what does *vectorization* means? *To vectorize* a code means to refactor its logic in order to compute all the operations over vectors ---or arrays of values---, as opposed to operate only with scalars. In computer jargon, is a way of implementing the [Single Instruction, Multiple Data (SIMD)](https://en.wikipedia.org/wiki/SIMD) paradigm: we apply **one operation** over **several values**.

The hardware architecture plays an important role in all this process, and it is important to understand it well in order to take advantage of its possibilities. Modern processors implement an extension to the common instruction set `x86`: it adds registers that do not contain single values but an array of them ---the number and size of the values are determined by the exact version of the extension, see this [Wikipedia article](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) for more information---, and instructions that operate over all the values in those special registers at once.

Obviously, we won't work directly with the processor instructions, and so we need a library that helps us abstracting the hardware layer and that gives us an interface to work with the vectors.

As you can imagine, `VecCore` and `Vc` are the tools we need:

* `VecCore` is the interface we will be using every day, as it provides an "architecture independent API for expressing vector operations on data". However, `VecCore` is just that, an interface, and the hard work is done by the chosen backend: in this case, a SIMD library that implements processor-level vectorization.
* `Vc` is the backend we will use to port our high-level code to the hardware layer. It will do the dirty work for us, and thanks to `VecCore`, if everything works fine, we won't even notice it exists. However, be thankful to `Vc`, our life will be way easier thanks to its existence <3
