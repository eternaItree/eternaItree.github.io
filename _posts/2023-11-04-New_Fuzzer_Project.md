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

## Introduction
For a long time I've wanted to develop a fuzzer on the blog, but for one reason or another, I could never really conceptualize a project that would be not only worthwhile as an educational tool, but also offer *some* utility to the fuzzing community in general. Recently, for Linux Kernel exploitation reasons, I've been very interested in Nyx. Nyx is a KVM-based hypervisor fuzzer that you can use to snapshot fuzz traditionally hard to fuzz targets. A lot of the time (most of the time?), we want to fuzz things that don't naturally lend themselves well to traditional fuzzing approaches. When faced with target complexity in fuzzing (leaving input generation aside for now), there have generally been two approaches.

One approach is to lobotomize the target such that you can isolate a small subset of the target that you find "interesting" and only fuzz that. That can look like a lot of things, such as ripping a small portion of a Kernel subsystem out of the kernel and compiling it into a userland application that can be fuzzed with traditional fuzzing tools. This could also look like taking an input parsing routine out of a Web Browser and fuzzing just the parsing logic. This approach has its limits though, in an ideal world, we want to fuzz anything that may come in contact with or be affected by the artifacts of this "interesting" target logic. This lobotomy approach is reducing the amount of target state we can explore to a large degree. Imagine if the hypothetical parsing routine successfully produces a data structure that is later consumed by separate target logic that actually reveals a bug. This fuzzing approach fails to explore that possibility.

Another approach, is to effectively sandbox your target in such a way that you can exert some control over its execution environment and fuzz the target in its entirety. This is the approach that fuzzers like Nyx take. By snapshot fuzzing an entire Virtual Machine, we are able to fuzz complex targets such as a Web Browser or Kernel in a way that we are able to explore much more state. Nyx provides us with a way to snapshot fuzz an entire Virtual Machine/system.  
