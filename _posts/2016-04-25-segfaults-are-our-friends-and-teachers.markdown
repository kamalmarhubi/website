---
title: Segfaults are our friends and teachers
date: 2016-04-25T16:47:14-04:00
---

This afternoon, I got a segmentation fault in a Rust program, and was confused. This is Rust, I shouldn't get segfaults! I quickly checked the couple of places I used `unsafe`, and they were either calling C functions (and checking the errors), or telling the compiler not to initialize some memory because I was going to write into it unconditionally before reading from it.

So, everything should be fine, right?

Evidently not:

~~~
$ cargo build --quiet
$ target/debug/kern-value 1
Segmentation fault
~~~


## ok, but what is a segfault?

If you've written some C, you've almost certainly seen a segmentation fault at some point. I spent a long time thinking of it as a ‘thing that happens when you use pointers wrong in C’. That's mostly true, but a segmentation fault actually has a very specific meaning. You get one when a process tries to access memory in a way that it's not allowed to, or accessing invalid memory.

Here's an example of a really simple program that will segfault:

~~~
$ cat segfault.c
#include <stdio.h>

int main(void) {
    char *uninitialized;
    printf("%c", *uninitialized);

    return 0;
}
$ gcc segfault.c
$ ./a.out
Segmentation fault
~~~

This program segfaults because the entire stack is set to `0` at program start. This includes the `uninitialized` pointer, which will be a null pointer, pointing to `0x0`. When the program tries to print the value it points to, it attempts to read from the null address. That address is invalid, and that's why we get a segmentation fault.

Segmentation faults are raised by the memory management unit (MMU), which is a piece of hardware! Sections of memory can have different access permissions on them: read, write, and execute. Operating systems use this to isolate processes, and to protect kernel memory from being written to by user code.

If a process tries to write to read-only memory, say, or execute non-executable memory, then the MMU hardware tells the kernel there was a segmentation fault. The kernel delivers the news to the process as a signal: `SIGSEGV`. By default, that terminates the process.


## how we got the segfault

Some context: I'm playing around with reading and writing to pipes with large buffers. I'd previously been setting the pipe size to 1 MB, which is the maximum an unprivileged process can set it to on my machine: 

~~~
$ cat /proc/sys/fs/pipe-max-size
1048576
~~~

My code had a constant `PIPE_SIZE` for the size of the pipe. I was *also* using it as the size of a stack-allocated buffer. For my experiments, I'd just changed that constant from 1 MB to 16 MB to test bigger pipes when running as root.

Realising this this set off something in the back of my mind: I vaguely remembered that stacks on Linux default to 8MB. Could the segfault be from going past the stack limit?

I switched to C to verify it:

~~~
$ cat -n test-stack.c
     1  #include <stdio.h>
     2
     3  #define BUF_SIZE (16*1024*1024)
     4
     5  int main(void) {
     6      char buf[BUF_SIZE];
     7      printf("%lu\n", sizeof(buf));
     8
     9      return 0;
    10  }
$ gcc -Wall -Wextra -Werror test-stack.c
$ ./a.out
Segmentation fault
~~~

Aha! A pretty minimal C program that has the same result. This program allocates a 16 MB array on the stack, and then prints its size. Except it segfaults instead. With the `BUF_SIZE` set to 8 KB less than 8 MB, everything works. With it set to 8 MB, it segfaults.[^why-8-mb] Hypothesis confirmed![^why-rust-ok]

[^why-8-mb]:
    Curiously, I found that if I had a buffer size of even 1 byte over (8 MB - 8 KB), I still got the segfault. I'm not yet sure what's going on there!

[^why-rust-ok]:
    Rust really likes to tout its memory safety. This seems like it should mean no segmentation faults, which is why I was confused at getting one. There's [a very specific set of behaviors][rust-ub] that aren't allowed in Rust programs. The Rust program got a segmentation fault because it attempted to write to inaccessible memory, but only through the stack pointer. None of the undefined behaviors disallow this, which I think is why it's ok for this Rust program to segfault.


[rust-ub]: http://doc.rust-lang.org/reference.html#behavior-considered-undefined


## seeing which instruction failed

I wanted to get an idea of exactly *when* the segfault happened, so I ran it under gdb:

~~~
$ gcc -g -Wall -Wextra -Werror test-stack.c
$ gdb -silent a.out
Reading symbols from a.out...done.
(gdb) run
Starting program: /tmp/a.out

Program received signal SIGSEGV, Segmentation fault.
0x0000000000400520 in main () at test-stack.c:7
7               printf("%lu\n", sizeof(buf));
(gdb) backtrace
#0  0x0000000000400520 in main () at test-stack.c:7
(gdb) disassemble
Dump of assembler code for function main:
   0x0000000000400506 <+0>:     push   %rbp
   0x0000000000400507 <+1>:     mov    %rsp,%rbp
   0x000000000040050a <+4>:     sub    $0x1000000,%rsp
   0x0000000000400511 <+11>:    mov    $0x1000000,%esi
   0x0000000000400516 <+16>:    mov    $0x4005b4,%edi
   0x000000000040051b <+21>:    mov    $0x0,%eax
=> 0x0000000000400520 <+26>:    callq  0x4003e0 <printf@plt>
   0x0000000000400525 <+31>:    mov    $0x0,%eax
   0x000000000040052a <+36>:    leaveq
   0x000000000040052b <+37>:    retq
End of assembler dump.
~~~

So the failure was on the call to `printf`. `0x1000000` is 16 MB, the size of the buffer. The `sub    $0x1000000,%rsp` instruction is modifying the stack pointer register `%rsp` to make space for the buffer. When we get to the `callq` instruction, things exploded!

The `callq` instruction on x86 first pushes the return address onto the stack, then jumps to the called function. The return address is the current value instruction pointer register `%ip`, and is where the `ret` instruction in `printf` will jump back to. The segfault happens at this point: the CPU attempts to store the return address somewhere outside of the region of memory that's set aside for the stack, and boom!


## luck and guard pages

I think we're actually kind of lucky to get a segfault. It's quite possible the bit of memory that's a few megabytes past our stack *was* writeable by our process. We'd then go happily corrupting whatever memory was there and probably all kinds of bad things would happen. Segmentation faults are actually our friends!

Modern compilers have stack protection features that help detect overflows. I'm *very* hazy on the details, but I think they include setting some pages of memory just after the stack to be read only. These are called guard pages. Attempts to write there would result in a segmentation fault. This helps catch stack overflow attempts that involve marching off the end of the stack. However, writing 8 MB after the end of the stack is beyond the guard pages, so we *could* have ended up in writeable memory.


## setting stack size limits in the shell: ulimit

I took this as an opportunity to Learn. I wanted to make this program run without changing its source. At first I thought I could set the stack size with compiler options, but some searching revealed this wasn't true on Linux. It turns out you can set it with the `ulimit` shell built-in[^why-builtin]:

~~~
$ ulimit --soft --stack-size 32768
$ ./a.out
16777216
~~~

Success, and without changing the source *or* the binary! The `--stack-size` flag says we're setting the stack size to 32768 KB or 32 MB. The `--soft` flag says to set the soft limit. I'll just paste from my `man ulimit` on my system to explain the difference between hard and soft limits—the `-S` flag is short for `--soft`:

> A hard limit can only be decreased. Once it is set it cannot be increased; a soft limit may be increased up to the value of the hard limit. If neither `-H` nor `-S` is specified, both the soft and hard limits are updated when assigning a new limit value, and the soft limit is used when reporting the current value.


[^why-builtin]:
    `ulimit` must be a shell built-in and not an executable file because it sets a property of the shell's process. Other fun examples include setting the working directory (`cd`), and setting environment variables (`set -x` in bash). These were some fun things I found out when [building a shell][shell-workshop], which is a totally worthwhile and fun exercise!

[shell-workshop]: https://github.com/kamalmarhubi/shell-workshop


## setting stack size limits from inside your program: setrlimit(2) 

If you use a lot of stack, it's not great to force the user to run `ulimit` before running your program. Instead, you can set it yourself using the [`setrlimit(2)`][setrlimit] system call. Here's proof it works:

[setrlimit]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/setrlimit.html

~~~
$ cat -n test-stack.c
     1  #include <errno.h>
     2  #include <stdio.h>
     3  #include <sys/resource.h>
     4
     5  #define BUF_SIZE (16*1024*1024)
     6
     7  void run(void) {
     8      char buf[BUF_SIZE];
     9      printf("%lu\n", sizeof(buf));
    10  }
    11
    12  int main(void) {
    13      struct rlimit stack_limit = {
    14          .rlim_cur = 2 * BUF_SIZE,
    15          .rlim_max = RLIM_INFINITY,
    16      };
    17
    18      if (setrlimit(RLIMIT_STACK, &stack_limit)) {
    19          perror("setrlimit failed");
    20          return 1;
    21      }
    22
    23      run();
    24
    25      return 0;
    26  }
$ gcc -Wall -Wextra -Werror test-stack.c
$ ulimit -Ss 8192
$ ./a.out
16777216
~~~

A fun note: we had to put the buffer in a separate function. Otherwise the stack pointer would be adjusted before the call to `setrlimit`, and then *that* call would result in a segfault! This way the stack adjustment happens after calling `setrlimit`, and everything works out.

Allocating lots of stack space in main isn't a great idea anyway, since you limit what's available for the rest of your program. A static buffer would probably be better for this program; for others a heap buffer would work better.


## diving into the depths for fun and learning

I had a fun hour or two investigating this. It's not always possible to take the time in the moment, but it's really rewarding when you can. I only vaguely knew about the `ulimit` command, and didn't know anything at all about `setrlimit(2)`. Since I've been programming in languages that output native code a bunch lately, I've also wanted to learn more about object file formats, linkers, in-process memory layout, and more. Learning a bit more about stack guard pages is a great step in that direction!

<br />

---

<small>*Thanks to Julia Evans for feedback on this post.*</small>
