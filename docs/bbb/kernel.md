# Linux Kernel


## Objectives

* Get the kernel sources from *git*, using the official Linux source tree.

* Fetch the sources for the *stable* Linux releases, by declaring a *remote* tree and getting *stable* branches from it.

* Set up a cross-compiling environment.

* Cross compile the kernel for the *BeagleBone Black* board (*ARM Cortex-A8*).

* Use *U-Boot* to download the kernel to the target board.

* Check that the custom kernel starts the system.


## Required tools

* Our [*cross-compile toolchain*](toolchain.md)

* Ubuntu packages: those from the previous labs.

* [*Linux kernel*](https://www.kernel.org/), either as:

    * [Linus Torvalds' *git* repository](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux/) tag `5.15`

    * [Source code archive for *bleeding-edge* release `5.15`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/snapshot/linux-5.15.tar.gz)

    * [Stable *git* repository](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux/) tag `5.15.104`

    * [Source code archive for *stable* release `5.15.104`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/snapshot/linux-5.15.104.tar.gz)


## Main repository

Let's first create the `kernel` folder under or lab folder:

```console
$ LAB_PATH="$HOME/embedded-linux-bbb-labs/kernel"
$ mkdir -p $LAB_PATH
$ cd $LAB_PATH
```

To begin working with the Linux kernel sources, we need to clone its reference *git* tree, the one managed by Linus Torvalds himself.

This requires downloading some gigabytes of data. If you have a very fast access to the Internet, you can do it directly by connecting to [the official *git* repository](https://git.kernel.org) (our main *remote* repository):

```console
$ git clone "https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux/"
$ cd linux
```

If your internet access is not fast enough, you can download a *git* snapshot of a specific version; for example, release `5.15`.<br/>
You just have to extract this archive in the current `kernel` directory, just like we did for the previous labs:

```console
$ cd $LAB_PATH
$ label="5.15"
$ wget "https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/snapshot/linux-${label}.tar.gz"
$ tar xfv "linux-${label}.tar.gz"
$ mv linux*/ linux
$ cd linux
```


## Stable releases

The Linux kernel repository from Linus Torvalds contains all the *main* releases of Linux, but not the *stable* releases: they are maintained by a separate team, and hosted in a separate repository.

After having downloaded the *main* repository, we have to add this separate `stable` repository as additional *remote* to be able to use the *stable* releases.

```console
$ cd $LAB_PATH/linux
$ git remote add stable "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux/"
$ git fetch stable
$ git branch -a
```

We're going to checkout the *stable* branch for version `5.15.y`, aliased as a branch named after our labs; alternatively, we can choose the specific version `5.15.104`:

```console
$ label="linux-5.15.y"  # for the ongoing branch
$ label="v5.15.104"     # for our specific version
$ git checkout -b embedded-linux-bbb "stable/${label}"
```

Again, if you internet speed is slow, or you want to save space, you can download a specific release archive directly.
You can find the list by browsing the repository webpage; we tested this lab with version `5.15.104`, an *LTS* branch. Of course, this is an alternative to the *main* releases, so make sure that one wasn't extracted into our `linux` subfolder.

```console
$ cd $LAB_PATH
$ rm -rf linux/
$ label="5.15.104"
$ wget "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/snapshot/linux-${label}.tar.gz"
$ tar xfv "linux-${label}.tar.gz"
$ mv linux*/ linux
$ cd linux
```


## Version check

First, execute the following command to check which version you currently have:

```console
$ make kernelversion
5.15.104
```

You can also open the `Makefile` and look at the first lines to find this information:

```console
$ head -n6 Makefile | tail -n+2
VERSION = 5
PATCHLEVEL = 15
SUBLEVEL = 104
EXTRAVERSION =
NAME = Trick or Treat
```


## Configuration

To cross-compile Linux, you need to have a cross-compiling toolchain. We will use the cross-compiling toolchain that we previously produced, so we just need to add it to the `PATH`. We also need the `CROSS_COMPILE` prefix, and the `ARCH` label of the CPU architecture. You'd better parallelize with the `-j` option to save time.<br/>
Quick reminder:

```console
$ TC_NAME="arm-training-linux-uclibcgnueabihf"
$ TC_BASE="$HOME/x-tools/$TC_NAME"
$ export PATH="$TC_BASE/bin:$PATH"
$ export CROSS_COMPILE=arm-linux-
$ export ARCH=arm
$ export MAKEFLAGS=-j$(nproc)
```

By running `make help`, look for the proper `Makefile` target to configure the kernel for your processor (within `less`, press the `[Q]` key to quit).<br/>
In our case, look for a configuration for boards in the *OMAP2 and later* family, which the *AM335x*
of the *BeagleBone* belongs to: `omap2plus_defconfig`.<br/>
So, apply this configuration, and then run `make menuconfig`.

```console
$ make help | grep omap2
  omap2plus_defconfig         - Build for omap2plus
$ make omap2plus_defconfig
$ make menuconfig
```

> See: [`menuconfig`](../kb/menuconfig.md)

* Set `GCC_PLUGINS=n` (*disabled*), which skips building special GCC plugins we don't need, requiring extra dependencies for the build.

* Add options to support USB host and networking over USB device:

    * Set `USB=y` (*static*), *i.e.* search for it (++slash++), and press ++y++ to switch from *module* to *static*.

    * Set `USB_GADGET=y`.

    * Set `USB_MUSB_HDRC=y`, the driver for the USB OTG controller.

    * Set `USB_MUSB_DSPS=y`.

    * Set `USB_MUSB_DUAL_ROLE=y` (via choice menu) to use the *USB OTG* both in *host* and *device* (*gadget*) modes.
      This will be needed to use the board's *USB host* port.

    * Check the dependencies of `AM335X_PHY_USB` and find the way to set `AM335X_PHY_USB=y`.

        * Use the *search* tool of `menuconfig` against `AM335X_PHY_USB`.

        * You can see that it depends on:<br/>
          `USB_SUPPORT [=y] && (ARM [=y] || COMPILE_TEST [=n]) && NOP_USB_XCEIV [=m]`.<br/>
          We're missing `NOP_USB_XCEIV`, because it's compiled as *module* right now, so we cannot have `AM335X_PHY_USB=y`.

        * Search for and set `NOP_USB_XCEIV=y`.

        * Search again `AM335X_PHY_USB` to confirm the dependency is now resolved.

        * Set `AM335X_PHY_USB=y` and confirm the dependency chain is all *static*.

    * Set `USB_ETH=y`.

    * Set `USB_ETH_RNDIS=y`.

    * Set `INPUT_EVDEV=y`, to have the same default setting as in our labs with the *STM32MP1* boards.

* Add *static* support (instead of *module*) for the *hardware random generator*.
  Without it, you would get annoying messages and lower performance.

    * Set `CONFIG_HW_RANDOM=y`.
    * Set `CONFIG_HW_RANDOM_OMAP=y`.
    * Set `CONFIG_HW_RANDOM_OMAP3=y`.

Now you can `<Save>` and backup:

```console
$ cp .config ../kernel.config
```


## Cross compiling

You’re now ready to cross-compile your kernel. Simply run `make` and wait for the kernel to be compiled.
The build takes some time to perform &mdash; a clean build took around 10 minutes on an *Intel i7 7700* laptop with 4 cores, of course within the *Lubuntu* VM.

```console
$ make
    ...
  Kernel: arch/arm/boot/zImage is ready
```

Look at the end of the kernel build output to see which file contains the kernel image.<br/>
You can also see the Device Tree `.dtb` files which got compiled. Find which `.dtb` file corresponds to your board. **TODO**

```console hl_lines="6"
$ find . -name "am335x-bone*.dtb"
./arch/arm/boot/dts/am335x-boneblue.dtb
./arch/arm/boot/dts/am335x-bonegreen-wireless.dtb
./arch/arm/boot/dts/am335x-bonegreen.dtb
./arch/arm/boot/dts/am335x-bone.dtb
./arch/arm/boot/dts/am335x-boneblack.dtb
./arch/arm/boot/dts/am335x-boneblack-wireless.dtb
```


## Bootloader

As we are going to boot the Linux kernel from [our *U-Boot* installation](bootloader.md), we need to set the `bootargs` environment variable according to the
[Linux kernel command line](https://www.kernel.org/doc/html/v5.15/admin-guide/kernel-parameters.html).

Let's connect `picocomBBB` and run our *U-Boot* on the board.<br/>
Press any key as soon as the board boots, to stop *U-Boot* and enter its prompt.

> A separate shell is suggested for `picocomBBB` instances from now on.

```console title="picocomBBB - U-Boot"
$ picocomBBB
    ...
Terminal ready
    ...
Hit any key to stop autoboot:  0
=>
```

Enter the *prompt* (press a key before the timeout) and set the `bootargs` environment variable:

``` title="picocomBBB - U-Boot"
=> setenv bootargs console=ttyS0,115200n8
=> saveenv
```

We use TFTP to load the kernel image on the board:

On your workstation, copy the `zImage` and DTB (`am335x-boneblack.dtb`) to the directory exposed by the TFTP server (`/srv/tftp/`).

```console
$ cd "$LAB_PATH/linux/"
$ cp "arch/$ARCH/boot/zImage" /srv/tftp/
$ cp "arch/$ARCH/boot/dts/am335x-boneblack.dtb" /srv/tftp/
```

On the target (within the *U-Boot* prompt, accessed by pressing a key before the initial timeout), load `zImage` from *TFTP* into RAM, as well as the DTB, and let the kerbel boot rom RAM with its device tree.<br/>
You should see Linux boot and finally panicking. This is expected: we haven’t provided a working root filesystem for our device yet!

``` title="picocomBBB - U-Boot" hl_lines="7"
=> tftp 0x81000000 zImage
    ...
=> tftp 0x82000000 am335x-boneblack.dtb
    ...
=> bootz 0x81000000 - 0x82000000
    ...
---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

You can now automate all of this every time the board is booted or reset.<br/>
Reset the board (`S1` button), enter *U-Boot* prompt, and set the `bootcmd` environment variable, chaining the previous commands in sequence in a long line.

``` title="picocomBBB - U-Boot"
=> setenv bootcmd "tftp 0x81000000 zImage;  tftp 0x82000000 am335x-boneblack.dtb;  bootz 0x81000000 - 0x82000000"
=> saveenv
```

Restart the board again to make sure that booting the kernel is now automated.

``` title="picocomBBB - U-Boot" hl_lines="3"
=> reset
    ...
---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

You can now disconnect and turn off the board.


## Backup and restore

This isn't really required now, because we're going to recompile the kernel and adapt the *U-Boot* configuration again.
Anyway, in case you need them, you can backup a snapshot of the images up to this point.

```console
$ cd /srv/tftp/
$ tar cfJv "$LAB_PATH/kernel-tftp.tar.xz" zImage am335x-boneblack.dtb
```

To restore the content:

```console
$ sudo mkdir -p /srv/tftp/
$ sudo chown tftp:tftp /srv/tftp
$ cd /srv/tftp/
$ tar xfv "$LAB_PATH/kernel-tftp.tar.xz"
```


## Licensing

This document is an extension to: [*Embedded Linux System Development - Practical Labs - BeagleBone Black Variant*](https://bootlin.com/doc/training/embedded-linux-bbb/)
 &mdash; &copy; 2004-2023, *Bootlin* [https://bootlin.com/](https://bootlin.com), [`CC-BY-SA-3.0`]((https://creativecommons.org/licenses/by-sa/3.0/)) license.
