# App - Remote debug


## Objectives

* Use `strace` and `ltrace` to diagnose program issues.

* Use `gdbserver` and a cross-debugger to remotely debug an embedded application.


## Setup

Go to the `$HOME/embedded-linux-qemu-labs/debugging/` directory.
Create an `nfsroot` directory there.

```console
$ LAB_PATH="$HOME/embedded-linux-qemu-labs/debugging"
$ cd $LAB_PATH
$ mkdir nfsroot
```


## Debugging setup

Because of issues in `gdb` and `ltrace` in the *uClibc* version that we are using in our toolchain, we'r going to use a different toolchain in this lab, based on `glibc`.

As `glibc` has more complete features than lighter libraries, it looks like a good idea to perform your application debugging work with a `glibc` toolchain first, and switch to lighter libraries once your application and software stack are production ready.

As done in the *Buildroot* lab, clone once again the *Buildroot* *git* repository, and checkout the tag corresponding to the `2022.02` release (*Long Term Support*), which we have tested for this lab.<br/>
You could re-use the installation already performed previously.

Then, in the `menuconfig` interface, configure the target architecture as done previously, but configure the *toolchain* and *target packages* differently.


In `Toolchain`:

* `Toolchain type` = `External toolchain`.

* `Toolchain` = `Bootlin toolchains`.

* `Toolchain origin` = `Toolchain to be downloaded and installed`.

* `Bootlin toolchain variant` = `armv7-eabihf glibc stable 2021.11-1`.

* Enable `Copy gdb server to the Target`.


In `Target packages` &rarr; `Debugging, profiling and benchmark`:

* Enable `ltrace`.

* Enable `strace`.


Now, `<Save>` the `.config`, backup it, and build your *root* filesystem.

```console
$ cd "$LAB_PATH/../buildroot/buildroot/"
$ make menuconfig
$ cp .config "$LAB_PATH/buildroot-debugging.config"
$ export MAKEFLAGS=-j$(nproc)
$ make clean
$ make
```

Go back to the lab directory and extract the `buildroot/output/images/rootfs.tar` archive into the `nfsroot` directory.
Add this directory to the `/etc/exports` file and run `sudo exportfs -r`. You can of course remap the *symlink* as done previously.<br/>
Boot your ARM board over NFS on this new filesystem, using the same kernel as before.

```console
$ cd "$LAB_PATH/nfsroot/"
$ tar xfv "$LAB_PATH/../buildroot/buildroot/output/images/rootfs.tar"
$ sudo rm -f /srv/nfs
$ sudo ln -snv "$LAB_PATH/nfsroot/" /srv/nfs
'/srv/nfs' -> '/home/me/embedded-linux-qemu-labs/debugging/nfsroot/'
$ sudo chown -R tftp:tftp /srv/nfs
$ sudo exportfs -ar
$ sudo systemctl restart nfs-kernel-server
```


## Using `strace`

The `strace` allows to *trace* all the system calls made by a process: opening, reading and writing files,
starting other processes, accessing time, etc.
When something goes wrong in your application, `strace` is an invaluable tool to see what it actually does, even when you don’t have the source code.

Update the `PATH`.<br/>
With your cross-compiling toolchain compile the `data/vista-emulator.c` program, copy the
resulting binary to the `/root` directory of the *root* filesystem and then strip it.

```console
$ export PATH="$HOME/embedded-linux-qemu-labs/buildroot/buildroot/output/host/bin:$PATH"
$ cd "$LAB_PATH/data/"
$ arm-linux-gcc -o vista-emulator vista-emulator.c
$ cp vista-emulator "$LAB_PATH/nfsroot/root/"
```

Back to target system, try to run the `/root/vista-emulator` program. It should hang indefinitely!<br/>
Interrupt this program by hitting ++ctrl+c++.

Now, running this program again through the `strace` command and understand why it hangs.
You can guess it without reading the source code!

```console title="QEMU - Buildroot" hl_lines="4"
# strace /root/vista-emulator
execve("/root/vista-emulator", ["/root/vista-emulator"], 0x7ed7ce60 /* 10 vars */) = 0
    ...
openat(AT_FDCWD, "/etc/vista.key", O_RDONLY) = -1 ENOENT (No such file or directory)
clock_nanosleep(CLOCK_REALTIME, 0, {tv_sec=1, tv_nsec=0}, 0x7e926c70) = 0
openat(AT_FDCWD, "/etc/vista.key", O_RDONLY) = -1 ENOENT (No such file or directory)
clock_nanosleep(CLOCK_REALTIME, 0, {tv_sec=1, tv_nsec=0}, 0x7e926c70) = 0
openat(AT_FDCWD, "/etc/vista.key", O_RDONLY) = -1 ENOENT (No such file or directory)
clock_nanosleep(CLOCK_REALTIME, 0, {tv_sec=1, tv_nsec=0}, 0x7e926c70) = 0
    ...
```

Interrupt again by hitting ++ctrl+c++.

Now add what the program was waiting for, and see your program proceed to another bug, failing with a mighty *segmentation fault*.

```console title="QEMU - Buildroot" hl_lines="5 12 15"
# touch /etc/vista.key
# strace /root/vista-emulator
execve("/root/vista-emulator", ["/root/vista-emulator"], 0x7efe8e60 /* 10 vars */) = 0
    ...
openat(AT_FDCWD, "/etc/vista.key", O_RDONLY) = 3
close(3)                                = 0
mmap2(NULL, 659456, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x1b100000
mmap2(NULL, 659456, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x1b05f000
mmap2(NULL, 659456, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x1afbe000
    ...
mmap2(NULL, 659456, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7eed9000
mmap2(NULL, 659456, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = -1 ENOMEM (Cannot allocate memory)
brk(0x4e3000)                           = 0x443000
mmap2(NULL, 1048576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = -1 ENOMEM (Cannot allocate memory)
--- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_MAPERR, si_addr=NULL} ---
+++ killed by SIGSEGV +++
Segmentation fault
```


## Using `ltrace`

Now run the program through `ltrace`. You should see what the program does: it tries to consume as much system memory as it can! You can also notice that `ltrace` makes the execution much slower than with `strace`.

```console title="QEMU - Buildroot" hl_lines="13"
# ltrace /root/vista-emulator
__libc_start_main([ "/root/vista-emulator" ] <unfinished ...>
fopen("/etc/vista.key", "r")                     = 0x432190
fclose(0x432190)                                 = 0
malloc(655360)                                   = 0x76d3e008
strstr("", "Mickey Mouse")                       = nil
malloc(655360)                                   = 0x76c9d008
strstr("", "Mickey Mouse")                       = nil
malloc(655360)                                   = 0x76bfc008
    ...
malloc(655360)                                   = 0x7ef56008
strstr("", "Mickey Mouse")                       = nil
malloc(655360)                                   = nil
strstr(nil, "Mickey Mouse" <no return ...>
--- SIGSEGV (Segmentation fault) ---
+++ killed by SIGSEGV +++
```

Also run the program through `ltrace -c`, to see what function call statistics this utility can
provide.

```console title="QEMU - Buildroot"
# ltrace -c /root/vista-emulator
% time     seconds  usecs/call     calls      function
------ ----------- ----------- --------- --------------------
 55.92   39.256634    39256634         1 __libc_start_main
 25.19   17.688146        5484      3225 malloc
 18.86   13.242600        4106      3225 strstr
  0.02    0.013549       13549         1 fopen
  0.01    0.006764        6764         1 fclose
------ ----------- ----------- --------- --------------------
100.00   70.207693                  6453 total
```

It’s also interesting to run the program again with `strace` (or, refer to the previous paragraph).
You can see that memory allocations translate into `mmap()` system calls. That’s how you can recognize them when you’re using `strace`.


## Using `gdbserver`

We are now going to use gdbserver to understand why the program segfaults.

Compile `vista-emulator.c` again with the `-g` option to include debugging symbols.
This time, just keep it on your workstation, as you already have the version without debugging symbols
on your target.

```console
$ cd "$LAB_PATH/data/"
$ arm-linux-gcc -g -o vista-emulator vista-emulator.c
```

Then, on the *target* side, run `vista-emulator` under `gdbserver`, which will listen on a TCP
port for a connection from `gdb`, and will control the execution of `vista-emulator` according to
the `gdb` commands.

```console title="QEMU - Buildroot"
# gdbserver localhost:2345 /root/vista-emulator
Process /root/vista-emulator created; pid = 160
Listening on port 2345
```

On the *host* side, run `arm-linux-gdb` (also found in your toolchain).
This way, `gdb` starts and loads the debugging information from the `vista-emulator` binary that was
compiled with `-g`.<br/>
We need to tell where to find our libraries, since they are not present in the default `/lib/`
and `/usr/lib/` directories on your workstation.
This is done by setting the `sysroot` variable of `gdb`. We do this directly by the command line (`-ex` argument) for convenience.

```console
$ cd "$LAB_PATH/data/"
$ BUILDROOT_STAGING="$LAB_PATH/../buildroot/buildroot/output/staging"
$ arm-linux-gdb  -ex "set sysroot $BUILDROOT_STAGING"  vista-emulator
    ...
Reading symbols from ./vista-emulator...
```

Tell `gdb` to connect to the remote *target* system, and it will break at the `_start()` library function.

``` hl_lines="5"
(gdb) target remote 10.0.2.69:2345
Remote debugging using 10.0.2.69:2345
Reading symbols from /home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/staging/lib/ld-linux-armhf.so.3...
0x76fc88c0 in _start ()
   from /home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/staging/lib/ld-linux-armhf.so.3
```

Then, you can use `gdb` as usual to set breakpoints, look at the source code, run the application step by
step, etc.
Graphical versions of `gdb` (such as `ddd`) can also be used in the same way.<br/>
In our case, we’ll just start the program and wait for it to hit the *segmentation fault*.

``` hl_lines="5"
(gdb) continue
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x76edbbd0 in strchr () from /home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/staging/lib/libc.so.6
```

You could then ask for a *backtrace* to see where this happened.

``` hl_lines="2-3"
(gdb) backtrace
#0  0x76edbbd0 in strchr ()
   from /home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/staging/lib/libc.so.6
#1  0x76edd11c in strstr ()
   from /home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/staging/lib/libc.so.6
#2  0x00400784 in log_activity (buffer=0x0) at vista-emulator.c:39
#3  0x00400808 in init_resources () at vista-emulator.c:52
#4  0x0040085c in main () at vista-emulator.c:72
```

This will tell you that the *segmentation fault* occurred in a function of the C library, called by
our program. This should help you in finding the bug in our application.

Finally, `quit` *GDB* to close the debugging session.

```
(gdb) set confirm off
(gdb) quit
```


## Post mortem analysis

Configure your shell on the *target* to get a *core file* dumped when you run `vista-emulator` again.

```console title="QEMU - Buildroot" hl_lines="4 6"
# ulimit -c unlimited
# echo "vista-emulator.core" > /proc/sys/kernel/core_pattern
# /root/vista-emulator
Segmentation fault (core dumped)
# du -h vista-emulator.core
12.7M   vista-emulator.core
# chmod a+r vista-emulator.core
```

Once you have such a file, inspect it with `arm-linux-gdb` on the *host*, set the `sysroot` setting, and then generate a *backtrace* to see where the program crashed.<br/>
This way, you can have information about the crash without running the program through the debugger.

Because the executable on the *target* does not contain debug information (it was compiled without the `-g` option), and because the core dump holds the *absolute paths* of the *target*, we have to emulate the state of the *target sysroot* somehow.<br/>
One way works by copying the *debug* executable and the *core dump* to the corresponding places within the `staging` folder by *Buildroot*.<br/>
With `set sysroot`, `gdb` sees the debug files as if they were there on the fake *target*.

``` title="Folder structure excerpts to analyze our core dump"
[TARGET]
/                            <<<  host NFS: $LAB_PATH/nfsroot/
├── lib/
|   └── libc.so.6
└── root/
    ├── vista-emulator       <<<  without debug symbols
    └── vista-emulator.core

[HOST]
buildroot/output/staging/    <<<  gdb: sysroot
├── lib/
|   └── libc.so.6
└── root/
    ├── vista-emulator       <<<  with debug symbols, copied from: $LAB_PATH/data/
    └── vista-emulator.core  <<<  copied from: $LAB_PATH/nsfroot/root/
```

Here's the sequence of operations, switching between the shell and `gdb`:

```console
$ cd $BUILDROOT_STAGING
$ cp "$LAB_PATH/data/vista-emulator" root/
$ cp "$LAB_PATH/nfsroot/root/vista-emulator.core" root/
$ arm-linux-gdb  -ex "set sysroot $BUILDROOT_STAGING"
    ...
```

``` hl_lines="1 3 9 17 18"
(gdb) file vista-emulator
Reading symbols from vista-emulator...
(gdb) core-file vista-emulator.core
[New LWP 122]
Core was generated by `/root/vista-emulator'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x76e93bd0 in strchr ()
   from /home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/staging/lib/libc.so.6
(gdb) backtrace
#0  0x76e93bd0 in strchr ()
   from /home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/staging/lib/libc.so.6
#1  0x76e9511c in strstr ()
   from /home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/staging/lib/libc.so.6
#2  0x00490784 in log_activity (buffer=0x0) at vista-emulator.c:39
#3  0x00490808 in init_resources () at vista-emulator.c:52
#4  0x0049085c in main () at vista-emulator.c:72
(gdb) set confirm off
(gdb) quit
```

```console
$ rm root/vista-emulator
$ rm root/vista-emulator.core
```


## What to remember

During this lab, we learned that...

* It’s easy to study the behavior of programs and diagnose issues without even having the source code, thanks to `strace` and `ltrace`.

* You can leave a small `gdbserver` program (about 300 KB) on your target that allows to debug target applications, using a standard `gdb` debugger on the development host.

* It is fine to strip applications and binaries on the target machine, as long as the programs and libraries with debugging symbols are available on the development host.

* Thanks to *core dumps*, you can know where a program crashed, without having to reproduce the issue by running the program through the debugger.


## Licensing

This document is an extension to: [*Embedded Linux System Development - Practical Labs - QEMU Variant*](https://bootlin.com/doc/training/embedded-linux-qemu/)
 &mdash; &copy; 2004-2023, *Bootlin* [https://bootlin.com/](https://bootlin.com), [`CC-BY-SA-3.0`]((https://creativecommons.org/licenses/by-sa/3.0/)) license.
