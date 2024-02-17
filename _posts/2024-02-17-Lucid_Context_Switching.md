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
Now with our global set, it's time to edit the functions responsible for making syscalls. Musl is very well organized so finding the syscall invoking logic was not too difficult. For our target architecture, which is x86_64, those syscall invoking functions are in `arch/x86_64/syscall_arch.h`. They are organized by how many arguments the syscall takes:
```c
static __inline long __syscall0(long n)
{
	unsigned long ret;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n) : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall1(long n, long a1)
{
	unsigned long ret;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1) : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall2(long n, long a1, long a2)
{
	unsigned long ret;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2)
						  : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall3(long n, long a1, long a2, long a3)
{
	unsigned long ret;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2),
						  "d"(a3) : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall4(long n, long a1, long a2, long a3, long a4)
{
	unsigned long ret;
	register long r10 __asm__("r10") = a4;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2),
						  "d"(a3), "r"(r10): "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall5(long n, long a1, long a2, long a3, long a4, long a5)
{
	unsigned long ret;
	register long r10 __asm__("r10") = a4;
	register long r8 __asm__("r8") = a5;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2),
						  "d"(a3), "r"(r10), "r"(r8) : "rcx", "r11", "memory");
	return ret;
}

static __inline long __syscall6(long n, long a1, long a2, long a3, long a4, long a5, long a6)
{
	unsigned long ret;
	register long r10 __asm__("r10") = a4;
	register long r8 __asm__("r8") = a5;
	register long r9 __asm__("r9") = a6;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2),
						  "d"(a3), "r"(r10), "r"(r8), "r"(r9) : "rcx", "r11", "memory");
	return ret;
}
```

For syscalls, there is a well defined calling convention. Syscalls take a "syscall number" which determines what syscall you want in `eax`, then the next n parameters are passed in via the registers in order: `rdi`, `rsi`, `rdx`, `r10`, `r8`, and `r9`. 

This is pretty intuitive but the syntax is a bit mystifying, like for example on those `__asm__ __volatile__ ("syscall"` lines, it's kind of hard to see what it's doing. Let's take the most convoluted function, `__syscall6` and break down all the syntax. We can think of the assembly syntax as a format string like for printing, but this is for emitting code instead:

- `unsigned long ret` is where we will store the result of the syscall to indicate whether or not it was a success. In the raw assembly, we can see that there is a `:` and then `"=a(ret)"`, this first set of parameters after the initial colon is to indicate *output* parameters. We are saying please store the result in `eax` (symbolized in the syntax as `a`) into the variable `ret`.
- The next series of params after the next colon are *input* parameters. `"a"(n)` is saying, place the function argument `n`, which is the syscall number, into `eax` which is symbolized again as `a`. Next is store `a1` in `rdi`, which is symbolized as `D`, and so forth
- Arguments 4-6 are placed in registers above, for instance the syntax `register long r10 __asm__("r10") = a4;` is a strong compiler hint to store `a4` into `r10`. And then later we see `"r"(r10)` says input the variable `r10` into a general purpose register (which is already satisfied).
- The last set of colon-separated values are known as "clobbers". These tell the compiler what our syscall is expected to corrupt. So the syscall calling convention specifies that `rcx`, `r11`, and memory may be overwritten by the kernel.

With the syntax explained, we see what is taking place. The job of these functions is to translate the function call into a syscall. The calling convention for functions, known as the System V ABI, is different from that of a syscall, the register utilization differs. So when we call `__syscall6` and pass its arguments, each argument is stored in the following register:

- `n` → `rax`
- `a1` → `rdi`
- `a2` → `rsi`
- `a3` → `rdx`
- `a4` → `rcx`
- `a5` → `r8`
- `a6` → `r9`

So the compiler will take those function args from the System V ABI and translate them into the syscall via the assembly that we explained above. So now these are the functions we need to edit so that we don't emit that `syscall` instruction and instead call into Lucid.

## Conditionally Calling Into Lucid
So we need a way in these function bodies to call into Lucid instead of emit `syscall` instructions. To do so we need to define our own calling convention, for now I've been using the following:
- `r15`: contains the address of the global Lucid execution context
- `r14`: contains an "exit reason" which is just an `enum` explaining why we are context switching
- `r13`: is the base address of the register bank structure of the Lucid execution context, we need this memory section to store our register values to save our state when we context switch
- `r12`: stores the address of the "exit handler" which is the function to call to context switch

This will no doubt change some as we add more features/functionality. I should also note that it is the functions responibility to preserve these values according to the ABI, so the function caller expects that these won't change during a function call, well we are changing them. That's ok because in the function where we use them, we are marking them as clobbers, remember? So the compiler is aware that they change, what the compiler is going to do now is before it executes any code, it's going to push those registers onto the stack to save them, and then before exiting, pop them back into the registers so that the caller gets back the expected values. So we're free to use them. 

So to alter the functions, I changed the function logic to first check if we have a global Lucid execution context, if we do not, then execute the normal Musl function, you can see that here as I've moved the normal function logic out to a separate function called `__syscall6_original`:
```c
static __inline long __syscall6_original(long n, long a1, long a2, long a3, long a4, long a5, long a6)
{
	unsigned long ret;
	register long r10 __asm__("r10") = a4;
	register long r8  __asm__("r8")  = a5;
	register long r9  __asm__("r9")  = a6;
	__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n), "D"(a1), "S"(a2), "d"(a3), "r"(r10),
							"r"(r8), "r"(r9) : "rcx", "r11", "memory");

	return ret;
}

static __inline long __syscall6(long n, long a1, long a2, long a3, long a4, long a5, long a6)
{
	if (!g_lucid_ctx) { return __syscall6_original(n, a1, a2, a3, a4, a5, a6); }
```

However, if we are running under Lucid, I set up our calling convention by explicitly setting the registers `r12-r15` in accordance to what we are expecting there when we context-switch to Lucid. 
```c
static __inline long __syscall6(long n, long a1, long a2, long a3, long a4, long a5, long a6)
{
    if (!g_lucid_ctx) { return __syscall6_original(n, a1, a2, a3, a4, a5, a6); }
	
    register long ret;
    register long r12 __asm__("r12") = (size_t)(g_lucid_ctx->exit_handler);
    register long r13 __asm__("r13") = (size_t)(&g_lucid_ctx->register_bank);
    register long r14 __asm__("r14") = SYSCALL;
    register long r15 __asm__("r15") = (size_t)(g_lucid_ctx);
```

Now with our calling convention set up, we can then use inline assembly as before. Notice we've replaced the `syscall` instruction with `call r12`, calling our exit handler as if it's a normal function:
```c
__asm__ __volatile__ (
        "mov %1, %%rax\n\t"
	"mov %2, %%rdi\n\t"
	"mov %3, %%rsi\n\t"
	"mov %4, %%rdx\n\t"
	"mov %5, %%r10\n\t"
	"mov %6, %%r8\n\t"
	"mov %7, %%r9\n\t"
        "call *%%r12\n\t"
        "mov %%rax, %0\n\t"
        : "=r" (ret)
        : "r" (n), "r" (a1), "r" (a2), "r" (a3), "r" (a4), "r" (a5), "r" (a6),
		  "r" (r12), "r" (r13), "r" (r14), "r" (r15)
        : "rax", "rcx", "r11", "memory"
    );
	
	return ret;
```

So now we're calling the exit handler instead of syscalling into the kernel, and all of the registers are setup *as if* we're syscalling. We've also got our calling convention registers set up. Let's see what happens when we land on the exit handler, a function that is implemented in Rust inside Lucid. We are jumping from Bochs code directly to Lucid code!

## Implementing a Context Switch
The first thing we need to do is create a function body for the exit handler. In Rust, we can make the function visible to Bochs (via our edited Musl) by declaring the function as an extern C function and giving it a label in inline assembly as such:
```rust
extern "C" { fn exit_handler(); }
global_asm!(
    ".global exit_handler",
    "exit_handler:",
```

So this function is what will be jumped to by Bochs when it tries to syscall under Lucid. The first thing we need to consider is that we need to keep track of Bochs' state the way the kernel would upon entry to the context switching routine. The first thing we'll want to save off is the general purpose registers. By doing this, we can preserve the state of the registers, but also unlock them for our own use. Since we save them first, we're then free to use them. Remember that our calling convention uses `r13` to store the base address of the execution context register bank:
```rust
#[repr(C)]
#[derive(Default, Clone)]
pub struct RegisterBank {
    pub rax:    usize,
    rbx:        usize,
    rcx:        usize,
    pub rdx:    usize,
    pub rsi:    usize,
    pub rdi:    usize,
    rbp:        usize,
    rsp:        usize,
    pub r8:     usize,
    pub r9:     usize,
    pub r10:    usize,
    r11:        usize,
    r12:        usize,
    r13:        usize,
    r14:        usize,
    r15:        usize,
}
```

We can save the register values then by doing this:
```rust
// Save the GPRS to memory
"mov [r13 + 0x0], rax",
"mov [r13 + 0x8], rbx",
"mov [r13 + 0x10], rcx",
"mov [r13 + 0x18], rdx",
"mov [r13 + 0x20], rsi",
"mov [r13 + 0x28], rdi",
"mov [r13 + 0x30], rbp",
"mov [r13 + 0x38], rsp",
"mov [r13 + 0x40], r8",
"mov [r13 + 0x48], r9",
"mov [r13 + 0x50], r10",
"mov [r13 + 0x58], r11",
"mov [r13 + 0x60], r12",
"mov [r13 + 0x68], r13",
"mov [r13 + 0x70], r14",
"mov [r13 + 0x78], r15",
```

This will save the register values to memory in the memory bank for preservation. Next, we'll want to preserve the CPU's flags, luckily there is a single instruction for this purpose which pushes the flag values to the stack called `pushfq`.

We're using a pure assembly stub right now but we'd like to start using Rust at some point, that point is now. We have saved all the state we can for now, and it's time to call into a real Rust function that will make programming and implementation easier. To call into a function though, we need to set up the register values to adhere to the function calling ABI remember. Two pieces of data that we want to be accessible are the execution context and the reason why we exited. Those are in `r15` and `r14` respectively remember. So we can simply place those into the registers used for passing function arguments and call into a Rust function called `lucid_handler` now. 
```rust
// Bochs saves its GPRs before calling into us, so all we need to do is 
// save the flags
"pushfq",

// Set up the function arguments for lucid_handler according to ABI
"mov rdi, r15", // Put the pointer to the context into RDI
"mov rsi, r14", // Put the exit reason into RSI

// At this point, we've been called into by Bochs, this should mean that 
// at the beginning of our exit_handler, rsp was only 8-byte aligned and
// thus, by ABI, we cannot legally call into a Rust function since to do so
// requires rsp to be 16-byte aligned. Luckily, `pushfq` just 16-byte
// aligned the stack for us and so we are free to `call`
"call lucid_handler",
```

So now, we are free to execute real Rust code! Here is `lucid_handler` as of now:
```rust
// This is where the actual logic is for handling the Bochs exit, we have to 
// use no_mangle here so that we can call it from the assembly blob. We need
// to see why we've exited and dispatch to the appropriate function
#[no_mangle]
fn lucid_handler(context: *mut LucidContext, exit_reason: i32) {
    // We have to make sure this bad boy isn't NULL 
    if context.is_null() {
        println!("LucidContext pointer was NULL");
        fatal_exit();
    }

    // Ensure that we have our magic value intact, if this is wrong, then we 
    // are in some kind of really bad state and just need to die
    let magic = LucidContext::ptr_to_magic(context);
    if magic != CTX_MAGIC {
        println!("Invalid LucidContext Magic value: 0x{:X}", magic);
        fatal_exit();
    }

    // Before we do anything else, save the extended state
    let save_inst = LucidContext::ptr_to_save_inst(context);
    if save_inst.is_err() {
        println!("Invalid Save Instruction");
        fatal_exit();
    }
    let save_inst = save_inst.unwrap();

    // Get the save area
    let save_area =
        LucidContext::ptr_to_save_area(context, SaveDirection::FromBochs);

    if save_area == 0 || save_area % 64 != 0 {
        println!("Invalid Save Area");
        fatal_exit();
    }

    // Determine save logic
    match save_inst {
        SaveInst::XSave64 => {
            // Retrieve XCR0 value, this will serve as our save mask
            let xcr0 = unsafe { _xgetbv(0) } as u64;

            // Call xsave to save the extended state to Bochs save area
            unsafe { _xsave64(save_area as *mut u8, xcr0); }             
        },
        SaveInst::FxSave64 => {
            // Call fxsave to save the extended state to Bochs save area
            unsafe { _fxsave64(save_area as *mut u8); }
        },
        _ => (), // NoSave
    }

    // Try to convert the exit reason into BochsExit
    let exit_reason = BochsExit::try_from(exit_reason);
    if exit_reason.is_err() {
        println!("Invalid Bochs Exit Reason");
        fatal_exit();
    }
    let exit_reason = exit_reason.unwrap();
    
    // Determine what to do based on the exit reason
    match exit_reason {
        BochsExit::Syscall => {
            syscall_handler(context);
        },
    }

    // Restore extended state, determine restore logic
    match save_inst {
        SaveInst::XSave64 => {
            // Retrieve XCR0 value, this will serve as our save mask
            let xcr0 = unsafe { _xgetbv(0) } as u64;

            // Call xrstor to restore the extended state from Bochs save area
            unsafe { _xrstor64(save_area as *const u8, xcr0); }             
        },
        SaveInst::FxSave64 => {
            // Call fxrstor to restore the extended state from Bochs save area
            unsafe { _fxrstor64(save_area as *const u8); }
        },
        _ => (), // NoSave
    }
}
```

There are a few important pieces here to discuss. 
