+++
title = "Intro to Debugging X86-64 Assembly"
date = "2016-01-20"
slug = "2016/01/20/debugging-x86-64-assembly-with-lldb-and-dtrace"
Categories = []
+++
I'm hacking on an assembly project, and wanted to document some of the tricks I
was using for figuring out what was going on.  This post might seem a little
basic for folks who spend all day heads down in gdb or who do this stuff
professionally, but I just wanted to share a quick intro to some tools that
others may find useful.
([oh god, I'm doing it](https://pchiusano.github.io/2014-10-11/defensive-writing.html))

If your coming from gdb to lldb, there's a few differences in commands.  LLDB
has
[great documentation](http://lldb.llvm.org/lldb-gdb.html)
on some of the differences. Everything in this post about LLDB is pretty much
there.

The bread and butter commands when working with gdb or lldb are:

* r (run the program)
* s (step in)
* n (step over)
* finish (step out)
* c (continue)
* q (quit the program)

You can hit enter if you want to run the last command again, which is really
useful if you want to keep stepping over statements repeatedly.

I've been using LLDB on OSX.  Let's say I want to debug a program I can build,
but is crashing or something:
```sh
$ sudo lldb ./asmttpd web_root
```
Setting a breakpoint on jump to label:
```sh
(lldb) b sys_write
Breakpoint 3: where = asmttpd`sys_write, address = 0x00000000000029ae
```
Running the program until breakpoint hit:
```sh
(lldb) r
Process 32236 launched: './asmttpd' (x86_64)
Process 32236 stopped
* thread #1: tid = 0xe69b9, 0x00000000000029ae asmttpd`sys_write, queue = 'com.apple.main-thread', stop reason = breakpoint 3.1
    frame #0: 0x00000000000029ae asmttpd`sys_write
asmttpd`sys_write:
->  0x29ae <+0>: pushq  %rdi
    0x29af <+1>: pushq  %rsi
    0x29b0 <+2>: pushq  %rdx
    0x29b1 <+3>: pushq  %r10
```
Seeing more of the current stack frame:
```sh
(lldb) d
asmttpd`sys_write:
->  0x29ae <+0>:  pushq  %rdi
    0x29af <+1>:  pushq  %rsi
    0x29b0 <+2>:  pushq  %rdx
    0x29b1 <+3>:  pushq  %r10
    0x29b3 <+5>:  pushq  %r8
    0x29b5 <+7>:  pushq  %r9
    0x29b7 <+9>:  pushq  %rbx
    0x29b8 <+10>: pushq  %rcx
    0x29b9 <+11>: movq   %rsi, %rdx
    0x29bc <+14>: movq   %rdi, %rsi
    0x29bf <+17>: movq   $0x1, %rdi
    0x29c6 <+24>: movq   $0x2000004, %rax
    0x29cd <+31>: syscall
    0x29cf <+33>: popq   %rcx
    0x29d0 <+34>: popq   %rbx
    0x29d1 <+35>: popq   %r9
    0x29d3 <+37>: popq   %r8
    0x29 <+39>: popq   %r10
    0x29d7 <+41>: popq   %rdx
    0x29d8 <+42>: popq   %rsi
    0x29d9 <+43>: popq   %rdi
    0x29da <+44>: retq
```
Getting a back trace (call stack):
```sh
(lldb) bt
* thread #1: tid = 0xe69b9, 0x00000000000029ae asmttpd`sys_write, queue = 'com.apple.main-thread', stop reason = breakpoint 3.1
  * frame #0: 0x00000000000029ae asmttpd`sys_write
    frame #1: 0x00000000000021b6 asmttpd`print_line + 16
    frame #2: 0x0000000000002ab3 asmttpd`start + 35
    frame #3: 0x00007fff9900c5ad libdyld.dylib`start + 1
    frame #4: 0x00007fff9900c5ad libdyld.dylib`start + 1
```
peeking at the upper stack frame:
```sh
(lldb) up
frame #1: 0x00000000000021b6 asmttpd`print_line + 16
asmttpd`print_line:
    0x21b6 <+16>: movabsq $0x30cb, %rdi
    0x21c0 <+26>: movq   $0x1, %rsi
    0x21c7 <+33>: callq  0x29ae                    ; sys_write
    0x21cc <+38>: popq   %rcx
```
back down to the breakpoint-halted stack frame:
```sh
(lldb) down
frame #0: 0x00000000000029ae asmttpd`sys_write
asmttpd`sys_write:
->  0x29ae <+0>: pushq  %rdi
    0x29af <+1>: pushq  %rsi
    0x29b0 <+2>: pushq  %rdx
    0x29b1 <+3>: pushq  %r10
```
dumping the values of registers:
```sh
(lldb) register read
General Purpose Registers:
       rax = 0x0000000000002a90  asmttpd`start
       rbx = 0x0000000000000000
       rcx = 0x00007fff5fbffaf8
       rdx = 0x00007fff5fbffa40
       rdi = 0x00000000000030cc  start_text
       rsi = 0x000000000000000f
       rbp = 0x00007fff5fbffa18
       rsp = 0x00007fff5fbff9b8
        r8 = 0x0000000000000000
        r9 = 0x00007fff7b1670c8  atexit_mutex + 24
       r10 = 0x00000000ffffffff
       r11 = 0xffffffff00000000
       r12 = 0x0000000000000000
       r13 = 0x0000000000000000
       r14 = 0x0000000000000000
       r15 = 0x0000000000000000
       rip = 0x00000000000029ae  asmttpd`sys_write
    rflags = 0x0000000000000246
        cs = 0x000000000000002b
        fs = 0x0000000000000000
        gs = 0x0000000000000000
```
read just one register:
```sh
(lldb) register read rdi
     rdi = 0x00000000000030cc  start_text
```
When you're trying to figure out what system calls are made by some C code,
using dtruss is very helpful.  dtruss is available on OSX and seems to be some
kind of wrapper around DTrace.
```sh
$ cat sleep.c
#include <time.h>
int main () {
  struct timespec rqtp = {
    2,
    0
  };

  nanosleep(&rqtp, NULL);
}

$ clang sleep.c

$ sudo dtruss ./a.out
...all kinds of fun stuff
__semwait_signal(0xB03, 0x0, 0x1)    = -1 Err#60
```
If you compile with `-g` to emit debug symbols, you can use lldb's disassemble
command to get the equivalent assembly:
```sh
$ clang sleep.c -g
$ lldb a.out
(lldb) target create "a.out"
Current executable set to 'a.out' (x86_64).
(lldb) b main
Breakpoint 1: where = a.out`main + 16 at sleep.c:3, address = 0x0000000100000f40
(lldb) r
Process 33213 launched: '/Users/Nicholas/code/assembly/asmttpd/a.out' (x86_64)
Process 33213 stopped
* thread #1: tid = 0xeca04, 0x0000000100000f40 a.out`main + 16 at sleep.c:3, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100000f40 a.out`main + 16 at sleep.c:3
   1    #include <time.h>
   2    int main () {
-> 3      struct timespec rqtp = {
   4        2,
   5        0
   6      };
   7
(lldb) disassemble
a.out`main:
    0x100000f30 <+0>:  pushq  %rbp
    0x100000f31 <+1>:  movq   %rsp, %rbp
    0x100000f34 <+4>:  subq   $0x20, %rsp
    0x100000f38 <+8>:  leaq   -0x10(%rbp), %rdi
    0x100000f3c <+12>: xorl   %eax, %eax
    0x100000f3e <+14>: movl   %eax, %esi
->  0x100000f40 <+16>: movq   0x49(%rip), %rcx
    0x100000f47 <+23>: movq   %rcx, -0x10(%rbp)
    0x100000f4b <+27>: movq   0x46(%rip), %rcx
    0x100000f52 <+34>: movq   %rcx, -0x8(%rbp)
    0x100000f56 <+38>: callq  0x100000f68               ; symbol stub for: nanosleep
    0x100000f5b <+43>: xorl   %edx, %edx
    0x100000f5d <+45>: movl   %eax, -0x14(%rbp)
    0x100000f60 <+48>: movl   %edx, %eax
    0x100000f62 <+50>: addq   $0x20, %rsp
    0x100000f66 <+54>: popq   %rbp
    0x100000f67 <+55>: retq
```

Anyways, I've been learning some interesting things about OSX that I'll be
sharing soon. If you'd like to learn more about x86-64 assembly programming,
you should read my other posts about
[writing x86-64](/blog/2014/04/18/lets-write-some-x86-64/)
and a toy
[JIT for Brainfuck](/blog/2015/05/25/interpreter-compiler-jit/)
([the creator of Brainfuck liked it](https://www.reddit.com/r/programming/comments/377ov9/interpreter_compiler_jit/crkkrz4)).

I should also do a post on
[Mozilla's rr](http://rr-project.org/),
because it can do amazing things like step backwards.  Another day...

