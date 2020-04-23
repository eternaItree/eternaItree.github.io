---
layout: single
title: HEVD Exploits -- Windows 7 x86 Use-After-Free
date: 2020-04-23
classes: wide
header:
  teaser: /assets/images/avatar.jpg
tags:
  - Exploit Dev
  - Drivers
  - Windows
  - x86
  - Shellcoding
  - Kernel Exploitation
  - UAF
---

## Introduction
Continuing on with my goal to develop exploits for the [Hacksys Extreme Vulnerable Driver](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver). **I will be using HEVD 2.0**. There are a ton of good blog posts out there walking through various HEVD exploits. I recommend you read them all! I referenced them heavily as I tried to complete these exploits. Almost nothing I do or say in this blog will be new or my own thoughts/ideas/techniques. There were instances where I diverged from any strategies I saw employed in the blogposts out of necessity or me trying to do my own thing to learn more.

**This series will be light on tangential information such as:**
+ how drivers work, the different types, communication between userland, the kernel, and drivers, etc
+ how to install HEVD,
+ how to set up a lab environment
+ shellcode analysis

The reason for this is simple, the other blog posts do a much better job detailing this information than I could ever hope to. It feels silly writing this blog series in the first place knowing that there are far superior posts out there; I will not make it even more silly by shoddily explaining these things at a high-level in poorer fashion than those aforementioned posts. Those authors have way more experience than I do and far superior knowledge, I will let them do the explaining. :)

This post/series will instead focus on my experience trying to craft the actual exploits.

## Thanks
- To [@r0oki7](https://twitter.com/r0otki7) for their [walkthrough,](https://rootkits.xyz/blog/2018/04/kernel-use-after-free/)
- To [@FuzzySec](https://twitter.com/FuzzySec) for their [walkthrough,](http://www.fuzzysecurity.com/tutorials/expDev/19.html)

## UAF Setup
I've never exploited a use-after-free bug on any system before. I vaguely understood the concept before starting this excercise. We need what, in my noob opinion, seems like quite a lot of primities in order to make this work. Obviously HEVD goes out of its way to be vulnerable in precisely the correct way for us to get an exploit working which is perfect for me since I have no experience with this bug class and we're just here to learn. I feel like although we have to utilize multiple functions via IOCTL, this is actually a more simple exploit to pull off than the pool overflow that we just did. 

Also, I wanted to do this on 64 bit; however, most of the strategies I saw outlined required that we use `NtQuerySystemInformation`, which as far as I know requires your process to be elevated to an extent so I wanted to avoid that. On 64 bit, the pool header structure size changes from `0x8` bytes to `0x10` bytes which makes exploitation more cumbersome; however, there are some good walkthroughs out there about how to accomplish this. For now, let's stick to x86. 

What do we need in order to exploit a use-after-free bug? Well, it seems like after doing this excercise we need to be able to do the following: 
+ allocate an object in the non-paged pool,
+ a mechansim that creates a reference to the object as a global variable, ie if our object is allocated at `0xFFFFFFFF`, there is some variable out there in the program that is storing that address for later use,
+ the ability to free the memory and not have the previously established reference NULLed out, ie when the chunk is freed the program author doesn't specify that the reference=NULL,
+ the ability to create "fake" objects that have the same size and **controllable** contents in the non-paged pool,
+ the ability to spray the non-paged pool and create perfectly sized holes so that our UAF and fake objects can be fitted in our created holes,
+ finally, the ability to **use** the no-longer valid reference to our freed chunk. 

## Allocating the UAF Object in the Pool
Let's take a look at the UAF object allocation routine in the driver in IDA. 
![](/assets/images/AWE/uaf1.PNG)