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

## C Library
Normally programmers don't have to worry about making syscalls directly. They instead use functions that are defined and implemented in a C library instead, and its these functions that actually make the syscalls. You can think of these functions as wrappers around a syscall. For instance if you use the C library function for `open`, you're not directly making a syscall, you're calling into the library's `open` function and that function is the one emitting a `syscall` instruction that actually peforms the context switch into the kernel. Doing things this way takes a lot of the portability work off of the programmer's shoulders because the guts of the library functions perform all of the conditional checks for environmental variables and execute accordingly. Programmers just call the `open` function and don't have to worry about things like syscall numbers, error handling, etc as those things are kept abstracted and uniform in the code exported to the programmer. 

This provides a nice chokepoint for our purposes, since Bochs programmers also use C library functions instead of invoking syscalls directly. When Bochs wants to make a syscall, it's going to call a C library function. This gives us an opportunity to *intercept* these syscalls before they are made. We can insert our own logic into these functions that check to see whether or not Bochs is executing under Lucid, if it is, we can insert logic that directs execution to Lucid instead of the kernel. In pseudocode we can achieve something like the following:
```
fn syscall()
  if lucid:
    lucid_syscall()
  else:
    normal_syscall()
```

## Musl
[Musl](https://musl.libc.org/) is a C library that is meant to be "lightweight." This gives us some simplicity to work with vs. something like Glibc which is a monstrosity an affront to God. Importantly, Musl is reputationally great for static linking, which is what we need when we build our static PIE Bochs. So the idea here is that we can manually alter Musl code to change how syscall-invoking wrapper functions work so that we can hijack execution in a way that context-switches into Lucid rather than the kernel. 

In this post we'll be working with Musl 1.2.4 which is the latest version as of today. 

## Baby Steps
Instead of jumping straight into Bochs, we'll be using a test program for the purposes of developing our first context-switching routines. This is just easier. The test program is this:
```c
#include <stdio.h>
#include <unistd.h>
#include <lucid.h>

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

        if (iters == 5) { break; }
    }

    printf("g_lucid_ctx: %p\n", g_lucid_ctx);
}
```

The program will just tell us it's argument count, each argument, live for ~5 seconds, and then print the memory address of a Lucid execution context data structure. This data structure will be allocated and initialized by Lucid if the program is running under Lucid, and it will be NULL otherwise. So how do we accomplish this? 

## Execution Context Tracking
Our problem is that we need a globally accessible way for the program we load (eventually Bochs) to tell whether or not its running under Lucid or running as normal. We also have to provide many data structures and function addresses to Bochs so we need a vehicle do that. 

What I've done is I've just created my own header file and placed it in Musl called `lucid.h`. This file defines all of the Lucid-specific data structures we need Bochs to have access to when it's compiled against Musl. So in the header file right now we've defined a `lucid_ctx` data structure, and we've also created a global instance of one called `g_lucid_ctx`:
```c
// An execution context definition that we use to switch contexts between the
// fuzzer and Bochs. This should contain all of the information we need to track
// all of the mutable state between snapshots that we need such as file data.
// This has to be consistent with LucidContext in context.rs
typedef struct lucid_ctx {
    // This must always be the first member of this struct
    size_t exit_handler;
    int save_inst;
    size_t save_size;
    size_t lucid_save_area;
    size_t bochs_save_area;
    struct register_bank register_bank;
    size_t magic;
} lucid_ctx_t;

// Pointer to the global execution context, if running inside Lucid, this will
// point to the a struct lucid_ctx_t inside the Fuzzer 
lucid_ctx_t *g_lucid_ctx;
```

## Program Start Under Lucid
So in Lucid's main function right now we do the following:
- Load Bochs
- Create an execution context
- Jump to Bochs' entry point and start executing

When we jump to Bochs' entry point, one of the earliest functions called is a function in Musl called `_dlstart_c` located in the source file `dlstart.c`. Right now, we create that global execution context in Lucid on the heap, and then we pass that address in arbitrarily chosen `r15`. This whole function will have to change eventually because we'll want to context switch from Lucid to Bochs to perform this in the future, but for now this is all we do:
```rust
pub fn start_bochs(bochs: Bochs, context: Box<LucidContext>) {
    // rdx: we have to clear this register as the ABI specifies that exit
    // hooks are set when rdx is non-null at program start
    //
    // rax: arbitrarily used as a jump target to the program entry
    //
    // rsp: Rust does not allow you to use 'rsp' explicitly with in(), so we
    // have to manually set it with a `mov`
    //
    // r15: holds a pointer to the execution context, if this value is non-
    // null, then Bochs learns at start time that it is running under Lucid
    //
    // We don't really care about execution order as long as we specify clobbers
    // with out/lateout, that way the compiler doesn't allocate a register we 
    // then immediately clobber
    unsafe {
        asm!(
            "xor rdx, rdx",
            "mov rsp, {0}",
            "mov r15, {1}",
            "jmp rax",
            in(reg) bochs.rsp,
            in(reg) Box::into_raw(context),
            in("rax") bochs.entry,
            lateout("rax") _,   // Clobber (inout so no conflict with in)
            out("rdx") _,       // Clobber
            out("r15") _,       // Clobber
        );
    }
}
```

So when we jump to Bochs entry point having come from Lucid, `r15` should hold the address of the execution context. In `_dlstart_c`, we can check `r15` and act accordingly. Here are those additions I made to Musl's start routine:
```c
hidden void _dlstart_c(size_t *sp, size_t *dynv)
{
	// The start routine is handled in inline assembly in arch/x86_64/crt_arch.h
	// so we can just do this here. That function logic clobbers only a few
	// registers, so we can have the Lucid loader pass the address of the 
	// Lucid context in r15, this is obviously not the cleanest solution but
	// it works for our purposes
	size_t r15;
	__asm__ __volatile__(
		"mov %%r15, %0" : "=r"(r15)
	);

	// If r15 was not 0, set the global context address for the g_lucid_ctx that
	// is in the Rust fuzzer
	if (r15 != 0) {
		g_lucid_ctx = (lucid_ctx_t *)r15;

		// We have to make sure this is true, we rely on this
		if ((void *)g_lucid_ctx != (void *)&g_lucid_ctx->exit_handler) {
			__asm__ __volatile__("int3");
		}
	}

	// We didn't get a g_lucid_ctx, so we can just run normally
	else {
		g_lucid_ctx = (lucid_ctx_t *)0;
	}
```

When this function is called, `r15` remains untouched by the earliest Musl logic. So we use inline assembly to extract the value into a variable called `r15` and check it for data. If it has data, we set the global context variable to the address in `r15`; otherwise we explicitly set it to NULL and run as normal. Now with a global set, we can do runtime checks for our environment and optionally call into the real kernel or into Lucid. 

## Lobotomizing Musl Syscalls
Now with our global set, it's time to edit the functions responsible for making syscalls. After 
