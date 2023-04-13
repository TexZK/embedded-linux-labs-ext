# App - Development


## Objectives

* Compile and run your own *ncurses* application on the target.


## Required tools

* Our [*cross-compile toolchain*](toolchain.md)

* Ubuntu packages: those from the previous labs.


## Setup

Go to the `$HOME/embedded-linux-qemu-labs/appdev` directory.

```console
$ LAB_PATH="$HOME/embedded-linux-qemu-labs/appdev"
```

## Compile your own application

In the lab directory, the file `app.c` contains a very simple *ncurses* application.
It is a simple game where you need to reach a target using the arrow keys of your keyboard.
We will compile and integrate this simple application to our Linux system.

We will re-use the system built during the *Buildroot* lab and add to it our own application.
*Buildroot* has generated toolchain wrappers in `output/host/bin`, which make it easier to use the toolchain, since these wrappers pass some mandatory flags (especially the `--sysroot` *GCC* flag, which tells *GCC* where to look for the headers and libraries).

Let’s add this directory to our `PATH`.<br/>
**NOTE:** the *Buildroot GCC* wrappers **MUST** take priority over our toolchain, otherwise these wrappers won't be called!

```console
$ export PATH="$HOME/embedded-linux-qemu-labs/buildroot/buildroot/output/host/bin:$PATH"
$ which arm-linux-gcc
/home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/host/bin/arm-linux-gcc
```

Let’s try to compile the application.<br/>
It should complain about undefined references to some symbols. This is normal, since we didn’t tell the compiler to link with the necessary libraries.

```console hl_lines="4 6"
$ cd $LAB_PATH
$ arm-linux-gcc -o app app.c
    ...
app.c:(.text+0x30): undefined reference to `initscr'
    ...
collect2: error: ld returned 1 exit status
```

So, let’s use `pkg-config` to query the *pkgconfig* database about the location of the header files and the list of libraries needed to build an application against `ncurses`.<br/>
There's indeed a special `pkg-config` under `output/host/bin/` that automatically knows where to look, so it already knows the right paths to find `.pc` files and their *sysroot*.

```console
$ which pkg-config
/home/me/embedded-linux-qemu-labs/buildroot/buildroot/output/host/bin/pkg-config
$ pkg-config --libs --cflags ncurses
-D_GNU_SOURCE -lncurses
$ arm-linux-gcc -o app app.c $(pkg-config --libs --cflags ncurses)
```

Our application is now compiled!
Copy the generated binary to the NFS *root* filesystem (in the `root/` directory for example), start your system, and run your application!

```console
$ cp app "$LAB_PATH/../buildroot/nfsroot/root/"
```

```console title="QEMU - Buildroot"
# /root/app
                                 Hello World!






                                 X



                                       O











Move to the target (X), 'q' to quit
```

## Backup and restore

```console
$ cd $LAB_PATH
$ tar cfJv app.tar.xz app
$ cd "$LAB_PATH/../buildroot/nfsroot/"
$ find . -depth -print0 | cpio -ocv0 | xz > "$LAB_PATH/nfsroot-appdev.cpio.xz"
```


## Licensing

This document is an extension to: [*Embedded Linux System Development - Practical Labs - QEMU Variant*](https://bootlin.com/doc/training/embedded-linux-qemu/)
 &mdash; &copy; 2004-2023, *Bootlin* [https://bootlin.com/](https://bootlin.com), [`CC-BY-SA-3.0`]((https://creativecommons.org/licenses/by-sa/3.0/)) license.
