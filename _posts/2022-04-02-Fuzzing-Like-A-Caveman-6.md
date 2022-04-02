---
layout: single
title: "Fuzzing Like A Caveman 6: Binary Only Snapshot Fuzzing Harness"
date: 2022-04-02
classes: wide
header:
  teaser: /assets/images/avatar.jpg
tags:
  - fuzzing
  - harnessing
  - snapshot_fuzzing
---

## Introduction
It's been a while since I've done one of these, and one of my goals this year is to do more so here we are. A side project of mine is kind of reaching a good stopping point so I'll have more free-time to do my own research and blog again. Looking forward to sharing more and more this year. 

One of the most common questions that comes up in beginner fuzzing circles (of which I'm obviously a member) is how to harness a target so that it can be fuzzed in memory, as some would call in 'persistent' fashion, in order to gain performance. Persistent fuzzing has a niche use-case where the target doesn't touch much global state from fuzzcase to fuzzcase, an example would be a tight fuzzing loop for a single API in a library, or maybe a single function in a binary.

This style of fuzzing is faster than re-executing the target from scratch over and over as we bypass all the heavy syscalls/kernel routines associated with creating and destroying task structs. 

However, with binary targets for which we don't have source code, it's sometimes hard to discern what global state we're affecting while executing any code path without some heavy reverse engineering (disgusting, work? gross). Additionally, we often want to fuzz a wider loop. It doesn't do us much good to fuzz a function which returns a struct that is then never read or consumed in our fuzzing workflow. With these things in mind, we often find that 'snapshot' fuzzing would be a more robust workflow for binary targets, or even production binaries for which, we have source, but have gone through the sausage factory of enterprise build systems. 

So today, we're going to learn how to take an arbitrary binary only target that takes an input file from the user and turn it into a target that takes its input from memory instead and lends itself well to having its state reset between fuzzcases. 

## Target (Easy Mode)
For the purposes of this blogpost, we're going to harness objdump to be snapshot fuzzed. This will serve our purposes because its relatively simple (single threaded, single process) and it's a common fuzzing target, especially as people do development work on their fuzzers. The point of this is not to impress you by sandboxing some insane target like Chrome, but to show beginners how to start thinking about harnessing. You want to lobotomize your targets so that they are unrecognizable to their original selves but retain the same semantics. You can get as creative as you want, and honestly, sometimes harnessing targets is some of the most satisfying work related to fuzzing. It feels great to successfully sandbox a target and have it play nice with your fuzzer. On to it then. 

## Hello World
The first step is to determine how we want to change objdump's behavior. Let's try running it under `strace` and disassemble `ls` and see how it behaves at the syscall level with `strace objdump -D /bin/ls`. What we're looking for is the point where `objdump` starts interacting with our input, `/bin/ls` in this case. In the output, if you scroll down past the boilerplate stuff, you can see the first appearance of `/bin/ls`: 
```
stat("/bin/ls", {st_mode=S_IFREG|0755, st_size=133792, ...}) = 0
stat("/bin/ls", {st_mode=S_IFREG|0755, st_size=133792, ...}) = 0
openat(AT_FDCWD, "/bin/ls", O_RDONLY)   = 3
fcntl(3, F_GETFD)                       = 0
fcntl(3, F_SETFD, FD_CLOEXEC)           = 0
```
 ***Keep in mind that as you read through this, if you're following along at home, your output might not match mine exactly. I'm likely on a different distribution than you running a different objdump than you. But the point of the blogpost is to just show concepts that you can be creative on your own.***

I also noticed that the program doesn't close our input file until the end of execution:
```
read(3, "\0\0\0\0\0\0\0\0\10\0\"\0\0\0\0\0\1\0\0\0\377\377\377\377\1\0\0\0\0\0\0\0"..., 4096) = 2720
write(1, ":(%rax)\n  21ffa4:\t00 00         "..., 4096) = 4096
write(1, "x0,%eax\n  220105:\t00 00         "..., 4096) = 4096
close(3)                                = 0
write(1, "023e:\t00 00                \tadd "..., 2190) = 2190
exit_group(0)                           = ?
+++ exited with 0 +++
```

This is good to know, we'll need our harness to be able to emulate an input file fairly well since objdump doesn't just read our file into a memory buffer in one shot or `mmap()` the input file. It is continuously reading from the file throughout the `strace` output. 

Since we don't have source code for the target, we're going to affect behavior by using an `LD_PRELOAD` shared object. By using an `LD_PRELOAD` shared object, we should be able to hook the wrapper functions around the syscalls that interact with our input file and change their behavior to suit our purposes. If you are unfamiliar with dynamic linking or `LD_PRELOAD`, this would be a good stopping point to go Google around for more information. For starters, let's just get a *Hello, World!* shared object loaded. 

We can utilize `gcc` [Function Attributes](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html) to have our shared object execute code when it is loaded by the target by leveraging the `constructor` attribute. 

So our code so far will look like this:
```c
/* 
Compiler flags: 
gcc -shared -Wall -Werror -fPIC blog_harness.c -o blog_harness.so -ldl
*/

#include <stdio.h> /* printf */

// Routine to be called when our shared object is loaded
__attribute__((constructor)) static void _hook_load(void) {
    printf("** LD_PRELOAD shared object loaded!\n");
}
```

I added the compiler flags needed to compile to the top of the file as a comment. I got these flags from this blogpost on using `LD_PRELOAD` shared objects a while ago: https://tbrindus.ca/correct-ld-preload-hooking-libc/.

We can now use the `LD_PRELOAD` environment variable and run objdump with our shared object which should print when loaded:
```
h0mbre@ubuntu:~/blogpost$ LD_PRELOAD=/home/h0mbre/blogpost/blog_harness.so objdump -D /bin/ls > /tmp/output.txt && head -n 20 /tmp/output.txt
**> LD_PRELOAD shared object loaded!

/bin/ls:     file format elf64-x86-64


Disassembly of section .interp:

0000000000000238 <.interp>:
 238:   2f                      (bad)  
 239:   6c                      ins    BYTE PTR es:[rdi],dx
 23a:   69 62 36 34 2f 6c 64    imul   esp,DWORD PTR [rdx+0x36],0x646c2f34
 241:   2d 6c 69 6e 75          sub    eax,0x756e696c
 246:   78 2d                   js     275 <_init@@Base-0x34e3>
 248:   78 38                   js     282 <_init@@Base-0x34d6>
 24a:   36 2d 36 34 2e 73       ss sub eax,0x732e3436
 250:   6f                      outs   dx,DWORD PTR ds:[rsi]
 251:   2e 32 00                xor    al,BYTE PTR cs:[rax]

Disassembly of section .note.ABI-tag:
```

It works, now we can looking for functions to hook.

## Looking for Hooks
