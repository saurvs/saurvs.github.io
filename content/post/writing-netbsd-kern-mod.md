+++
date = "2016-11-03T21:27:11+05:30"
title = "Writing a NetBSD kernel module"
draft = false

+++

Kernel modules are object files used to dynamically extend the functionality of
the operating system kernel. This means you (the superuser) can load and unload modules at *run time*,
making recompiling the entier kernel unnecessary. Some usecases of writing kernel modules
include adding a new system call, filesystem, character device, etc.


In NetBSD 5.0, the
kernel module subsystem was replaced with an updated subsystem that fixed the issues that plaqued the older one. The only tutorial I could find on writing NetBSD
modules made referrence to the older subsystem, so I thought I’d write one based on the newer subsystem.

So let's write a kernel module!

First off, what will our module do? It will add a new character device `/dev/prandom` that generates very large prime random numbers.
Not only that, it will be writen in C *and* Lua. Lua is an embeddable scripting language that
can be run from within the NetBSD kernel, which we will be using to simplify our code for
testing the primiality of large numbers.

Lastly, you can find more examples of kernel modules right in the NetBSD source tree.
