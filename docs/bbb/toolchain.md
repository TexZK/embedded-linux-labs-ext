# Building a cross-compile toolchain


## Objectives

After this lab, you will be able to:

* Configure the *crosstool-NG* tool.

* Execute *crosstool-NG* and build up your own *cross-compiling toolchain*.


## Required tools

* Ubuntu packages:

    `autoconf`
    `bison`
    `build-essential`
    `flex`
    `gawk`
    `git`
    `help2man`
    `libncurses-dev`
    `libtool-bin`
    `qemu-user`
    `texinfo`
    `unzip`
    `wget`
    `xz-utils`

* [crosstool-NG](https://crosstool-ng.github.io/), either from:

    * [*GitHub* repository](https://github.com/crosstool-ng/crosstool-ng)

    * [Source code archive of commit `7622b490`](https://github.com/crosstool-ng/crosstool-ng/archive/7622b490a359f6cc6b212859b99d32020a8542e7.zip)


## Source code

First, install some packages required for compilation:

```console
$ sudo apt install \
  autoconf bison build-essential cpio flex gawk git help2man \
  libncurses-dev libtool-bin texinfo unzip wget xz-utils
```

Enter the folder of this lab, that's going to become our main workspace folder:

```console
$ LAB_PATH="$HOME/embedded-linux-bbb-labs/toolchain"
$ cd $LAB_PATH
```

You can now get *crosstool-NG* at the suggested version (*git* commit `7622b490`).

We're going to clone the *git* repository into the home folder, creating a new *branch* named after the *embedded-linux-bbb* tutorial just for convenience.

```console
$ git clone "https://github.com/crosstool-ng/crosstool-ng"
$ cd crosstool-ng/
$ label="7622b490"
$ git checkout -b embedded-linux-bbb ${label}
```

Alternatively, you can directly unpack an archive of the suggested version. This is usually much faster than cloning a big *git* repository, despite losing all the features of a *git* repository.

```console
$ label="7622b490a359f6cc6b212859b99d32020a8542e7"
$ wget "https://github.com/crosstool-ng/crosstool-ng/archive/${label}.zip"
$ unzip "${label}.zip"
$ mv crosstool-ng*/ crosstool-ng
```

> See: [*git commit* hash expansion](../kb/git-hash.md)


## Bootstrap

Now that the source code is available, its content requires some initial setup to become available. The local `bootstrap` executable takes care of this initial setup.

```console
$ cd $LAB_PATH/crosstool-ng/
$ ./bootstrap
```

We're going to compile and install everything locally, so you must
*configure* the code base for that, via the canonical `configure` executable.<br/>
To improve compile time from now on, let's provide the
[number of processors](../kb/processors.md) to `make`.

```console
$ export MAKEFLAGS=-j$(nproc)
$ ./configure --enable-local
$ make
```

From now on, the generated `ct-ng` executable is our main command for the toolchain generation.


## Configuration

A single installation of *crosstool-NG* allows to produce as many toolchains as you want, for different architectures, with different C libraries and different versions of the various components.

*crosstool-NG* comes with a set of ready-made configuration files for various typical setups, called *samples*. They can be listed by calling the `list-samples` command.

We're going to load the sample for the *ARM Cortex A9* processor, so let's filter the results:

```console hl_lines="2"
$ ./ct-ng list-samples | grep a8
[L...]   arm-cortex_a8-linux-gnueabi
```

Our target sample is `arm-cortexa9_neon-linux-gnueabihf`, so let's configure for it, generating a `.config` file:

```console
$ ./ct-ng arm-cortex_a8-linux-gnueabi
    ...
Now configured for "arm-cortex_a8-linux-gnueabi"
```

We can refine the configuration via a useful user interface called `menuconfig`:

```console
$ ./ct-ng menuconfig
```

> See: [`menuconfig`](../kb/menuconfig.md)


In `Path and misc options`:

* To resume a failed compilation, enable the following options.
  Without these, you have to restart the whole compilation from scratch, which usually takes a long time.

    * Enable `Debug crosstool-NG` (`DEBUG_CT`)
    * Enable `Save intermediate steps` (`CT_DEBUG_CT_SAVE_STEPS`)
    * Enable `gzip saved states` (`CT_DEBUG_CT_SAVE_STEPS_GZIP`)
    * Enable `Interactive shell on failed commands` (`CT_DEBUG_INTERACTIVE`)

* Set `Number of parallel jobs` (`CT_PARALLEL_JOBS`) = [number of processors](../kb/processors.md).<br/>
  This improves the performance of the building pipeline.

* Set `Maximum log level to see:` (`LOG_DEBUG`) = `DEBUG`.<br/>
  This way we can have more details on what happened during the build in case something went wrong.<br/>
  This isn't strictly required, and might produce too many messages. I'm keeping it to `EXTRA` actually.


In `Debug facilities`:

* Remove all the options here.
  Some debugging tools can be provided in the toolchain, but they can also be built by filesystem building tools.<br/>
  **Do this before anything else**: removing features often messes up elsewhere (*IPv6* and *WCHAR* for example)!


In `Target options`:

* Set `Use specific FPU` (`ARCH_FPU`) = `vfpv3`.

* Set `Floating point` to `hardware (FPU)` (`CT_ARCH_FLOAT_HW`).


In `Toolchain options`:

* Set `Tuple's vendor string` (`TARGET_VENDOR`) = `training`.

* Set `Tuple's alias` (`TARGET_ALIAS`) = `arm-linux`.<br/>
  This way, we will be able to use the compiler as `arm-linux-gcc` instead of <br/>
  `arm-training-linux-uclibcgnueabihf-gcc`, which is much longer to type.


In `Operating System`:

* Set `Version of linux` = `5.15.x` version that is proposed.<br/>
  We choose this version because this matches the version of the kernel we will run on the board.
  At least, the version of the kernel headers are not more recent (which might be incompatible).


In `C-library`:

* Set `C library` (`LIBC_UCLIBC_NG`) = `uClibc-ng`.

* Keep the default version that is proposed.

* Enable `Add support for IPv6` (`LIBC_UCLIBC_IPV6`), required by [*Buildroot*](buildroot.md).

* Enable `Add support for WCHAR` (`LIBC_UCLIBC_WCHAR`), required by [*Buildroot*](buildroot.md).

* Enable `Support stack smashing protection (SSP)` (`LIBC_UCLIBC_HAS_SSP`), required by [*Buildroot*](buildroot.md).


In `C compiler`:

* Set `Version of gcc` = `11.3.0`.<br/>
  We need to stick to *GCC 11.x*, because *Buildroot 2022.02* (which we are going to use later) doesn't support *GCC 12.x* toolchains yet (released after).

* Enable `C++` (`CC_LANG_CXX`).


> **Do cross-check everything now!**<br/>
> It can happen that, when changing feature states, some overlooked dependency chains mess up with the configuration!<br/>
> For example:
>
> *defconfig* `arm-cortex_a8-linux-gnueabi` &rArr; `DEBUG_GDB=y` &rArr; `LIBC_UCLIBC_IPV6=y`
>
> So, it looked like *IPv6* support was ok.<br/>
> But, after removing `gdb` (`DEBUG_GDB=n`) from the `Debug facilities`:
>
> `DEBUG_GDB=n` &rArr; `LIBC_UCLIBC_IPV6=n`
>
> We could have forced `LIBC_UCLIBC_IPV6=y` by pressing ++y++ even if it looked automatically enabled (marked with `-*-`), but I've found no easy visual feedback to ensure that.<br/>
> So, cross-checking before leaving is strongly encouraged!
> I've lost a lot of time re-building the toolchain because of overlooking these naiveities, just because *Buildroot* slammed errors in my face so late!


You can now `<Save>` this configuration to the default `.config` file.

It's best to save a back-up copy as well:

```console
$ cp .config ../crosstool-ng.config
```

You'd better also create the `~/src/` folder, where *crosstool-NG* stores the downloaded tarballs by default.
It's handy to store them in case of any errors or cleanup, to avoid repeated downloads.

```console
$ mkdir -p ~/src/
```


## Build

Run the `build` command:

```console
$ ./ct-ng build
```

The build takes quite a long time to perform &mdash; a clean build took around 50 minutes on an *Intel i7 7700* laptop with 4 cores, of course within the *Lubuntu* VM.<br/>
It requires a stable internet connection.

In case of any errors, the interactive error shell can help to resume compilation from the failed step.
Possible errors might occur because of an unstable connection, or because the VM was provided a small amount of RAM (suggested &ge; 4 GB).

The toolchain is installed by default under `~/x-tools/`.<br/>
In our example: `~/x-tools/arm-training-linux-uclibcgnueabihf/`.<br/>
That's something you could have changed in the configuration of *crosstool-NG*.


## Backup and restore

It's best to save an archive of the generated toolchain, because building times can be very long.
Let's archive it into the folder of the current lab.

We're going to use `cpio` instead of the common `tar` to preserve *symlinks* as they are within the filesystem.

> FYI: [https://linuxconfig.org/how-to-create-and-extract-cpio-archives-on-linux-examples](https://linuxconfig.org/how-to-create-and-extract-cpio-archives-on-linux-examples)

```console
$ pushd .
$ TC_NAME="arm-training-linux-uclibcgnueabihf"
$ TC_BASE="$HOME/x-tools/$TC_NAME"
$ cd $TC_BASE/..
$ find . -depth -print0 | cpio -ocv0 | xz > "$LAB_PATH/${TC_NAME}.cpio.xz"
$ popd
```

> `pushd`/`popd` allow to push/pop the current directory (`pwd`) against the directory stack (`dirs`).

To restore it under `~/x-tools/`, just unpack the backup archive there:

```console
$ cd $LAB_PATH
$ sudo rm -rf $TC_BASE
$ mkdir -p "$HOME/x-tools/"
$ xzcat "${TC_NAME}.cpio.xz" | cpio -iduv -D "$HOME/x-tools/"
```


## Quick test

The toolchain can be called by adding the generated `bin` folder to your `PATH` environment variable.

```console
$ export PATH="$TC_BASE/bin:$PATH"
```

You can now compile the simple `hello.c` with our new shiny `arm-linux-gcc` shorthand:

```console
$ cd $LAB_PATH
$ arm-linux-gcc -o hello hello.c
```

You can use the `file` command on your binary to make sure it has correctly been compiled for the ARM architecture.

```console
$ file hello
hello: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-uClibc.so.0, not stripped
```

Did you know that you can still execute this binary from your *x86* host? To do this, install the *QEMU user* emulator, which just emulates target instruction sets, not an entire system with devices.

```console
$ sudo apt install qemu-user
```

Try to run *QEMU* with the `hello` executable generated with our toolchain:

```console
$ qemu-arm hello
qemu-arm: Could not open '/lib/ld-uClibc.so.0': No such file or directory
```

What's happening is that `qemu-arm` is missing the shared C library (compiled for ARM) that
this binary uses. Let's find it in our newly compiled toolchain:

```console
$ find ~/x-tools/ -name ld-uClibc.so.0
/home/me/x-tools/arm-training-linux-uclibcgnueabihf/arm-training-linux-uclibcgnueabihf/sysroot/lib/ld-uClibc.so.0
```

We can now use the `-L` option of `qemu-arm` to let it know where shared libraries are:

```console
$ qemu-arm -L "$TC_BASE/$TC_NAME/sysroot/" hello
Hello world!
```


## Cleaning up

To save about 11 GB of storage space, run the `clean` command in the *crosstool-NG* source directory.

```console
$ ./ct-ng clean
```

This removes the source code of the toolchain components, as well as all the generated files that are now useless, since the toolchain has been installed in `~/x-tools`, and we made a back-up archive.

Do this only if you have limited storage space.
In case you made a mistake in the toolchain configuration, you may need to run *crosstool-NG* again to rebuild everything form scratch &mdash; keeping generated files would save a significant amount of time.


## Licensing

This document is an extension to: [*Embedded Linux System Development - Practical Labs - BeagleBone Black Variant*](https://bootlin.com/doc/training/embedded-linux-bbb/)
 &mdash; &copy; 2004-2023, *Bootlin* [https://bootlin.com/](https://bootlin.com), [`CC-BY-SA-3.0`]((https://creativecommons.org/licenses/by-sa/3.0/)) license.
