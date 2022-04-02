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
