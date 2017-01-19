---
layout: post
title:  "Docker Learning Series Part 1: Containers and Cats"
description: "Containers are kind of like cats because they just don't care"
date:   2017-01-18 23:08:15
categories: LearningDocker
---

#### Disclaimer: This post references cats a lot. Prepare yourself.

The most obvious place to start here would be to define *what a container actually is*.

Because I like cats I am going to look at containers like cats. Cats are very independent creatures. Realistically, they only depend on their humans for the occasional expression of love, and that dependency isn't really one necessary to live. They can hunt food on their own, are just as happy outside as they are inside, and are generally indifferent to their human's presence. *This* is very much like a container. Containers look and smell like a very small VM without an OS. They are not dependent on their human for food (i.e they have their own resources like RAM, CPU, and network). They are also completely indifferent to _*where*_ they're hosted, much like a cat doesn't really care *who* is loving them occasionally. The only thing a container actually uses from it's human (or host) is the OS/Kernel. And that is because they are extremely lightweight and do not house their own.

But why is this relevant? This means something because since containers are so self-contained, it takes the guesswork out of ensuring applications run reliably across multiple environments (i.e Dev, Test, and Production). As I said before, all the container "uses" the host for is the OS/Kernel. It sits on top of the host. The cat sits on top of it's human for the occasional expression of love. See where I'm going here?

Now, all of this leads us to our next question...
#### *Why use a container and not a virtual machine?*

There are many reasons to use a container instead of a virtual machine. All of which are extremely dependent on specific use cases. While I can't give you advice on which to use because I'm still working out the basics -- <a href="https://blog.docker.com/2016/05/vm-or-containers/" target="_blank">this guy probably can</a>

I think I'm done comparing cats to containers. Regardless of how ridiculous it sounds, it was actually helpful for me to wrap my head around this while coming up with comparisons to cats.
