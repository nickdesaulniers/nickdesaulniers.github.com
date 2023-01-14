+++
title = "Booting a Custom Linux Kernel in QEMU and Debugging It With GDB"
date = "2018-10-24"
slug = "2018/10/24/booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb"
Categories = []
+++
Typically, when we modify a program, we’d like to run it to verify our changes.
Before booting a compiled Linux kernel image on actual hardware, it can save us
time and potential headache to do a quick boot in a virtual machine like
[QEMU](https://www.qemu.org/)
as a sanity check.  If your kernel boots in QEMU, it’s not a guarantee it will
boot on metal, but it is a quick assurance that the kernel image is not
completely busted.  Since I finally got this working, I figured I’d post the
built up command line arguments (and error messages observed) for future
travelers.  Also, QEMU has more flags than virtually any other binary I’ve ever
seen (other than a google3 binary; shots fired), and simply getting it to print
to the terminal is 3/4 the battle.  If I don’t write it down now, or lose my
shell history, I’ll probably forget how to do this.

TL;DR:
```sh
# one time setup
$ mkinitramfs -o ramdisk.img
$ echo "add-auto-load-safe-path path/to/linux/scripts/gdb/vmlinux-gdb.py" >> ~/.gdbinit

# one time kernel setup
$ cd linux
$ ./scripts/config -e DEBUG_INFO -e GDB_SCRIPTS
$ <make kernel image>

# each debug session run
$ qemu-system-x86_64 \
  -kernel arch/x86_64/boot/bzImage \
  -nographic \
  -append "console=ttyS0 nokaslr" \
  -initrd ramdisk.img \
  -m 512 \
  --enable-kvm \
  -cpu host \
  -s -S &
$ gdb vmlinux
(gdb) target remote :1234
(gdb) hbreak start_kernel
(gdb) c
(gdb) lx-dmesg
```

## Booting in QEMU

We’ll play stupid and see what errors we hit, and how to fix them.  First,
let’s try just our new kernel:

```sh
$ qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage
```

A new window should open, and we should observe some dmesg output, a panic, and
your fans might spin up.  I find this window relatively hard to see, so let’s
get the output (and input) to a terminal:

```sh
$ qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage -nographic
```

This is missing an important flag, but it’s important to see what happens when
we forget it.  It will seem that there’s no output, and QEMU isn’t responding
to `ctrl+c`.  And my fans are spinning again.  Try `ctrl+a`, then `c`, to get a
`(qemu)` prompt.  A simple `q` will exit.

Next, We’re going to pass a kernel command line argument.  The kernel accepts
command line args just like userspace binaries, though usually the bootloader
sets these up.

```sh
$ qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage -nographic -append "console=ttyS0"
...
[    1.144348] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
[    1.144759] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 4.18.0-rc6 #10
[    1.144949] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.10.2-1ubuntu1 04/01/2014
[    1.145269] Call Trace:
[    1.146027]  dump_stack+0x5c/0x7b
[    1.146162]  panic+0xe4/0x252
[    1.146286]  mount_block_root+0x1f1/0x2d6
[    1.146445]  prepare_namespace+0x13b/0x171
[    1.146579]  kernel_init_freeable+0x227/0x254
[    1.146721]  ? rest_init+0xb0/0xb0
[    1.146826]  kernel_init+0xa/0x110
[    1.146931]  ret_from_fork+0x35/0x40
[    1.147412] Kernel Offset: 0x1c200000 from 0xffffffff81000000 (relocation range: 0xffffffff80000000-0xffffffffbfffffff)
[    1.147901] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
(qemu) q
```

Well at least we’re no longer in the dark (remember, `ctrl+a`, `c`, `q` to
exit).  Now we’re panicking because there’s no root filesystem, so there’s no
`init` binary to run.  Now we could create a custom filesystem image with the
bare bones (definitely a post for another day), but creating a ramdisk is the
most straightforward way, IMO.  Ok, let’s create the ramdisk,
then add it to QEMU’s parameters:

```sh
$ mkinitramfs -o ramdisk.img
$ qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage -nographic -append "console=ttyS0" -initrd ramdisk.img
```

Unfortunately, we’ll (likely) hit the same panic and the panic doesn’t provide
enough information, but the default maximum memory QEMU will use is too
limited.  `-m 512` will give our virtual machine enough memory to boot and get
a busybox based shell prompt:

```sh
$ qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage -nographic -append "console=ttyS0" -initrd ramdisk.img -m 512
...
(initramfs) cat /proc/version
Linux version 4.19.0-rc7+ (nick@nick-Blade-Stealth) (clang version 8.0.0 (https://git.llvm.org/git/clang.git/ 60c8c0cc0786c7f6b8dc5c1e3acd7ec98f0a7b6d) (https://git.llvm.org/git/llvm.git/ 64c3a57bec9dbe3762fa1d80055ba850d6658f5b)) #18 SMP Wed Oct 24 19:29:53 PDT 2018
```

Enabling kvm seems to help with those fans spinning up:

```sh
$ qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage -nographic -append "console=ttyS0" -initrd ramdisk.img -m 512 --enable-kvm
```

Finally, we might be seeing a warning when we start QEMU:

```
qemu-system-x86_64: warning: host doesn't support requested feature: CPUID.80000001H:ECX.svm [bit 2]
```

Just need to add `-cpu host` to our invocation of QEMU.  It can be helpful when
debugging to disable
[KASLR](https://lwn.net/Articles/569635/)
via `nokaslr` in the appended kernel command line parameters, or via
CONFIG_RANDOMIZE_BASE not being set in our kernel configs.

We can add `-s` to start a gdbserver on port 1234, and `-S` to pause the kernel
until we continue in gdb.

## Attaching GDB to QEMU
Now that we can boot this kernel image in QEMU, let’s attach gdb to it.

```sh
$ gdb vmlinux
```

If you see this on your first run:
```
warning: File "/home/nick/linux/scripts/gdb/vmlinux-gdb.py" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
	add-auto-load-safe-path /path/to/linux/scripts/gdb/vmlinux-gdb.py
line to your configuration file "/home/<username>/.gdbinit".
```
Then you can do this one time fix in order to load the gdb scripts each run:
```sh
$ cd linux
$ echo "add-auto-load-safe-path `pwd`/scripts/gdb/vmlinux-gdb.py" >> ~/.gdbinit
```

Now that QEMU is listening on port 1234 (via `-s`), let’s connect to it, and
set a break point early on in the kernel’s initialization.  Note the the use of
`hbreak` (I lost a lot of time just using `b start_kernel`, only for the
kernel to continue booting past that function).

```
(gdb) target remote :1234
Remote debugging using :1234
0x000000000000fff0 in cpu_hw_events ()

(gdb) hbreak start_kernel
Hardware assisted breakpoint 2 at 0xffffffff82904a1d: file init/main.c, line 536.
(gdb) c
Continuing.

Breakpoint 2, start_kernel () at init/main.c:536
536		set_task_stack_end_magic(&init_task);
(gdb) bt
#0  start_kernel () at init/main.c:536
#1  0xffffffff810000d4 in secondary_startup_64 () at arch/x86/kernel/head_64.S:243
#2  0x0000000000000000 in ?? ()
```

We can start/resume the kernel with `c`, and pause it with `ctrl+c`.  The gdb
scripts provided by the kernel via CONFIG_GDB_SCRIPTS can be viewed with
`apropos lx`. `lx-dmesg` is incredibly handy for viewing the kernel dmesg
buffer, particularly in the case of a kernel panic before the serial driver has
been brought up (in which case there’s output from QEMU to stdout, which is
just as much fun as debugging graphics code (ie. black screen)).

```
(gdb) apropos lx
function lx_current -- Return current task
function lx_module -- Find module by name and return the module variable
function lx_per_cpu -- Return per-cpu variable
function lx_task_by_pid -- Find Linux task by PID and return the task_struct variable
function lx_thread_info -- Calculate Linux thread_info from task variable
function lx_thread_info_by_pid -- Calculate Linux thread_info from task variable found by pid
lx-cmdline --  Report the Linux Commandline used in the current kernel
lx-cpus -- List CPU status arrays
lx-dmesg -- Print Linux kernel log buffer
lx-fdtdump -- Output Flattened Device Tree header and dump FDT blob to the filename
lx-iomem -- Identify the IO memory resource locations defined by the kernel
lx-ioports -- Identify the IO port resource locations defined by the kernel
lx-list-check -- Verify a list consistency
lx-lsmod -- List currently loaded modules
lx-mounts -- Report the VFS mounts of the current process namespace
lx-ps -- Dump Linux tasks
lx-symbols -- (Re-)load symbols of Linux kernel and currently loaded modules
lx-version --  Report the Linux Version of the current kernel

(gdb) lx-version
Linux version 4.19.0-rc7+ (nick@nick-Blade-Stealth) (clang version 8.0.0
(https://git.llvm.org/git/clang.git/ 60c8c0cc0786c7f6b8dc5c1e3acd7ec98f0a7b6d)
(https://git.llvm.org/git/llvm.git/ 64c3a57bec9dbe3762fa1d80055ba850d6658f5b))
#18 SMP Wed Oct 24 19:29:53 PDT 2018
```

## Where to Go From Here

Maybe try cross compiling a kernel (you'll need a cross
compiler/assembler/linker/debugger and likely a different `console=ttyXXX`
kernel command line parameter), building your own root filesystem with
[buildroot](https://buildroot.org/),
or exploring the rest of QEMU's command line options.
