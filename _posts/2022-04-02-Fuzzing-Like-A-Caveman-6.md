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
First thing we need to do, is create a fake file name to give objdump so that we can start testing things out. We will copy `/bin/ls` into the current working directory and call it `fuzzme`. This will allow us to generically play around with the harness for testing purposes. Now we have our `strace` output, we know that objdump calls `stat()` on the path for our input file (`/bin/ls`) a couple of times before we get that call to `openat()`. Since we know our file hasn't been opened yet, and the syscall uses the path for the first arg, we can guess that this syscall results from the libc exported wrapper function for `stat()` or `lstat()`. I'm going to assume `stat()` since we aren't dealing with any symbolic links for `/bin/ls` on my box. We can add a hook for `stat()` to test to see if we hit it and check if it's being called for our target input file (now changed to `fuzzme`).

In order to create a hook, we will follow a pattern where we define a pointer to the real function via a `typedef` and then we will initialize the pointer as `NULL`. Once we need to resolve the location of the **real** function we are hooking, we can use `dlsym(RLTD_NEXT, <symbol name>)` to get it's location and change the pointer value to the real symbol address. (This will be more clear later on). 

Now we need to hook `stat()` which appears as a `man 3` entry [here](https://linux.die.net/man/3/stat) (meaning it's a libc exported function) as well as a `man 2` entry (meaning it is a syscall). This was confusing to me for the longest time and I often misunderstood how syscalls actually worked because of this insistence on naming collisions. You can read one of the first research blogposts I ever did [here](https://h0mbre.github.io/Learn-C-By-Creating-A-Rootkit/) where the confusion is palpable and I often make erroneous claims. (PS, I'll never edit the old blogposts with errors in them, they are like time capsules, and it's kind of cool to me).

We want to write a function that when called, simply prints something and exits so that we know our hook was hit. For now, our code looks like this:
```c
/* 
Compiler flags: 
gcc -shared -Wall -Werror -fPIC blog_harness.c -o blog_harness.so -ldl
*/

#include <stdio.h> /* printf */
#include <sys/stat.h> /* stat */
#include <stdlib.h> /* exit */

// Filename of the input file we're trying to emulate
#define FUZZ_TARGET "fuzzme"

// Declare a prototype for the real stat as a function pointer
typedef int (*stat_t)(const char *restrict path, struct stat *restrict buf);
stat_t real_stat = NULL;

// Hook function, objdump will call this stat instead of the real one
int stat(const char *restrict path, struct stat *restrict buf) {
    printf("** stat() hook!\n");
    exit(0);
}

// Routine to be called when our shared object is loaded
__attribute__((constructor)) static void _hook_load(void) {
    printf("** LD_PRELOAD shared object loaded!\n");
}
```

However, if we compile and run that, we don't ever print and exit so our hook is not being called. Something is going wrong. Sometimes, file related functions in libc have `64` variants, such as `open()` and `open64()` that are used somewhat interchangably depending on configurations and flags. I tried hooking a `stat64()` but still had no luck with the hook being reached. 

Luckily, I'm not the first person with this problem, there is a [great answer on Stackoverflow about the very issue](https://stackoverflow.com/questions/5478780/c-and-ld-preload-open-and-open64-calls-intercepted-but-not-stat64) that describes how libc doesn't actually export `stat()` the same way it does for other functions like `open()` and `open64()`, instead it exports a symbol called `__xstat()` which has a slightly different signature and requires a new argument called `version` which is meant to describe which version of `stat struct` the caller is expecting. This is supposed to all happen magically under the hood but that's where we live now, so we have to make the magic happen ourselves. The same rules apply for `lstat()` and `fstat()` as well, they have `__lxstat()` and `__fxstat()` respectively.

I found the definitions for the functions [here](https://refspecs.linuxfoundation.org/LSB_1.1.0/gLSB/baselib-xstat-1.html). So we can add the `__xstat()` hook to our shared object in place of the `stat()` and see if our luck changes. Our code now looks like this:
```c
/* 
Compiler flags: 
gcc -shared -Wall -Werror -fPIC blog_harness.c -o blog_harness.so -ldl
*/

#include <stdio.h> /* printf */
#include <sys/stat.h> /* stat */
#include <stdlib.h> /* exit */
#include <unistd.h> /* __xstat, __fxstat */

// Filename of the input file we're trying to emulate
#define FUZZ_TARGET "fuzzme"

// Declare a prototype for the real stat as a function pointer
typedef int (*__xstat_t)(int __ver, const char *__filename, struct stat *__stat_buf);
__xstat_t real_xstat = NULL;

// Hook function, objdump will call this stat instead of the real one
int __xstat(int __ver, const char *__filename, struct stat *__stat_buf) {
    printf("** Hit our __xstat() hook!\n");
    exit(0);
}

// Routine to be called when our shared object is loaded
__attribute__((constructor)) static void _hook_load(void) {
    printf("** LD_PRELOAD shared object loaded!\n");
}
```

Now if we run our shared object, we get the desired outcome, somewhere, our hook is hit. Now we can help ourselves out a bit and print the filenames being requested by the hook and then actually call the real `__xstat()` on behalf of the caller. Now when our hook is hit, we will have to resolve the location of the real `__xstat()` by name, so we'll add a symbol resolving function to our shared object. Our shared object code now looks like this:
```c
/* 
Compiler flags: 
gcc -shared -Wall -Werror -fPIC blog_harness.c -o blog_harness.so -ldl
*/

#define _GNU_SOURCE     /* dlsym */
#include <stdio.h> /* printf */
#include <sys/stat.h> /* stat */
#include <stdlib.h> /* exit */
#include <unistd.h> /* __xstat, __fxstat */
#include <dlfcn.h> /* dlsym and friends */

// Filename of the input file we're trying to emulate
#define FUZZ_TARGET "fuzzme"

// Declare a prototype for the real stat as a function pointer
typedef int (*__xstat_t)(int __ver, const char *__filename, struct stat *__stat_buf);
__xstat_t real_xstat = NULL;

// Returns memory address of *next* location of symbol in library search order
static void *_resolve_symbol(const char *symbol) {
    // Clear previous errors
    dlerror();

    // Get symbol address
    void* addr = dlsym(RTLD_NEXT, symbol);

    // Check for error
    char* err = NULL;
    err = dlerror();
    if (err) {
        addr = NULL;
        printf("Err resolving '%s' addr: %s\n", symbol, err);
        exit(-1);
    }
    
    return addr;
}

// Hook function, objdump will call this stat instead of the real one
int __xstat(int __ver, const char *__filename, struct stat *__stat_buf) {
    // Print the filename requested
    printf("** __xstat() hook called for filename: '%s'\n", __filename);

    // Resolve the address of the real __xstat() on demand and only once
    if (!real_xstat) {
        real_xstat = _resolve_symbol("__xstat");
    }

    // Call the real __xstat() for the caller so everything keeps going
    return real_xstat(__ver, __filename, __stat_buf);
}

// Routine to be called when our shared object is loaded
__attribute__((constructor)) static void _hook_load(void) {
    printf("** LD_PRELOAD shared object loaded!\n");
}
```

Ok so now when we run this, and we check for our print statements, things get a little spicy. 
```
h0mbre@ubuntu:~/blogpost$ LD_PRELOAD=/home/h0mbre/blogpost/blog_harness.so objdump -D fuzzme > /tmp/output.txt && grep "** __xstat" /tmp/output.txt
** __xstat() hook called for filename: 'fuzzme'
** __xstat() hook called for filename: 'fuzzme'
```

So now we can have some fun. 

## __xstat() Hook

So the purpose of this hook will be to lie to objdump and make it think it successfully `stat()` the input file. Remember, we're making a snapshot fuzzing harness so our objective is to constantly be creating new inputs and feeding them to objdump through this harness. Most importantly, our harness will need to be able to represent our variable length inputs (which will be stored purely in memory) as files. Each fuzzcase, the file length can change and our harness needs to accomodate that.

My idea at this point was to create a somewhat "legit" `stat struct` that would normally be returned for our actual file `fuzzme` which is just a copy of `/bin/ls`. We can store this `stat struct` globally and only update the size field as each new fuzz case comes through. So the timeline of our snapshot fuzzing workflow would look something like:
1. Our `constructor` function is called when our shared object is loaded
2. Our `constructor` sets up a global "legit" `stat struct` that we can update for each fuzzcase and pass back to callers of `__xstat()` trying to `stat()` our fuzzing target
3. The imaginary fuzzer runs objdump to the snapshot location
4. Our `__xstat()` hook updates the the global "legit" `stat struct` size field and copies the `stat struct` into the callee's buffer
5. The imaginary fuzzer restores the state of objdump to its state at snapshot time
6. The imaginary fuzzer copies a new input into harness and updates the input size
7. Our `__xstat()` hook is called once again, and we repeat step 4, this process occurs over and over forever. 

One important thing to keep in mind is that if the snapshot fuzzer is restoring objdump to its snapshot state every fuzzing iteration, we must be careful not to depend on any global mutable memory. The global `stat struct` will be safe since it will be instantiated during the `constructor` however, its size-field will be restored to its original value each fuzzing iteration by the fuzzer's snapshot restore routine. 

We will also need a global, recognizable address to store variable mutable global data like the current input's size. Several snapshot fuzzers have the flexibility to ignore contiguous ranges of memory for restoration purposes. So if we're able to create some contiguous buffers in memory at recognizable addresses, we can have our imaginary fuzzer ignore those ranges for snapshot restorations. So we need to have a place to store the inputs, as well as information about their size. We would then somehow tell the fuzzer about these locations and when it generated a new input, it would copy it into the input location and then update the current input size information.

So now our constructor has an additional job: setup the input location as well as the input size information. We can do this easily with a call to `mmap()` which will allow us to specify an address we want our mapping mapped to with the `MAP_FIXED` flag. We'll also create a `MAX_INPUT_SZ` definition so that we know how much memory to map from the input location. 

Just by themselves, the functions related to mapping memory space for the inputs themselves and their size information looks like this. Notice that we use `MAP_FIXED` and we check the returned address from `mmap()` just to make sure the call didn't succeed but map our memory at a different location:
```c
// Map memory to hold our inputs in memory and information about their size
static void _create_mem_mappings(void) {
    void *result = NULL;

    // Map the page to hold the input size
    result = mmap(
        (void *)(INPUT_SZ_ADDR),
        sizeof(size_t),
        PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS | MAP_FIXED,
        0,
        0
    );
    if ((MAP_FAILED == result) || (result != (void *)INPUT_SZ_ADDR)) {
        printf("Err mapping INPUT_SZ_ADDR, mapped @ %p\n", result);
        exit(-1);
    }

    // Let's actually initialize the value at the input size location as well
    *(size_t *)INPUT_SZ_ADDR = 0;

    // Map the pages to hold the input contents
    result = mmap(
        (void *)(INPUT_ADDR),
        (size_t)(MAX_INPUT_SZ),
        PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS | MAP_FIXED,
        0,
        0
    );
    if ((MAP_FAILED == result) || (result != (void *)INPUT_ADDR)) {
        printf("Err mapping INPUT_ADDR, mapped @ %p\n", result);
        exit(-1);
    }

    // Init the value
    memset((void *)INPUT_ADDR, 0, (size_t)MAX_INPUT_SZ);
}
```

`mmap()` will actually map multiples of whatever the page size is on your system (typically 4096 bytes). So like, when we ask for `sizeof(size_t)` bytes for the mapping, `mmap()` is like: "Hmm, that's just a page dude" and gives us back a whole page from `0x1336000 - 0x1337000` not inclusive on the high-end. 

**Random sidenote, be careful about arithmetic in definitions and macros as I've done here with `MAX_INPUT_SIZE`, it's very easy for the pre-processor to substitute your text for the definition keyword and ruin some order of operations or even overflow a specific primitive type like `int`.**

Hell yeah dude, we now have memory allocated for the fuzzer to store inputs and also information about their size. 
