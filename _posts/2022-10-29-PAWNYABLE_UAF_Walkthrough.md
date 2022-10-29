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

The author does a great job explaining everything you need to know to get started, things like: setting up a debugging environment, CTF-specific tips, modern kernel exploitation mitigations, using QEMU, manipulating images, per-CPU slab caches, etc, so this blogpost will focus exclusively on my experience with the challenge and the way I decided to solve it. I'm going to try and limit redundant information within this blogpost so if you have any questions, it's best to consult PAWNYABLE and the other linked resources. 

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

After that, I started working on a popular Linux kernel pwn challenge called "kernel-rop" from hxpCTF 2020. I followed along and worked alongside the following blogposts from [@\_lkmidas](https://twitter.com/_lkmidas):

- [Learning Kernel Exploitation - Part 1](https://lkmidas.github.io/posts/20210123-linux-kernel-pwn-part-1/)
- [Learning Kernel Exploitation - Part 2](https://lkmidas.github.io/posts/20210128-linux-kernel-pwn-part-2/)
- [Learning Kernel Exploitation - Part 3](https://lkmidas.github.io/posts/20210205-linux-kernel-pwn-part-3/)

This was great because it gave me a chance to reinforce everything I had learned from the PAWNYABLE stack buffer overflow challenge and also I learned a few new things. I also used (https://0x434b.dev/dabbling-with-linux-kernel-exploitation-ctf-challenges-to-learn-the-ropes/) to supplement some of the information. 

As a bonus, I also wrote a version of the exploit that utilized a different technique to elevate privileges: [overwriting `modprobe_path`](https://lkmidas.github.io/posts/20210223-linux-kernel-pwn-modprobe/). 

After all this, I felt like I had a good enough base to get started on the UAF challenge. 

## UAF Challenge: Holstein v3

Some quick vulnerability analysis on the vulnerable driver provided by the author states the problem clearly. 

```c
char *g_buf = NULL;

static int module_open(struct inode *inode, struct file *file)
{
  printk(KERN_INFO "module_open called\n");

  g_buf = kzalloc(BUFFER_SIZE, GFP_KERNEL);
  if (!g_buf) {
    printk(KERN_INFO "kmalloc failed");
    return -ENOMEM;
  }

  return 0;
}
```

When we open the kernel driver, `char *g_buf` gets assigned the result of a call to `kzalloc()`. 

```c
static int module_close(struct inode *inode, struct file *file)
{
  printk(KERN_INFO "module_close called\n");
  kfree(g_buf);
  return 0;
}
```

When we close the kernel driver, `g_buf` is freed. As the author explains, this is a buggy code pattern since we can open multiple handles to the driver from within our program. Something like this can occur.

1. We've done nothing, `g_buf = NULL`
2. We've opened the driver, `g_buf = 0xffff...a0`, and we have `fd1` in our program
3. We've opened the driver a second time, `g_buf = 0xffff...b0` . The original value of `0xffff...a0` has been overwritten. It can no longer be freed and would cause a memory leak (not super important). We now have `fd2` in our program
4. We close `fd1` which calls `kfree()` on `0xffff...b0` and frees the same pointer we have a reference to with `fd2`. 

At this point, via our access to `fd2`, we have a use after free since we can still potentially use a freed reference to `g_buf`. The module also allows us to use the open file descriptor with read and write methods. 

```c
static ssize_t module_read(struct file *file,
                           char __user *buf, size_t count,
                           loff_t *f_pos)
{
  printk(KERN_INFO "module_read called\n");

  if (count > BUFFER_SIZE) {
    printk(KERN_INFO "invalid buffer size\n");
    return -EINVAL;
  }

  if (copy_to_user(buf, g_buf, count)) {
    printk(KERN_INFO "copy_to_user failed\n");
    return -EINVAL;
  }

  return count;
}

static ssize_t module_write(struct file *file,
                            const char __user *buf, size_t count,
                            loff_t *f_pos)
{
  printk(KERN_INFO "module_write called\n");

  if (count > BUFFER_SIZE) {
    printk(KERN_INFO "invalid buffer size\n");
    return -EINVAL;
  }

  if (copy_from_user(g_buf, buf, count)) {
    printk(KERN_INFO "copy_from_user failed\n");
    return -EINVAL;
  }

  return count;
}
```

So with these methods, we are able to read and write to our freed object. This is great for us since we're free to pretty much do anything we want. We are limited somewhat by the object size which is hardcoded in the code to `0x400`. 

At a high-level, UAFs are generally exploited by creating the UAF condition, so we have a reference to a freed object within our control, and then we want to cause the allocation of a *different* object to fill the space that was previously filled by our freed object. 

So if we allocated a `g_buf` of size `0x400` and then freed it, we need to place another object in its place. This new object would then be the target of our reads and writes. 

### KASLR Bypass

The first thing we need to do is bypass KASLR by leaking some address that is a known static offset from the kernel image base. I started searching for objects that have leakable members and again, @ptrYudai came to the rescue with a catalog on [useful Linux Kernel data structures](https://ptr-yudai.hatenablog.com/entry/2020/03/16/165628) for exploitation. This lead me to the [`tty_struct`](https://elixir.bootlin.com/linux/latest/source/include/linux/tty.h#L195) which is allocated on the same slab cache as our `0x400` buffer, the `kmalloc-1024`. The `tty_struct` has a field called `tty_operations` which is a pointer to a function table that is a static offset from the kernel base. So if we can leak the address of `tty_operations` we will have bypassed KASLR. This struct was used by [NCCGROUP for the same purpose in their exploit of CVE-2022-32250](https://research.nccgroup.com/2022/09/01/settlers-of-netlink-exploiting-a-limited-uaf-in-nf_tables-cve-2022-32250/). 

It's important to note that slab cache that we're targeting is per-CPU. Luckily, the VM we're given for the challenge only has one logical core so we don't have to worry about CPU affinity for this exercise. On most systems with more than one core, we would have to worry about influencing one specific CPU's cache. 

So with our `module_read` ability, we will simply:

1. Free `g_buf`
2. Create `dev_tty` structs until one hopefully fills the freed space where `g_buf` used to live
3. Call `module_read` to get a copy of the `g_buf` which is now actually our `dev_tty` and then inspect the value of `tty_struct->tty_operations`. 

Here are some snippets of code related to that from the exploit:

```c
// Leak a tty_struct->ops field which is constant offset from kernel base
uint64_t leak_ops(int fd) {
    if (fd < 0) {
        err("Bad fd given to `leak_ops()`");
    }

    /* tty_struct {
        int magic;      // 4 bytes
        struct kref;    // 4 bytes (single member is an int refcount_t)
        struct device *dev; // 8 bytes
        struct tty_driver *driver; // 8 bytes
        const struct tty_operations *ops; (offset 24 (or 0x18))
        ...
    } */

    // Read first 32 bytes of the structure
    unsigned char *ops_buf = calloc(1, 32);
    if (!ops_buf) {
        err("Failed to allocate ops_buf");
    }

    ssize_t bytes_read = read(fd, ops_buf, 32);
    if (bytes_read != (ssize_t)32) {
        err("Failed to read enough bytes from fd: %d", fd);
    }

    uint64_t ops = *(uint64_t *)&ops_buf[24];
    info("tty_struct->ops: 0x%lx", ops);

    // Solve for kernel base, keep the last 12 bits
    uint64_t test = ops & 0b111111111111;

    // These magic compares are for static offsets on this kernel
    if (test == 0xb40ULL) {
        return ops - 0xc39b40ULL;
    }

    else if (test == 0xc60ULL) {
        return ops - 0xc39c60ULL;
    }

    else {
        err("Got an unexpected tty_struct->ops ptr");
    }
}
```

There's a confusing part about `AND`ing off the lower 12 bits of the leaked value and that's because I kept getting one of two values during multiple runs of the exploit within the same boot. This is probably because there's two kinds of `tty_structs` that can be allocated and they are allocated in pairs. This `if` `else if` block just handles both cases and solves the kernel base for us. So at this point we have bypassed KASLR because we know the base address the kernel is loaded at. 

### RIP Control

Next, we need someway to high-jack execution. Luckily, we can use the same data structure, `tty_struct` as we can write to the object using `module_write` and we can overwrite the pointer value for `tty_struct->ops`. 

[`struct tty_operations` ](https://elixir.bootlin.com/linux/latest/source/include/linux/tty_driver.h#L349) is a table of function pointers, and looks like this:

```C
struct tty_struct * (*lookup)(struct tty_driver *driver,
			struct file *filp, int idx);
	int  (*install)(struct tty_driver *driver, struct tty_struct *tty);
	void (*remove)(struct tty_driver *driver, struct tty_struct *tty);
	int  (*open)(struct tty_struct * tty, struct file * filp);
	void (*close)(struct tty_struct * tty, struct file * filp);
	void (*shutdown)(struct tty_struct *tty);
	void (*cleanup)(struct tty_struct *tty);
	int  (*write)(struct tty_struct * tty,
		      const unsigned char *buf, int count);
	int  (*put_char)(struct tty_struct *tty, unsigned char ch);
	void (*flush_chars)(struct tty_struct *tty);
	unsigned int (*write_room)(struct tty_struct *tty);
	unsigned int (*chars_in_buffer)(struct tty_struct *tty);
	int  (*ioctl)(struct tty_struct *tty,
		    unsigned int cmd, unsigned long arg);
...SNIP...
```

These functions are invoked on the `tty_struct` when certain actions are performed on an instance of a `tty_struct`. For example, when the `tty_struct`'s controlling process exits, several of these functions are called in a row: `close()`, `shutdown()`, and `cleanup()`. 

So our plan, will be to:

1. Create UAF condition
2. Occupy free'd memory with `tty_struct`
3. Read a copy of the `tty_struct` back to us in userland
4. Alter the `tty->ops` value to point to a faked function table that we control
5. Write the new data back to the `tty_struct` which is now corrupted
6. Do something to the `tty_struct` that causes a function we control to be invoked

PAWNYABLE tells us that a popular target is invoking `ioctl()` as the function takes several arguments which are user-controlled. 

```c
int  (*ioctl)(struct tty_struct *tty,
		    unsigned int cmd, unsigned long arg);
```

From userland, we can supply the values for `cmd` and `arg`. This gives us some flexibility. The value we can provide for `cmd` is somewhat limited as an `unsigned int` is only 4 bytes. `arg` gives us a full 8 bytes of control over `RDX`. Since we can control the contents of `RDX` whenever we invoke `ioctl()`, we need to find a gadget to pivot the stack to some code in the kernel heap that we can control. I found such a gadget here:

 ```
0x14fbea: push rdx; xor eax, 0x415b004f; pop rsp; pop rbp; ret;
 ```

We will push a value from `RDX` onto the stack, and then later pop that value into `RSP`. When `ioctl()` returns, we will return to whatever value we called `ioctl()` with in `arg`. So the control flow will go something like:

1. Invoke `ioctl()` on our corrupted `tty_struct`
2. `ioctl()` has been overwritten by a stack-pivot gadget that places the location of our ROP chain into `RSP`
3. `ioctl()` returns execution to our ROP chain

So now we have a new problem, how do we create a fake function table and ROP chain in the kernel heap AND figure out where we stored them?

