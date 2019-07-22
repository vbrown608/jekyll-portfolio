---
title: Yehos
links:
  - <a href='https://github.com/domspad/yehos' target='_blank'>Code on GitHub</a>
thumb:
  filename: yehos
  alt: A still from the 20th Century Fox opening sequence, represented using colored ASCII characters.
description: A toy kernel in C and assembly for the Intel 386 processor.
weight: 90
---

Yehos is an operating system built from scratch by me and four friends at the Recurse Center. It is written in C and x86 assembly for the Intel 386 processor. We ran it on emulated hardware using qemu. Features include:

- A bootloader [described here](/blog/yehos-bootloader.html)
- Keyboard IO
- Video output
- Lazy loading of applications from ISO disk
- Virtual memory and demand paging
- Multiple concurrent processes

My focus was on virtual memory, demand paging, and concurrent processes. This PR describes the [implementation of cooperative multiprocessing](https://github.com/domspad/yehos/pull/20) including `fork` and `yield` syscalls.

Yehos supports two applications:

- A video player playing a text-encoded version of the film Star Wars (1977)
- An implementation and REPL for the Forth programming language, [implemented in x86 assembly](https://github.com/domspad/yehos/blob/master/apps/forth.asm)


