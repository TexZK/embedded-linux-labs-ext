# Third-party libraries and apps


## Objectives

* Learn how to leverage existing libraries and applications: how to configure, compile and install them.

To illustrate how to use existing libraries and applications, we will extend the small *root* filesystem built in the *BusyBox* lab to add the *ALSA* libraries and tools to run basic sound support tests, and the *libgpiod* library and executables to manage GPIOs.<br/>
*ALSA* stands for *Advanced Linux Sound Architecture*, and it's the Linux audio subsystem.

We'll see that manually re-using existing libraries is quite tedious, so that more automated procedures are necessary to make it easier.
However, learning how to perform these operations manually will significantly help you when you face issues with more automated tools.


## Required tools

* Our [*cross-compile toolchain*](toolchain.md)

* *Ubuntu* packages:

    `autoconf-archive `
    `meson`
    `pkg-config`

* [*alsa-lib*](https://www.alsa-project.org/)

    * [Archived source code release `1.2.7.1`](https://www.alsa-project.org/files/pub/lib/alsa-lib-1.2.7.1.tar.bz2)

* [*alsa-utils*](https://www.alsa-project.org/)

    * [Archived source code release `1.2.6`](https://www.alsa-project.org/files/pub/utils/alsa-utils-1.2.6.tar.bz2)

* [*libgpiod*](https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git/about/), either as:

    * [*git* repository](https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git)

    * [Source code archive for commit `7311e1d5`](https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git/snapshot/libgpiod-7311e1d5b9473ab561977bd730acb793821a683f.tar.gz) (after release `1.6.3`)


* [*ipcalc*](https://gitlab.com/ipcalc/)

    * [Archived source code release `1.0.1`](https://gitlab.com/ipcalc/ipcalc/-/archive/1.0.1/ipcalc-1.0.1.tar.bz2)


## Figuring out library dependencies

We're going to integrate the `alsa-utils`, `libgpiod` and `ipcalc` executables.<br/>
In our case, the dependency chain for `alsa-utils` is quite simple: it only depends on the `alsa-lib` library.<br/>
Instead, `libgpiod` and `ipcalc` are standalone and thus don't have any dependencies.

Of course, all these libraries rely on the C library, which is not mentioned here, because it is already part of the *root* filesystem built in the *BusyBox* lab.<br/>
You might wonder how to figure out this dependency tree by yourself. Basically, there are several ways, that can be combined:

* Read the library documentation, which often mentions the dependencies.
* Read the help message of the configure script (by running `./configure --help`).
* By running the configure script, compiling and looking at the errors.

To configure, compile and install all the components of our system, we're going to start from the bottom of the tree with `alsa-lib`, then continue with `alsa-utils`. Then, we will also build `libgpiod` and `ipcalc`.


## Preparation

For our cross-compilation work, we will need two separate spaces:

* A *staging* space in which we will directly install all the packages: non-stripped versions of the libraries, headers, documentation and other files needed for the compilation.<br/>
This *staging* space can be quite big, but will not be used on our target, only for compiling libraries or applications;

* A *target* space, in which we will only copy the required files from the *staging* space: binaries and libraries, after stripping, configuration files needed at runtime, etc.<br/>
This *target* space will take a lot less space than the *staging* space, and it will contain only the files that are really needed to make the system work on the target.

To sum up, the *staging* space will contain everything that's needed for compilation, while the *target* space will contain only what's needed for execution.

Create the `$HOME/embedded-linux-bbb-labs/thirdparty/` directory, and inside create two directories: `staging` and `target`.

For the *target*, we need a basic system with *BusyBox* and initialization scripts.
We're going to re-use the system built in the *BusyBox* lab, so copy this system in the `target` directory.

```console
$ LAB_PATH="$HOME/embedded-linux-bbb-labs/thirdparty"
$ mkdir -p "$LAB_PATH/target/"
$ mkdir -p "$LAB_PATH/staging/"
$ cd $LAB_PATH
$ cp -arv ../tinysystem/nfsroot/* target/
    ...
```


## Testing

Make sure the `target/` directory is exported by your NFS server to your board by modifying `/etc/exports` and restarting your NFS server.
You can of course change our previous `/srv/nfs/` *symlink* to point to this directory.<br/>
Make your board boot from this new directory through NFS.

```console
$ sudo rm -f /srv/nfs
$ sudo ln -snv "$LAB_PATH/target/" /srv/nfs
'/srv/nfs' -> '/home/me/embedded-linux-bbb-labs/thirdparty/target/'
$ sudo chown -R tftp:tftp /srv/nfs
$ sudo exportfs -ar
$ sudo systemctl restart nfs-kernel-server
```

Revert *U-Boot* to use TFTP and NFS:

``` title="picocomBBB - U-Boot"
    ...
Hit any key to stop autoboot:  0
=> setenv bootcmd $bootcmd_tftp
=> setenv bootargs $bootargs_nfs
=> saveenv
=> reset
    ...
```


## `alsa-lib`

`alsa-lib` is a library supposed to handle the interaction with the *ALSA* subsystem.
It is available at [https://alsa-project.org/](https://alsa-project.org/).


### Source code

Download version `1.2.7.1`, and extract it in `$HOME/embeddedlinux-qemu-labs/thirdparty/`.

```console
$ cd $LAB_PATH
$ label="1.2.7.1"
$ wget "https://www.alsa-project.org/files/pub/lib/alsa-lib-${label}.tar.bz2"
$ tar xfv "alsa-lib-${label}.tar.bz2"
$ mv alsa-lib*/ alsa-lib
$ cd alsa-lib/
```

> **Tip:** if the website for any of the source packages that we need to download in the next sections is down, a great mirror that you can use is
[http://sources.buildroot.net/](http://sources.buildroot.net/).


### Configuration

Back to `alsa-lib` sources, look at the `configure` script and see that it has been generated by `autoconf` (the header contains a sentence like Generated by GNU Autoconf 2.69).
Most of the time, `autoconf` comes with `automake`, that generates *Makefiles* from `Makefile.am` files.
So `alsa-lib` uses a rather common *build system*.<br/>
Let's try to configure and build it:

```console hl_lines="5"
$ export MAKEFLAGS=-j$(nproc)
$ head -n3 configure
#! /bin/sh
# Guess values for system-dependent variables and create Makefiles.
# Generated by GNU Autoconf 2.69 for alsa-lib 1.2.7.1.
$ ./configure
$ make
```

If you look at the generated binaries, you'll see that they are *x86* ones because we compiled the sources with `gcc`, the default compiler.
This is obviously not what we want, so let's clean up the generated objects and tell the configure script to use the *ARM cross-compiler*.

```console hl_lines="2 10-11"
$ file aserver/aserver.o
aserver/aserver.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), with debug_info, not stripped
$ make clean
$ TC_NAME="arm-training-linux-uclibcgnueabihf"
$ TC_BASE="$HOME/x-tools/$TC_NAME"
$ export PATH="$TC_BASE/bin:$PATH"
$ CC=arm-linux-gcc ./configure
    ...
checking whether we are cross compiling... configure: error: in `/home/me/embedded-linux-bbb-labs/thirdparty/alsa-lib':
configure: error: cannot run C compiled programs.
If you meant to cross compile, use `--host'.
See `config.log' for more details
```

Of course, the `arm-linux-gcc` cross-compiler must be in your `PATH` prior to running the configure script.
The `CC` environment variable is the classical name for specifying the compiler to use.<br/>
Quickly, you should get an error, as highlighted above.

If you look at the `config.log` file, you can see that the `configure` script compiles a binary with the cross-compiler and then tries to run it on the development workstation. This is a rather usual thing to do for a `configure` script, and that's why it tests so early that it's actually doable, and bails out if not.

``` title="config.log"
    ...
configure:3719: arm-linux-gcc -o conftest    conftest.c  >&5
configure:3723: $? = 0
configure:3730: ./conftest
arm-binfmt-P: Could not open '/lib/ld-uClibc.so.0': No such file or directory
configure:3734: $? = 255
configure:3741: error: in `/home/me/embedded-linux-bbb-labs/thirdparty/alsa-lib':
configure:3743: error: cannot run C compiled programs.
If you meant to cross compile, use `--host'.
See `config.log' for more details
    ...
```

Obviously, it cannot work in our case, and the scripts exits.
The job of the `configure` script is to test the configuration of the system.
To do so, it tries to compile and run a few sample applications to test if this library is available, if this compiler option is supported, etc.
But in our case, running the test examples is definitely not possible.

We need to tell the `configure` script that we are cross-compiling, and this can be done using the `--build` and `--host` options, as described in the `help` of the `configure` script.

```text title="./configure --help"
    ...
System types:
  --build=BUILD     configure for building on BUILD [guessed]
  --host=HOST       cross-compile to build programs to run on HOST [BUILD]
    ...
```

The `--build` option allows to specify on which system the package is built, while the `--host` option
allows to specify on which system the package will run.
By default, the value of the `--build` option is guessed, and the value of `--host` is the same as the value of the `--build` option.
The value is guessed using the `./config.guess` script, which on your system should return `x86_64-pc-linux-gnu`.
See [the `autoconf` manual](https://www.gnu.org/software/autoconf/manual/html_node/Specifying-Names.html)
for more details on these options.

So, let's override the value of the `--host` option:

```console
$ ./configure --host=arm-linux
```

Note that `CC` is not required anymore. It is implied by `--host`.

The `configure` script should end properly now, and create a `Makefile`.
However, there is one subtle last issue to handle: because the C library used in our toolchain (*uClibc-ng*) does not support symbol versioning, we need to tell `alsa-lib` about this by passing `--without-versioned`.
Without this option, `alsa-lib` will build fine, but
[it will not work properly at runtime](https://mailman.alsa-project.org/pipermail/alsa-devel/2009-February/014999.html).


### Build

So, you should configure `alsa-lib` and `run` the make command, which should run just fine.

```console
$ make clean
$ ./configure --host=arm-linux --without-versioned
$ make
```

Look at the result of compiling in `src/.libs/`: a set of object files and a set of `libasound.so*` files.

```console hl_lines="12-14"
$ ls -l src/.libs/
total 4304
-rw-rw-r-- 1 me me   36160 Apr 22 10:45 async.o
-rw-rw-r-- 1 me me   15828 Apr 22 10:45 confeval.o
-rw-rw-r-- 1 me me   70424 Apr 22 10:45 confmisc.o
-rw-rw-r-- 1 me me  237212 Apr 22 10:45 conf.o
-rw-rw-r-- 1 me me   19108 Apr 22 10:45 dlmisc.o
-rw-rw-r-- 1 me me   11532 Apr 22 10:45 error.o
-rw-rw-r-- 1 me me   18688 Apr 22 10:45 input.o
lrwxrwxrwx 1 me me      15 Apr 22 10:45 libasound.la -> ../libasound.la
-rw-rw-r-- 1 me me     937 Apr 22 10:45 libasound.lai
lrwxrwxrwx 1 me me      18 Apr 22 10:45 libasound.so -> libasound.so.2.0.0
lrwxrwxrwx 1 me me      18 Apr 22 10:45 libasound.so.2 -> libasound.so.2.0.0
-rwxrwxr-x 1 me me 3981232 Apr 22 10:45 libasound.so.2.0.0
-rw-rw-r-- 1 me me    3524 Apr 22 10:45 names.o
-rw-rw-r-- 1 me me   20296 Apr 22 10:45 output.o
-rw-rw-r-- 1 me me    6468 Apr 22 10:45 shmarea.o
-rw-rw-r-- 1 me me    7292 Apr 22 10:45 socket.o
-rw-rw-r-- 1 me me    7532 Apr 22 10:45 userfile.o
```

The `libasound.so*` files are a dynamic version of the library.
The shared library itself is `libasound.so.2.0.0`, which was generated by the following command line:

```console title="Compilation of libasound.so.2.0.0 by make"
$ arm-linux-gcc -shared conf.o confmisc.o input.o output.o async.o  \
  error.o dlmisc.o socket.o shmarea.o userfile.o names.o  \
  -lm -ldl -lpthread -lrt -Wl,-soname -Wl,libasound.so.2  \
  -o libasound.so.2.0.0
```

It creates the *symbolic links* `libasound.so` and `libasound.so.2`.

```console title="Symbolic linking of libasound.so.2.0.0 by make"
$ ln -s libasound.so.2.0.0 libasound.so.2
$ ln -s libasound.so.2.0.0 libasound.so
```

These *symlinks* are needed for two different reasons:

* `libasound.so` is used at compile time when you want to compile an application that is dynamically linked against the library.
To do so, you pass the `-l<LIBNAME>` option to the compiler, which will look for a file named `lib<LIBNAME>.so`.
In our case, the compilation option is `-lasound` and the name of the library file is `libasound.so`.<br/>
So, the `libasound.so` *symlink* is needed at *compile time*.

* `libasound.so.2` is needed because it is the *SONAME* of the library. *SONAME* stands for *Shared Object Name*.
It is the name of the library as it will be stored in applications linked against this library.
It means that at runtime, the dynamic loader will look for exactly this name when looking for the shared library.<br/>
So, this *symbolic link* is needed at *runtime*.

To know what's the *SONAME* of a library, you can use:

```console hl_lines="5-7"
$ arm-linux-readelf -d src/.libs/libasound.so.2.0.0

Dynamic section at offset 0xdff20 contains 24 entries:
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libc.so.0]
 0x00000001 (NEEDED)                     Shared library: [ld-uClibc.so.1]
 0x0000000e (SONAME)                     Library soname: [libasound.so.2]
 0x0000000c (INIT)                       0x23b1c
 0x0000000d (FINI)                       0xbc22c
 0x00000019 (INIT_ARRAY)                 0xed2fc
 0x0000001b (INIT_ARRAYSZ)               4 (bytes)
 0x0000001a (FINI_ARRAY)                 0xed300
 0x0000001c (FINI_ARRAYSZ)               8 (bytes)
 0x00000004 (HASH)                       0x114
 0x6ffffef5 (GNU_HASH)                   0x4318
 0x00000005 (STRTAB)                     0x1021c
 0x00000006 (SYMTAB)                     0x7a7c
 0x0000000a (STRSZ)                      52374 (bytes)
 0x0000000b (SYMENT)                     16 (bytes)
 0x00000003 (PLTGOT)                     0xf0000
 0x00000002 (PLTRELSZ)                   7776 (bytes)
 0x00000014 (PLTREL)                     REL
 0x00000017 (JMPREL)                     0x21cbc
 0x00000011 (REL)                        0x1ceb4
 0x00000012 (RELSZ)                      19976 (bytes)
 0x00000013 (RELENT)                     8 (bytes)
 0x6ffffffa (RELCOUNT)                   2084
 0x00000000 (NULL)                       0x0
```

and look at the `(SONAME)` line. You'll also see that this library needs the C library, because of the `(NEEDED)` line on `libc.so.0`.
The mechanism of *SONAME* allows to change the library without recompiling the applications linked with this library.

Let's imagine that a security problem is found in the `alsa-lib` release that provides `libasound.so.2.0.0`, and fixed in the next `alsa-lib` release, which will now provide `libasound.so.2.0.1`.<br/>
You can just recompile the library, install it on your target system, change the `libasound.so.2` link so that it points to `libasound.so.2.0.1` and restart your applications.
It will work, because your applications don't look specifically for `libasound.so.2.0.0` but for the *SONAME* `libasound.so.2`.<br/>
However, it also means that as a library developer, if you break the *ABI* of the library, you must change the *SONAME*: change from `libasound.so.2` to `libasound.so.3`.


### Installation

Finally, the last step is to tell the `configure` script where the library is going to be installed.
Most `configure` scripts consider that the installation prefix is `/usr/local/` (so that the library is installed in `/usr/local/lib/`, the headers in `/usr/local/include/`, etc.).
Instead, in our system we simply want the libraries to be installed in the `/usr/` prefix, so let's tell the `configure` script about this:

```console
$ make clean
$ ./configure --host=arm-linux --without-versioned --prefix=/usr/
$ make
```

For this library, this option may not change anything to the resulting binaries.
But, for safety it is always recommended to make sure that the prefix matches where your library will be running on the target system.

Do not confuse the *prefix* (where the application or library will be running on the target system) with the location where the application or library will be installed on your host while building the *root* filesystem.

For example, `libasound` is installed in `$HOME/embedded-linux-bbb-labs/thirdparty/target/usr/lib/` because this is the directory where we are building the *root* filesystem.
But, once our target system is running, it looks for `libasound` in `/usr/lib/`.

The *prefix* corresponds to the path in the *target* system and **never** on the *host*.
So, one should **never** pass a *prefix* like `$HOME/embedded-linux-bbb-labs/thirdparty/target/usr/`, otherwise at runtime the application or library may look for files inside this directory on the *target*
system, which obviously doesn't exist!<br/>
By default, most build systems install the application or library in the given prefix (`/usr/` or `/usr/local/`), but with most build systems (including `autotools`), the installation *prefix* can be overridden,  different from the configuration *prefix*.

We now only have the installation process left to do.<br/>
First, let's make the installation in the *staging* space:

```console
$ make DESTDIR="$LAB_PATH/staging" install
$ cd "$LAB_PATH/staging/"
$ tree -C | less -R
```

Now look at what has been installed by `alsa-lib`:

* Some configuration files in `/usr/share/alsa/`:

    ```console
    $ ls -p usr/share/alsa/
    alsa.conf  cards/  ctl/  pcm/
    ```

* The headers in `/usr/include/`:

    ```console
    $ ls -p usr/include/
    alsa/  asoundlib.h  sys/
    ```

* The shared library and its `libtool` (`.la`) file in `/usr/lib/`:

    ```console
    $ ls -p usr/lib/
    libasound.la  libasound.so.2      libatopology.la  libatopology.so.2      pkgconfig/
    libasound.so  libasound.so.2.0.0  libatopology.so  libatopology.so.2.0.0
    ```

* A pkgconfig file in `/usr/lib/pkgconfig/` &mdash; we'll come back to these later:

    ```console
    $ ls -p usr/lib/pkgconfig/
    alsa.pc  alsa-topology.pc
    ```

Finally, let's install the library in the `target` space:

1. Create the `target/usr/lib` directory, it will contain the *stripped* version of the library.

2. Copy the *dynamic* version of the library. Only `libasound.so.2` and `libasound.so.2.0.0` are needed, since `libasound.so.2` is the *SONAME* of the library and `libasound.so.2.0.0` is the real binary.

3. Measure the size of the `target/usr/lib/libasound.so.2.0.0` library before *stripping*.

4. *Strip* the library.

5. Measure the size of the `target/usr/lib/libasound.so.2.0.0` library again after *stripping*.
How many unnecessary bytes were saved?

```console hl_lines="6 9"
$ cd $LAB_PATH
$ cp -av staging/usr/lib/libasound.so.2* target/usr/lib/
'staging/usr/lib/libasound.so.2' -> 'target/usr/lib/libasound.so.2'
'staging/usr/lib/libasound.so.2.0.0' -> 'target/usr/lib/libasound.so.2.0.0'
$ du -h target/usr/lib/libasound.so.2.0.0
3.8M    target/usr/lib/libasound.so.2.0.0
$ arm-linux-strip target/usr/lib/libasound.so.2.0.0
$ du -h target/usr/lib/libasound.so.2.0.0
904K    target/usr/lib/libasound.so.2.0.0
```

Then, we need to install the `alsa-lib` configuration files:

```console
$ mkdir -p target/usr/share
$ cp -arv staging/usr/share/alsa/ target/usr/share/
    ...
```

Now, we need to adjust one small detail in one of the configuration files.
Indeed, `/usr/share/alsa/alsa.conf` assumes a *UNIX group* called `audio` exists, which is not the case on our very small system; let's set to *root* instead (*group id* `0`).<br/>
Edit this file and replace `defaults.pcm.ipc_gid audio` with `defaults.pcm.ipc_gid 0` instead.

```console
$ sed -i "s/defaults.pcm.ipc_gid audio/defaults.pcm.ipc_gid 0/g"  \
  target/usr/share/alsa/alsa.conf
```

And we're done with `alsa-lib`!


## `alsa-utils`


### Source code

Download `alsa-utils` from the *ALSA* offical webpage. We tested the lab with version `1.2.6`.
Once uncompressed, we quickly discover that the `alsa-utils` build system is based on the *autotools*, so we will work once again with a regular `configure` script.

```console
$ cd $LAB_PATH
$ label="1.2.6"
$ wget "https://www.alsa-project.org/files/pub/utils/alsa-utils-${label}.tar.bz2"
$ tar xfv "alsa-utils-${label}.tar.bz2"
$ mv alsa-utils*/ alsa-utils
$ cd alsa-utils/
```


### Configuration

As we've seen previously, we have to provide the *prefix*, *host* options, and the `CC` variable.<br/>
We should quiclky get an error in the execution of the configure script.

```console hl_lines="4"
$ ./configure --host=arm-linux --prefix=/usr
    ...
checking for libasound headers version >= 1.2.5 (1.2.5)... not present.
configure: error: Sufficiently new version of libasound not found.
```

Again, we can check in `config.log` what the configure script is trying to do.

``` title="File: config.log" hl_lines="4"
    ...
configure:8080: checking for libasound headers version >= 1.2.5 (1.2.5)
configure:8127: arm-linux-gcc -c -g -O2  conftest.c >&5
conftest.c:12:10: fatal error: alsa/asoundlib.h: No such file or directory
   12 | #include <alsa/asoundlib.h>
      |          ^~~~~~~~~~~~~~~~~~
compilation terminated.
    ...
```

Of course, since `alsa-utils` uses `alsa-lib`, it includes its header file!
So, we need to tell the C compiler where the headers can be found.
They aren't in the default directory `/usr/include/`, but in the equivalent directory of our staging space.
The `help` text of the `configure` script says:

``` title="File: configure"
    ...
Some influential environment variables:
    ...
  CPPFLAGS    (Objective) C/C++ preprocessor flags, e.g. -I<include dir> if
              you have headers in a nonstandard directory <include dir>
    ...
```

Let's use it!
Now, it should stop a bit later, this time with another error.

```console hl_lines="7"
$ CPPFLAGS=-I"$LAB_PATH/staging/usr/include"  \
  ./configure --host=arm-linux --prefix=/usr
    ...
checking for libasound headers version >= 1.2.5 (1.2.5)... found.
    ...
checking for snd_ctl_open in -lasound... no
configure: error: No linkable libasound was found.
```

The configure script tries to compile an application against `libasound` (see the `-lasound` option): `alsa-utils` uses `alsa-lib`, so the `configure` script wants to make sure this library is already installed.<br/>
Unfortunately, the `ld` linker doesn't find it. So, let's tell the linker where to look for libraries using the `-L` option followed by the directory where our libraries are (in `staging/usr/lib`).
This `-L` option can be passed to the linker by using the `LDFLAGS` at `configure` time, as told by the help text of the configure script:

``` title="File: configure"
    ...
Some influential environment variables:
    ...
  LDFLAGS     linker flags, e.g. -L<lib dir> if you have libraries in a
              nonstandard directory <lib dir>
    ...
```

Let's use this LDFLAGS variable:

```console hl_lines="8"
$ LDFLAGS=-L"$LAB_PATH/staging/usr/lib"  \
  CPPFLAGS=-I"$LAB_PATH/staging/usr/include"  \
  ./configure --host=arm-linux --prefix=/usr
    ...
checking panel.h usability... no
checking panel.h presence... no
checking for panel.h... no
configure: error: required curses helper header not found
```

Once again, it should fail a bit further down the tests, this time complaining about a missing
*curses helper header*.
`curses` (or `ncurses`, *new curses*) is a graphical framework to design text UIs within the terminal.

This is only used by `alsamixer`, one of the tools provided by `alsa-utils`, that we are not going to use.
Hence, we can just disable the build of `alsamixer`. Of course, if we wanted it, we would have had to build `ncurses` first, just like we built `alsa-lib`.


### Build

Then, run the compilation with `make`. Hopefully, it works!

```console
$ LDFLAGS=-L"$LAB_PATH/staging/usr/lib"  \
  CPPFLAGS=-I"$LAB_PATH/staging/usr/include"  \
  ./configure --host=arm-linux --prefix=/usr --disable-alsamixer
$ make
```


### Installation

Let's now begin the installation process. Before really installing in the `staging` directory, let's install in a dummy directory, to see what's going to be installed &mdash; this dummy directory will not be used afterwards, it is only to verify what will be installed before polluting the staging space.

```console
$ make DESTDIR=/tmp/alsa-utils install
```

The `DESTDIR` variable can be used with all *Makefiles* based on `automake`.
It allows to override the installation directory: instead of being installed in the configuration `prefix` directory, the files will be installed in `$DESTDIR/configuration-prefix`.<br/>
Now, let's see what has been installed in `/tmp/alsa-utils/`;

```console
$ tree -C /tmp/alsa-utils/ | less -R
/tmp/alsa-utils/
├── lib
│   ├── systemd
│   │   └── system
│   │       ├── alsa-restore.service
│   │       ├── alsa-state.service
│   │       └── sound.target.wants
│   │           ├── alsa-restore.service -> ../alsa-restore.service
│   │           └── alsa-state.service -> ../alsa-state.service
│   └── udev
│       └── rules.d
│           └── 90-alsa-restore.rules
├── usr
│   ├── bin
│   │   ├── aconnect
│   │   ├── alsabat
│   │   ├── alsaloop
│   │   ├── alsatplg
│   │   ├── alsaucm
│   │   ├── amidi
│   │   ├── amixer
│   │   ├── aplay
│   │   ├── aplaymidi
│   │   ├── arecord -> aplay
│   │   ├── arecordmidi
│   │   ├── aseqdump
│   │   ├── aseqnet
│   │   ├── axfer
│   │   ├── iecset
│   │   └── speaker-test
│   ├── sbin
│   │   ├── alsabat-test.sh
│   │   ├── alsaconf
│   │   ├── alsactl
│   │   └── alsa-info.sh
│   └── share
│       ├── alsa
│       │   ├── init
│       │   │   ├── 00main
│       │   │   ├── ca0106
│       │   │   ├── default
│       │   │   ├── hda
│       │   │   ├── help
│       │   │   ├── info
│       │   │   └── test
│       │   └── speaker-test
│       │       └── sample_map.csv
│       ├── man
│       │   ├── fr
│       │   │   └── man8
│       │   │       └── alsaconf.8
│       │   ├── man1
│       │   │   ├── aconnect.1
│       │   │   ├── alsabat.1
│       │   │   ├── alsactl.1
│       │   │   ├── alsa-info.sh.1
│       │   │   ├── alsaloop.1
│       │   │   ├── amidi.1
│       │   │   ├── amixer.1
│       │   │   ├── aplay.1
│       │   │   ├── aplaymidi.1
│       │   │   ├── arecord.1 -> aplay.1
│       │   │   ├── arecordmidi.1
│       │   │   ├── aseqdump.1
│       │   │   ├── aseqnet.1
│       │   │   ├── axfer.1
│       │   │   ├── axfer-list.1
│       │   │   ├── axfer-transfer.1
│       │   │   ├── iecset.1
│       │   │   └── speaker-test.1
│       │   ├── man7
│       │   └── man8
│       │       └── alsaconf.8
│       └── sounds
│           └── alsa
│               ├── Front_Center.wav
│               ├── Front_Left.wav
│               ├── Front_Right.wav
│               ├── Noise.wav
│               ├── Rear_Center.wav
│               ├── Rear_Left.wav
│               ├── Rear_Right.wav
│               ├── Side_Left.wav
│               └── Side_Right.wav
└── var
    └── lib
        └── alsa

24 directories, 62 files
```

So, we have

* The `systemd` service definitions in `lib/systemd/`.
* The `udev` rules in `lib/udev/`.
* The `alsa-utils` binaries in `/usr/bin/` and `/usr/sbin/`.
* Some sound samples in `/usr/share/sounds/`.
* The various translations in `/usr/share/locale/`.
* The manual pages in `/usr/share/man/`, explaining how to use the various tools.
* Some configuration samples in `/usr/share/alsa/`.

Now, let's make the installation in the `staging` space:
Then, let's manually install only the necessary files in the `target` space. We are only interested in `speaker-test`:

```console
$ make DESTDIR="$LAB_PATH/staging" install
$ cd $LAB_PATH
$ cp -a staging/usr/bin/speaker-test target/usr/bin/
$ arm-linux-strip target/usr/bin/speaker-test
```

You may need to add the missing libraries from the toolchain install directory.

We're finally done with `alsa-utils`!


### Quick test

Now test that all is working fine by running the `speaker-test` utility on your board.

Now you can launch your machine as usual, with *rootfs* on NFS, and test `speaker-test` in a shell:

```console title="picocomBBB - BusyBox"
# speaker-test -t sine -l 1

speaker-test 1.2.6

Playback device is default
Stream parameters are 48000Hz, S16_LE, 1 channels
Sine wave rate is 440.0000Hz
Rate set to 48000Hz (requested 48000Hz)
Buffer size range from 2229 to 17832
Period size range from 1114 to 1115
Using max buffer size 17832
Periods = 4
was set period_size = 1114
was set buffer_size = 17832
 0 - Front Left
Time per period = 2.642531
```

There you are: you built and ran your first program depending on a library different from the C library!


## `libgpiod`


### Compiling

We are now going to use `libgpiod` instead of the deprecated interface in `/sys/class/gpio`.
Its executables (`gpiodetect`, `gpioset`, `gpioget`, *etc.*) will allow us to drive and manage GPIOs from shell scripts.

We are going to need code that is more recent than the latest release at the time of this writing (`1.6.3`), and checkout a commit that we tested (`7311e1d5`).

```console
$ cd $LAB_PATH
$ label="7311e1d5"
$ git clone https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git
$ cd libgpiod/
$ git checkout -b embedded-linux-bbb $label
```

Alternatively, you could use an archive of that commit
(see also: [*git commit hash expansion*](../kb/git-hash.md)).

```console
$ cd $LAB_PATH
$ label="7311e1d5b9473ab561977bd730acb793821a683f"
$ wget "https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git/snapshot/libgpiod-${label}.tar.gz"
$ tar xfv "libgpiod-${label}.tar.gz"
$ mv libgpiod*/ libgpiod
$ cd libgpiod/
```

As we are not starting from a release, we will need to install further development tools to generate some files like the configure script:

```console
$ sudo apt install autoconf-archive pkg-config
```

Now let's generate the files which are present in a release via `autogen.sh`.

```console
$ ./autogen.sh
    ...
```

Run `./configure --help` script, and see that this script provides an `--enable-tools` option, which allows to build the user-space executables that we want.

```console hl_lines="5"
$ ./configure --help
    ...
Optional Features:
    ...
  --enable-tools          enable libgpiod command-line tools [default=no]
    ...
```

As this project doesn't have any external library dependency, let's configure *libgpiod* in a similar way as *alsa-utils*, so that we can run `make`.<br/>
Installation to the *staging* space can be done using the classical `DESTDIR` mechanism.

```console
$ ./configure --host=arm-linux --prefix=/usr --enable-tools
$ make
$ make DESTDIR="$LAB_PATH/staging" install
```

Finally, only manually install and strip the files needed at runtime in the target space.

```console
$ cd $LAB_PATH
$ cp -a staging/usr/lib/libgpiod.so.2* target/usr/lib/
$ arm-linux-strip target/usr/lib/libgpiod*
$ cp -a staging/usr/bin/gpio* target/usr/bin/
$ arm-linux-strip target/usr/bin/gpio*
```


### Testing

First, connect `GPIO1_28` (pin 12 of connector *P9*) connected to ground (pin 1 of connector *P9*), as done for the [*Accessing Hardware Devices* lab](hardware.md).

Reboot the board, then run the `gpiodetect` command to check that you can list the various GPIO banks on your system.

```console title="picocomBBB - BusyBox" hl_lines="2"
# gpiodetect
gpiochip0 [gpio-0-31] (32 lines)
gpiochip1 [gpio-32-63] (32 lines)
gpiochip2 [gpio-64-95] (32 lines)
gpiochip3 [gpio-96-127] (32 lines)
```

So, `GPIO1_28` should be part of `gpiochip0`.

We can then get details on *GPIOE* GPIOs by running `gpioinfo gpiochip0`, or on all GPIOs by
simply running `gpioinfo`.

```console title="picocomBBB - BusyBox"
# gpioget gpiochip0 28
0
```

Now, connect your wire to 3.3 V (pin 3 or 4 of connector *P9*). You should now read:

```console title="picocomBBB - BusyBox"
# gpioget gpiochip0 28
1
```

You see that you didn’t have to configure the GPIO as input. *libgpiod* did that for you.

If you have an LED and a small breadboard (or M-F breadboard wires), you could also try to drive the GPIO in output mode.<br/>
Connect the short pin of the LED to GND, and the long one to the GPIO
Then then following command should light up the diode for 5 seconds (for example):

```console title="picocomBBB - BusyBox"
# gpioset -m time -s 5 gpiochip0 28=1
```

The `gpioset` command offers many more options.<br/>
Run `gpioset -h` to check by yourself.


## `ipcalc`

After practicing with autotools based packages, let's build `ipcalc`, which is using *Meson* as build system.
We won't really need this utility in our system, but at least it has no dependencies and therefore offers an easy way to build our first *Meson* based package.

So, first install the `meson` package:

```console
$ sudo apt install meson
```


### Source code

In the main lab directory, then let's check out the sources through git:

```console
$ cd $LAB_PATH
$ label="1.0.1"
$ git clone https://gitlab.com/ipcalc/ipcalc.git
$ cd ipcalc/
$ git checkout -b embedded-linux-bbb $label
```

Alternatively, you can retrieve an archived release:

```console
$ cd $LAB_PATH
$ label="1.0.1"
$ wget "https://gitlab.com/ipcalc/ipcalc/-/archive/${label}/ipcalc-${label}.tar.bz2"
$ tar xfv "ipcalc-${label}.tar.bz2"
$ mv ipcalc*/ ipcalc
$ cd ipcalc/
```


### Configuration

To cross-compile with *Meson*, we need to create a *cross file*.
Let's create the `$LAB_PATH/cross-file.txt` file with the following content:

```ini title="File: $LAB_PATH/cross-file.txt"
[binaries]
c = 'arm-linux-gcc'
[host_machine]
system = 'linux'
cpu_family = 'arm'
cpu = 'cortex-a9'
endian = 'little'
```

Command-line shortcut:

```console
$ cd "$LAB_PATH/ipcalc/"
$ cat > ../cross-file.txt <<'EOF'
[binaries]
c = 'arm-linux-gcc'
[host_machine]
system = 'linux'
cpu_family = 'arm'
cpu = 'cortex-a9'
endian = 'little'
EOF
```

We also need to create a special directory for building:

```console
$ mkdir cross-build
$ cd cross-build/
```


### Build

We can now have *Meson* create the *Ninja* build files for us:

```console
$ meson --cross-file ../../cross-file.txt --prefix /usr ..
The Meson build system
    ...
C compiler for the host machine: arm-linux-gcc (gcc 11.3.0 "arm-linux-gcc (crosstool-NG 1.25.0.95_7622b49) 11.3.0")
C linker for the host machine: arm-linux-gcc ld.bfd 1.25.0.95
C compiler for the build machine: cc (gcc 11.3.0 "cc (Ubuntu 11.3.0-1ubuntu1~22.04) 11.3.0")
C linker for the build machine: cc ld.bfd 2.38
Build machine cpu family: x86_64
Build machine cpu: x86_64
Host machine cpu family: arm
Host machine cpu: cortex-a9
Target machine cpu family: arm
Target machine cpu: cortex-a9
    ...
Build targets in project: 1

ipcalc 1.0.1

  User defined options
    Cross files: ../../cross-file.txt
    prefix     : /usr

Found ninja-1.10.1 at /usr/bin/ninja
WARNING: Cross file does not specify strip binary, result will not be stripped.
```

We're finally ready to build `ipcalc`:

```console
$ ninja
[7/7] Linking target ipcalc
```


### Installation

And now install `ipcalc` into the `staging` space.
Check that the `staging/usr/bin/ipcalc` file is indeed an *ARM* executable!

```console
$ DESTDIR="$LAB_PATH/staging" ninja install
[0/1] Installing files.
Installing ipcalc to /home/me/embedded-linux-bbb-labs/thirdparty/staging/usr/bin
$ file "$LAB_PATH/staging/usr/bin/ipcalc"
/home/me/embedded-linux-bbb-labs/thirdparty/staging/usr/bin/ipcalc: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-uClibc.so.0, with debug_info, not stripped
```

The last thing to do is to copy it to the target space and strip it.

```console
$ cd $LAB_PATH
$ cp staging/usr/bin/ipcalc target/usr/bin/
$ arm-linux-strip target/usr/bin/ipcalc
```

Note that we could have asked `ninja install` to strip the executable for us when installing it into the `staging` directory.
To do, this, we would have added a `strip` entry in the *cross file*, and passed `--strip` to *Meson*.
However, it's better to keep files unstripped in the `staging` space, in case we need to debug them!


### Quick test

You can now test that `ipcalc` works on the target:

```console title="picocomBBB - BusyBox"
# ipcalc 192.168.0.15
Address:        192.168.0.15
Address space:  Private Use
```


## Final touch

To finish this lab completely, and to be consistent with what we've done before, let's *strip* the *C library* and its *loader* too. Let's compare the size of the binaries to confirm.

```console
$ du -hs target/lib/ target/usr/lib/
78M     target/lib/
976K    target/usr/lib/
$ so_files=$(find target -name "*.so*")
$ chmod +w $so_files
$ arm-linux-strip $so_files
$ ko_files=$(find target -name "*.ko")
$ chmod +w $ko_files
$ arm-linux-strip $ko_files
$ du -hs target/lib/ target/usr/lib/
17M     target/lib/
976K    target/usr/lib/
```


## Backup and restore

```console
$ cd $LAB_PATH
$ tar cfJv thirdparty-staging.tar.xz staging/
$ tar cfJv thirdparty-target.tar.xz target/
$ cd target/
$ find . -depth -print0 | cpio -ocv0 | xz > ../thirdparty-target.cpio.xz
```


## Licensing

This document is an extension to: [*Embedded Linux System Development - Practical Labs - BeagleBone Black Variant*](https://bootlin.com/doc/training/embedded-linux-bbb/)
 &mdash; &copy; 2004-2023, *Bootlin* [https://bootlin.com/](https://bootlin.com), [`CC-BY-SA-3.0`]((https://creativecommons.org/licenses/by-sa/3.0/)) license.
