---
layout: single
title: "Fuzzer Development: The Soul of a New Machine"
date: 2023-11-04
classes: wide
header:
  teaser: /assets/images/avatar.jpg
tags:
  - Fuzzing
  - Fuzzer
  - Development
  - Emulator
  - Bochs
---

## Introduction && Credit to Gamozolabs
For a long time I've wanted to develop a fuzzer on the blog, but for one reason or another, I could never really conceptualize a project that would be not only worthwhile as an educational tool, but also offer *some* utility to the fuzzing community in general. Recently, for Linux Kernel exploitation reasons, I've been very interested in [Nyx](https://nyx-fuzz.com/). Nyx is a KVM-based hypervisor fuzzer that you can use to snapshot fuzz traditionally hard to fuzz targets. A lot of the time (most of the time?), we want to fuzz things that don't naturally lend themselves well to traditional fuzzing approaches. When faced with target complexity in fuzzing (leaving input generation and nuance aside for now), there have generally been two approaches.

One approach is to lobotomize the target such that you can isolate a small subset of the target that you find "interesting" and only fuzz that. That can look like a lot of things, such as ripping a small portion of a Kernel subsystem out of the kernel and compiling it into a userland application that can be fuzzed with traditional fuzzing tools. This could also look like taking an input parsing routine out of a Web Browser and fuzzing just the parsing logic. This approach has its limits though, in an ideal world, we want to fuzz anything that may come in contact with or be affected by the artifacts of this "interesting" target logic. This lobotomy approach is reducing the amount of target state we can explore to a large degree. Imagine if the hypothetical parsing routine successfully produces a data structure that is later consumed by separate target logic that actually reveals a bug. This fuzzing approach fails to explore that possibility.

Another approach, is to effectively sandbox your target in such a way that you can exert some control over its execution environment and fuzz the target in its entirety. This is the approach that fuzzers like Nyx take. By snapshot fuzzing an entire Virtual Machine, we are able to fuzz complex targets such as a Web Browser or Kernel in a way that we are able to explore much more state. Nyx provides us with a way to snapshot fuzz an entire Virtual Machine/system. This is, in my opinion, the ideal way to fuzz things because you are drastically closing the gap between a contrived fuzzing environment and how the target applications exist in the "real-world". Now obviously there are tradeoffs here, one being the complexity of the fuzzing tooling itself. But, I think given the propensity of complex native code applications to harbor infinite bugs, the manual labor and complexity are worth it in order to increase the bug-finding potential of our fuzzing workflow.

And so, in my pursuit of understanding how Nyx works so that I could build a fuzzer ontop of it, I revisited [Gamozolabs'](https://twitter.com/gamozolabs) stream [paper review](https://www.youtube.com/watch?v=JpU-jrFnmfE) he did on the [Nyx paper](https://nyx-fuzz.com/papers/). It's a great stream, the Nyx authors were present in Twitch chat and so there were some good back and forths and the stream really highlights what an amazing utility Nyx is for fuzzing. But something *else* besides Nyx piqued my interest during the stream! During the stream, gamozo described a fuzzing architecture he had previously built that utilized the Bochs emulator to snapshot fuzz complex targets and entire systems. This architecture sounded extremely interesting and clever to me, and coincidentally it had several attributes in common with a sandboxing utility I had been designing with a friend for fuzzing as well. 

This fuzzing architecture seemed to meet several criteria that I personally value when it comes to doing a fuzzer development project on the blog:
- it is relatively simple in its design,
- it allows for almost endless introspection utilities to be added,
- it lends itself well to iterative development cycles,
- it can scale and be used on my servers I bought for fuzzing (but haven't used yet because I don't have a fuzzer!),
- it can fuzz the Linux Kernel,
- it can fuzz userland and kernel components on other OSes and platforms (Windows, MacOS),
- it is pretty unique in its design compared to open source fuzzing tools that exist,
- it can be designed from scratch to work well with existing flexible tooling such as LibAFL,
- there is no source code available anywhere publicly, so I'm free to implement it from scratch the way I see fit,
- it will allow me to do a lot of learning and low-level computing research and learning.

So all things considered, this seemed like the ideal project to implement on the blog and so I reached out to gamozo to make sure he'd be ok with it as I didn't want to be seen as clout chasing off of his ideas and he was very charitable and encouraged me to do it. So huge thanks to gamozo for sharing so much content and we're off to developing the fuzzer. 

Also huge shoutout to [@is_eqv](https://twitter.com/is_eqv) and [@ms_s3c](https://twitter.com/ms_s3c) at least two of the Nyx authors who are always super friendly and charitable with their time/answering questions. Some great people to have around.  

## Bochs
What is Bochs? Good question. [Bochs](https://bochs.sourceforge.io/) is an x86 full-system emulator capable of running an entire operating system with software-simulated hardware devices. In short, its a JIT-less, smaller, less-complex emulation tool similar to QEMU but with way less use-cases and way less performant. Instead of taking QEMU's approach of "let's emulate anything and everything and do it with good performance", Bochs has taken the approach of "let's emulate an entire x86 system 100% in software without worrying about performance for the most part. This approach has its obvious drawbacks, but if you are only interested in running x86 systems, Bochs is a great utility. We are going to use Bochs as the target execution engine in our fuzzer. Our target code will run inside Bochs. So if we are fuzzing the Linux Kernel for instance, that kernel will live and execute inside Bochs. Bochs is written in C++ and apparently still maintained, but do not expect much code changes or rapid development, the last release was over 2 years ago. 

## Fuzzer Architecture
