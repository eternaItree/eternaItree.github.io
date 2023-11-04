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
For a long time I've wanted to develop a fuzzer on the blog during my weekends and freetime, but for one reason or another, I could never really conceptualize a project that would be not only worthwhile as an educational tool, but also offer *some* utility to the fuzzing community in general. Recently, for Linux Kernel exploitation reasons, I've been very interested in [Nyx](https://nyx-fuzz.com/). Nyx is a KVM-based hypervisor fuzzer that you can use to snapshot fuzz traditionally hard to fuzz targets. A lot of the time (most of the time?), we want to fuzz things that don't naturally lend themselves well to traditional fuzzing approaches. When faced with target complexity in fuzzing (leaving input generation and nuance aside for now), there have generally been two approaches.

One approach is to lobotomize the target such that you can isolate a small subset of the target that you find "interesting" and only fuzz that. That can look like a lot of things, such as ripping a small portion of a Kernel subsystem out of the kernel and compiling it into a userland application that can be fuzzed with traditional fuzzing tools. This could also look like taking an input parsing routine out of a Web Browser and fuzzing just the parsing logic. This approach has its limits though, in an ideal world, we want to fuzz anything that may come in contact with or be affected by the artifacts of this "interesting" target logic. This lobotomy approach is reducing the amount of target state we can explore to a large degree. Imagine if the hypothetical parsing routine successfully produces a data structure that is later consumed by separate target logic that actually reveals a bug. This fuzzing approach fails to explore that possibility.

Another approach, is to effectively sandbox your target in such a way that you can exert some control over its execution environment and fuzz the target in its entirety. This is the approach that fuzzers like Nyx take. By snapshot fuzzing an entire Virtual Machine, we are able to fuzz complex targets such as a Web Browser or Kernel in a way that we are able to explore much more state. Nyx provides us with a way to snapshot fuzz an entire Virtual Machine/system. This is, in my opinion, the ideal way to fuzz things because you are drastically closing the gap between a contrived fuzzing environment and how the target applications exist in the "real-world". Now obviously there are tradeoffs here, one being the complexity of the fuzzing tooling itself. But, I think given the propensity of complex native code applications to harbor infinite bugs, the manual labor and complexity are worth it in order to increase the bug-finding potential of our fuzzing workflow.

And so, in my pursuit of understanding how Nyx works so that I could build a fuzzer ontop of it, I revisited [gamozolabs (Brandon Falk's)](https://twitter.com/gamozolabs) stream [paper review](https://www.youtube.com/watch?v=JpU-jrFnmfE) he did on the [Nyx paper](https://nyx-fuzz.com/papers/). It's a great stream, the Nyx authors were present in Twitch chat and so there were some good back and forths and the stream really highlights what an amazing utility Nyx is for fuzzing. But something *else* besides Nyx piqued my interest during the stream! During the stream, Gamozo described a fuzzing architecture he had previously built that utilized the Bochs emulator to snapshot fuzz complex targets and entire systems. This architecture sounded extremely interesting and clever to me, and coincidentally it had several attributes in common with a sandboxing utility I had been designing with a friend for fuzzing as well. 

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
- it can be made to be portable, ie, there is nothing stopping us for running this fuzzer on Windows instead of just Linux,
- it will allow me to do a lot of learning and low-level computing research and learning.

So all things considered, this seemed like the ideal project to implement on the blog and so I reached out to Gamozo to make sure he'd be ok with it as I didn't want to be seen as clout chasing off of his ideas and he was very charitable and encouraged me to do it. So huge thanks to Gamozo for sharing so much content and we're off to developing the fuzzer. 

Also huge shoutout to [@is_eqv](https://twitter.com/is_eqv) and [@ms_s3c](https://twitter.com/ms_s3c) at least two of the Nyx authors who are always super friendly and charitable with their time/answering questions. Some great people to have around.

Another huge shoutout to [@Kharosx0](https://twitter.com/Kharosx0) for helping me understand Bochs and for answering all my questions about my design intentions, another very charitable person who is always helping out on the Fuzzing discord.

## Bochs
What is Bochs? Good question. [Bochs](https://bochs.sourceforge.io/) is an x86 full-system emulator capable of running an entire operating system with software-simulated hardware devices. In short, it's a JIT-less, smaller, less-complex emulation tool similar to QEMU but with way less use-cases and way less performant. Instead of taking QEMU's approach of "let's emulate anything and everything and do it with good performance", Bochs has taken the approach of "let's emulate an entire x86 system 100% in software without worrying about performance for the most part. This approach has its obvious drawbacks, but if you are only interested in running x86 systems, Bochs is a great utility. We are going to use Bochs as the target execution engine in our fuzzer. Our target code will run inside Bochs. So if we are fuzzing the Linux Kernel for instance, that kernel will live and execute inside Bochs. Bochs is written in C++ and apparently still maintained, but do not expect much code changes or rapid development, the last release was over 2 years ago. 

## Fuzzer Architecture
This is where we discuss how the fuzzer will be designed according to the information laid out on stream by Gamozo. In simple terms, we will create a "fuzzer" process, which will execute Bochs, which in turn is executing our fuzz target. Instead of snapshotting and restoring our target each fuzzing iteration, we will reset Bochs which contains the target and all of the target system's simulated state. By snapshotting and restoring Bochs, we are snapshotting and restoring our target.

Going a bit deeper, this setup requires us to sandbox Bochs and run it inside of our "fuzzer" process. In an effort to isolate Bochs from the user's OS and Kernel, we will sandbox Bochs so that it cannot interact with our operating system. This allows us to achieve a few things, but chiefly this should make Bochs deterministic. As Gamozo explains on stream, isolating Bochs from the operating system, prevents Bochs from accessing any random/randomish data sources. This means that we will prevent Bochs from making syscalls into the kernel as well as executing any instructions that retrieve hardware-sourced data such as `CPUID` or something similar. I actually haven't given much thought to the latter yet, but syscalls I have a plan for. With Bochs isolated from the operating system, we can expect it to behave the same way each fuzzing iteration. Given Fuzzing Input A, Bochs should execute exactly the same way for 1 trillion successive iterations.

Secondly, it also means that the entirety of Bochs' state will be contained within our sandbox, which should enable us to reset Bochs' state more easily instead of it being a remote process. In a paradigm where Bochs executes as intended as a normal Linux process for example, resetting its state is not trivial and may require a heavy handed approach such as page table walking in the kernel for each fuzzing iteration or something even worse. 

So in general, this is how our fuzzing setup should look:
![Fuzzer Architecture](/assets/images/pwn/FuzzingArch.PNG)

In order to provide a sandboxed environment, we must load an executable Bochs image into our own fuzzer process. So for this, I've chosen to build Bochs as an ELF and then load the ELF into my fuzzer process in memory. Let's dive into how that has been accomplished thus far. 

## Loading an ELF In Memory
So in order to make this portion of loading Bochs in memory in the most simplistic way possible, I've chosen to compile Bochs as a `-static-pie` ELF. Now this means that the built ELF has no expectations about where it is loaded. In its `_start` routine, it actually has all of the logic of the normal OS ELF loader necessary to perform all of its own relocations. How cool is that? But before we get too far ahead of ourselves, the first goal will just be to simply build and load a `-static-pie` test program and make sure we can do that correctly. 

In order to make sure we have everything correctly implemented, we'll make sure that the test program can correctly access any command line arguments we pass and can execute and exit.
```c
#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    printf("Argument count: %d\n", argc);
    printf("Args:\n");
    for (int i = 0; i < argc; i++) {
        printf("   -%s\n", argv[i]);
    }

    size_t iters = 0;
    while (1) {
        printf("Test alive!\n");
        sleep(1);
        iters++;

        if (iters > 5) { return 0; }
    }
}
```
Remember, at this point we don't sandbox our loaded program at all, all we're trying to do at this point is load it in our fuzzer virtual address space and jump to it and make sure the stack and everything is correctly setup. So we could run into issues that aren't real issues if jump straight into executing Bochs at this point.

So compiling the `test` program and examining it with `readelf -l`, we can see that there is actually a `DYNAMIC` segment. Likely because of the relocations that need to be performed during the aforementioned `_start` routine.

```console
dude@lol:~/lucid$ gcc test.c -o test -static-pie
dude@lol:~/lucid$ file test
test: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, BuildID[sha1]=6fca6026edb756fa32c966844b29529d579e83b9, for GNU/Linux 3.2.0, not stripped
dude@lol:~/lucid$ readelf -l test

Elf file type is DYN (Shared object file)
Entry point 0x9f50
There are 12 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000008158 0x0000000000008158  R      0x1000
  LOAD           0x0000000000009000 0x0000000000009000 0x0000000000009000
                 0x0000000000094d01 0x0000000000094d01  R E    0x1000
  LOAD           0x000000000009e000 0x000000000009e000 0x000000000009e000
                 0x00000000000285e0 0x00000000000285e0  R      0x1000
  LOAD           0x00000000000c6de0 0x00000000000c7de0 0x00000000000c7de0
                 0x0000000000005350 0x0000000000006a80  RW     0x1000
  DYNAMIC        0x00000000000c9c18 0x00000000000cac18 0x00000000000cac18
                 0x00000000000001b0 0x00000000000001b0  RW     0x8
  NOTE           0x00000000000002e0 0x00000000000002e0 0x00000000000002e0
                 0x0000000000000020 0x0000000000000020  R      0x8
  NOTE           0x0000000000000300 0x0000000000000300 0x0000000000000300
                 0x0000000000000044 0x0000000000000044  R      0x4
  TLS            0x00000000000c6de0 0x00000000000c7de0 0x00000000000c7de0
                 0x0000000000000020 0x0000000000000060  R      0x8
  GNU_PROPERTY   0x00000000000002e0 0x00000000000002e0 0x00000000000002e0
                 0x0000000000000020 0x0000000000000020  R      0x8
  GNU_EH_FRAME   0x00000000000ba110 0x00000000000ba110 0x00000000000ba110
                 0x0000000000001cbc 0x0000000000001cbc  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x00000000000c6de0 0x00000000000c7de0 0x00000000000c7de0
                 0x0000000000003220 0x0000000000003220  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .rela.dyn .rela.plt 
   01     .init .plt .plt.got .plt.sec .text __libc_freeres_fn .fini 
   02     .rodata .stapsdt.base .eh_frame_hdr .eh_frame .gcc_except_table 
   03     .tdata .init_array .fini_array .data.rel.ro .dynamic .got .data __libc_subfreeres __libc_IO_vtables __libc_atexit .bss __libc_freeres_ptrs 
   04     .dynamic 
   05     .note.gnu.property 
   06     .note.gnu.build-id .note.ABI-tag 
   07     .tdata .tbss 
   08     .note.gnu.property 
   09     .eh_frame_hdr 
   10     
   11     .tdata .init_array .fini_array .data.rel.ro .dynamic .got
``` 

