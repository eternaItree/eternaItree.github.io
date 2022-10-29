---
layout: single
title: "PAWNYABLE UAF Walkthrough (Holstein v3)"
date: 2022-10-29
classes: wide
header:
  teaser: /assets/images/avatar.jpg
tags:
  - Linux kernel
  - Pwn
  - UAF
  - Tutorial
  - CTF
  - Walkthrough
---

## Introduction

I've been wanting to learn Linux Kernel exploitation for some time and a couple months ago [@ptrYudai](https://twitter.com/ptrYudai) from [@zer0pts](https://twitter.com/zer0pts) tweeted that they released the beta version of their website [PAWNYABLE!](https://pawnyable.cafe/), which is a "resource for middle to advanced learners to study Binary Exploitation". The first section on the website with material already ready is "Linux Kernel", so this was a perfect place to start learning. 

The author does a great job explaining everything you need to know to get started, things like: setting up a debugging environment, CTF-specific tips, modern kernel exploitation mitigations, using QEMU, manipulating images, etc, so this blogpost will focus exclusively on my experience with the challenge and the way I decided to solve it.

### What I Started With

PAWNYABLE ended up being a great way for me to start learning about Linux Kernel exploitation, mainly because I didn't have to spend any time getting up to speed on a kernel subsystem in order to start wading into the exploitation metagame. For instance, if you are the type of person who learns by doing, and you're first attempt at learning about this stuff was to write your own exploit for CVE-2022-32250, you would first have to spend a considerable amount of time learning about Netfilter. Instead, PAWNYABLE gives you a straightforward example of a vulnerability in one of a handful of bug-classes, and then gets to work showing you how you could exploit it. I think this strategy is great for beginners like me. It's worth noting that after having spent some time with PAWNYABLE, I have been able to write some exploits for real world bugs similar to CVE-2022-32250, so my strategy did prove to be fruitful (at least for me). 

I've been doing low-level binary stuff (mostly on Linux) for the past 3 years. Initially I was very interested in learning binary exploitation but starting gravitating towards vulnerability discovery and fuzzing. Fuzzing has captivated me since early 2020, and developing my own fuzzing frameworks actually lead to me working as a full time software developer for the last couple of years. So after going pretty deep with fuzzing (objectively not that deep as it relates to the entire fuzzing space, but deep for the uninitiated) , I wanted to circle back and learn at least some aspect of binary exploitation that applied to modern targets.

The Linux Kernel, as a target, seemed like a happy marriage between multiple things: it's relatively easy to write exploits for due to a lack of mitigations, exploitable bugs and their resulting exploits have a wide and high impact, and there are active bounty systems/programs for Linux Kernel exploits. As a quick side-note, there have been some tremendous strides made in the world of Linux Kernel fuzzing in the last few years so I knew that specializing in this space would allow me to get up to speed on those approaches/tools. 

So coming into this, I had a pretty good foundation of basic binary exploitation (mostly dated Windows and Linux userland stuff), a few years of C development (to include a few Linux Kernel modules), and some reverse engineering skills. 

### What I Did

To get started, I read through the following PAWNYABLE sections (section names have been Google translated to English):

- Introduction to kernel exploits
- kernel debugging with gdb
- security mechanism (Overview of Exploitation Mitigations)
- Compile and transfer exploits (working with the kernel image)

This was great as a starting point because everything is so well organized you don't have to spend time setting up your environment, its basically just copy pasting a few commands and you're off and remotely debugging a kernel via GDB (with GEF even). 

Next, I started working on the first challenge which is a stack-based buffer overflow vulnerability in Holstein v1. This is a great starting place because right away you get control of the instruction pointer and from there, you're learning about things like the way CTF players (and security researchers) often leverage kernel code execution to escalate privileges like `prepare_kernel_creds` and `commit_creds`. 

You can write an exploit that bypasses mitigations or not, it's up to you. I started slowly and wrote an exploit with no mitigations enabled, then slowly turned the mitigations up and changed the exploit as needed. 

After that, I started working on a popular Linux kernel pwn challenge called "kernel-rop" from hxpCTF 2020. I followed along and worked alongside the following blogposts from [@\_lkmidas](https://twitter.com/_lkmidas). 

