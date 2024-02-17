---
layout: single
title: "Fuzzer Development: Sandboxing Syscalls"
date: 2024-02-17
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
If you haven't heard, we're developing a fuzzer on the blog these days. I don't even know if "fuzzer" is the right word for what we're building, it's almost more like an execution engine that will expose hooks? Anyways, if you missed the first episode you can catch up [here](https://h0mbre.github.io/New_Fuzzer_Project/). We are creating a fuzzer that loads a statically built Bochs emulator into itself, and executes Bochs logic while maintaining a sandbox for Bochs. You can think of it as, we were too lazy to implement our own x86_64 emulator from scratch so we've just literally taken a complete emulator and stuffed it into our own process to use it. The fuzzer is written in Rust and Bochs is a C++ codebase. Bochs is a full system emulator, so the devices and everything else is just simulated in software. This is great for us because we can simply snapshot and restore Bochs itself to achieve snapshot fuzzing of our target. So the fuzzer runs Bochs and Bochs runs our target. This allows us to snapshot fuzz arbitrarily complex targets: web browsers, kernels, network stacks, etc. This episode, we'll delve into the concept of sandboxing Bochs from syscalls. We do not want Bochs to be capable of escaping its sandbox or retrieving any data from outside of our environment. So today we'll get into the implementation details of my first stab at Bochs-to-fuzzer context switching to handle syscalls. In the future we will also need to implement context switching from fuzzer-to-Bochs as well, but for now let's focus on syscalls. 

This fuzzer was conceived of and implemented originally by [Brandon Falk](https://twitter.com/gamozolabs).

## Syscalls
[Syscalls](https://wiki.osdev.org/System_Calls) are a way for userland to voluntarily context switch to kernel-mode in order to utilize some kernel provided utility or function. Context switching simply means changing the context in which code is executing. When you're adding integers, reading/writing memory, your process is executing in user-mode within your processes' virtual address space. But if you want to open a socket or file, you need the kernel's help. To do this, you make a syscall which will tell the processor to switch execution modes from user-mode to kernel-mode. In order to leave user-mode go to kernel-mode and then return to user-mode, a lot of care must be taken to accurately save the execution state at every step. Once you try to execute a syscall, the first thing the OS has to do is save your current execution state before it starts executing your requested kernel code, that way once the kernel is done with your request, it can return gracefully to executing your user-mode process.

Context-switching can be thought of as switching from executing one process to another. In our case, we're switching from Bochs execution to Lucid execution. Bochs is doing it's thing, reading/writing memory, doing arithmetic etc, but when it needs the kernel's help it attempts to make a syscall. When this occurs we need to:

1. recognize that Bochs is trying to syscall, this isn't always easy to do weirdly
2. intercept execution and redirect to the appropriate code path
3. save Bochs' execution state
4. execute our Lucid logic in place of the kernel, think of Lucid as Bochs' kernel
5. return gracefully to Bochs by restoring its state



