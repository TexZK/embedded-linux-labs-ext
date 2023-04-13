# Tiny embedded system with BusyBox


## Objectives

* Make a tiny yet full-featured embedded system.

* Be able to configure and build a Linux kernel that boots on a directory on your workstation, shared through the network by *NFS*.

* Be able to create and configure a minimalistic root filesystem from scratch (*ex nihilo*, out of nothing, entirely hand made...) for your target board.

* Understand how small and simple an embedded Linux system can be.

* Be able to install *BusyBox* on this filesystem.

* Be able to create a simple startup script based on `/sbin/init`.

* Be able to set up a simple web interface for the target.


## Required tools

* Our [*cross-compile toolchain*](toolchain.md)

* Ubuntu packages:

    `nfs-kernel-server`

    plus those from the previous labs.

* [*BusyBox*](https://busybox.net/), either as:

    * [*git* repository](https://git.busybox.net/busybox/) tag `1_35_0`

    * [Source code archive for release `1_35_0`](https://git.busybox.net/busybox/snapshot/busybox-1_35_0.tar.bz2)


## Overview

While developing a *root filesystem* for a device, a developer needs to make frequent changes to the filesystem contents, like modifying scripts or adding newly compiled programs.

It isn’t practical at all to reflash the root filesystem on the target every time a change is made.<br/>
Fortunately, it is possible to set up networking between the development workstation and the target. Then, workstation files can be accessed by the target through the network, using NFS.

Unless you test a boot sequence, you no longer need to reboot the target to test the impact of script or application updates.

```console
$ LAB_PATH="$HOME/embedded-linux-qemu-labs/tinysystem"
```


## Kernel configuration

We'll re-use the kernel sources from the [kernel lab](kernel.md).

In the kernel configuration (see: [`menuconfig`](../kb/menuconfig.md)) verify that you have all the options needed for booting the system using a root filesystem mounted over NFS. Also enable the automatic mount of `devtmpfs`.

```console
$ cd "$LAB_PATH/../kernel/linux"
$ cp ../kernel.config .config
$ make menuconfig
```

In `File systems`:

* Enable `Network File Systems` (`NETWORK_FILESYSTEMS`), which should enable also the *client* for *v2* and *v3*.<br/>
  Inside this menu entry:

    * Enable `Root file system on NFS` (`ROOT_NFS`).

In `Device Drivers` &rarr; `Generic Driver Options`:

* Enable `Maintain a devtmpfs filesystem to mount at /dev` (`CONFIG_DEVTMPFS_MOUNT`).

Everything should already be as expected. If necessary, backup the new configuration and rebuild the kernel:

```console
$ cp .config "$LAB_PATH/kernel-busybox.config"
$ make
```


## NFS server

A NFS server is provided by the `nfs-kernel-server` package:

```console
$ sudo apt install nfs-kernel-server
```

Create a `nfsroot` directory in the current lab directory. We're going to use this folder as our NFS *root* folder.<br/>
If we were to add it directly, we would need to specify the whole path when requesting files from it, which might become verbose, and exposes our private paths to the public.
Instead, we can create a *symbolic link* at `/srv/nfs/`, and we can assign it the same `tftp` group of the previous labs:

```console
$ mkdir -p "$LAB_PATH/nfsroot"
$ sudo ln -snv "$LAB_PATH/nfsroot" /srv/nfs
'/srv/nfs' -> '/home/me/embedded-linux-qemu-labs/tinysystem/nfsroot'
$ sudo chown -R tftp:tftp /srv/nfs
```

Once installed, edit the `/etc/exports` file (as *root*) to export our `nfsroot` to the whole `10.0.2.x` subnet, with some specific options:

``` title="File: /etc/exports - line to append"
/srv/nfs 10.0.2.0/255.255.255.0(rw,no_root_squash,no_subtree_check)
```

A quick way is by appending that line to the file via a shell command.
In our example we use `tee` (we cannot `sudo echo "text" >> file`!):

```console
$ NET_IP="10.0.2.0"
$ NET_MASK="255.255.255.0"
$ line="$NFS_ROOT $NET_IP/$NET_MASK(rw,no_root_squash,no_subtree_check)"
$ echo $line | sudo tee -a /etc/exports
/srv/nfs 10.0.2.0/255.255.255.0(rw,no_root_squash,no_subtree_check)
```

Make sure that the path and the options are on the same line. Also make sure that there is **no space** between the IP address and the NFS options, otherwise default options will be used for this IP address, causing your root filesystem to be read-only.

Then, restart the NFS server with the new configuration:

```console
$ sudo exportfs -ar
$ sudo systemctl restart nfs-kernel-server
```


## Booting the system

First, boot the board to the *U-Boot* prompt (press a key before the timeout).

```console
$ cd "$LAB_PATH/../bootloader"
$ ./qemu
```

Before booting the kernel, we need to tell it that the root filesystem should be mounted over NFS, by setting some kernel parameters.<br/>
So, add the required settings to the `bootargs` environment variable (on a single long line!), and save it permanently:

``` title="QEMU - U-Boot"
=> setenv netmask 255.255.255.0
=> setenv servernfs /srv/nfs
=> setenv bootargs console=ttyAMA0 root=/dev/nfs ip=${ipaddr}::${serverip}:${netmask}:: nfsroot=${serverip}:${servernfs},nfsvers=3,tcp rw
=> printenv bootargs
bootargs=console=ttyAMA0 root=/dev/nfs ip=10.0.2.69::10.0.2.15:255.255.255.0:: nfsroot=10.0.2.15:/srv/nfs,nfsvers=3,tcp rw
=> saveenv
```

* `console`: the debug console.

* `root`: mounting to the standard virtual device for NFS.

* `ip`: a detailed string with all the network parameters, for the maximum portability.

* `nfsroot`: the path to the NFS folder on the server, as seen form the net, with the specified protocol.

* `rw`: read and write permissions.

Now, reboot your system. The kernel should be able to mount the root filesystem over NFS (it might take some time to get there).

``` title="QEMU - U-Boot & Kernel" hl_lines="3"
=> reset
    ...
VFS: Mounted root (nfs filesystem) on device 0:14.
    ...
```

If the kernel fails to mount the NFS filesystem, look carefully at the error messages in the console.<br/>
If this doesn’t give any clue, you can also have a look at the NFS server logs in `/var/log/syslog`.

However, at this stage, the kernel should stop because of the following issue:

``` title="QEMU - Kernel" hl_lines="3"
    ...
VFS: Mounted root (nfs filesystem) on device 0:14.
devtmpfs: error mounting -2
    ...
---[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance. ]---
```

This happens because the kernel is trying to mount the devtmpfs filesystem in `/dev/` in the root filesystem.
This virtual filesystem contains device files for all the devices known to the kernel, and with `CONFIG_DEVTMPFS_MOUNT`, our kernel tries to automatically mount `devtmpfs` on `/dev`.

To address this, just create a `dev` directory under `nfsroot`:

```console
$ mkdir -p "$LAB_PATH/nfsroot/dev"
```

Now restart *QEMU*. The kernel should complain for the last time, saying that it can’t find an `init` application:

``` title="QEMU - Kernel" hl_lines="5"
    ...
VFS: Mounted root (nfs filesystem) on device 0:14.
devtmpfs: mounted
    ...
---[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance. ]---
```

Obviously, our *root* filesystem being mostly empty, there isn’t such an application yet.
In the next paragraph, you will add *BusyBox* to your root filesystem, and finally make it usable.

You might want to create a spare backup copy of your virtual SD image now:

```console
$ cd "$LAB_PATH/../bootloader/"
$ tar cfJv "$LAB_PATH/nfs-boot-sd.img.tar.xz" sd.img
```


## Root filesystem with BusyBox

Download the sources of *BusyBox* release `1.35.0`:

```console
$ cd $LAB_PATH
$ label="1_35_0"
$ git clone https://git.busybox.net/busybox
$ cd busybox/
$ git checkout -b embedded-linux-qemu $label
```

Alternatively, you can get a source code archive:

```console
$ cd $LAB_PATH
$ label="1_35_0"
$ wget "https://git.busybox.net/busybox/snapshot/busybox-${label}.tar.bz2"
$ tar xfv "busybox-${label}.tar.bz2"
$ mv busybox*/ busybox
$ cd busybox/
```

Now, configure *BusyBox* with the configuration file provided in the `data/` directory.<br/>
Then, you can run `make menuconfig` to further customize the configuration.
At least, keep the setting that builds a static *BusyBox*. Compiling it statically in the first place makes it easy to set up the system, because there are no dependencies. Later on, we will set up shared libraries and recompile *BusyBox*.

```console
$ cp "$LAB_PATH/data/busybox-1.35.config" .config
$ make menuconfig
```

Build BusyBox using the toolchain that you used to build the kernel.

```console
$ TC_NAME="arm-training-linux-uclibcgnueabihf"
$ TC_BASE="$HOME/x-tools/${TC_NAME}"
$ export PATH="$TC_BASE/bin:$PATH"
$ export CROSS_COMPILE=arm-linux-
$ export MAKEFLAGS=-j$(nproc)
$ make
```

Going back to the *BusyBox* configuration interface, check the installation directory. Set it to the path to your `nfsroot` directory if necessary.

```console
$ make menuconfig
```

`Settings` &rarr; `Install Options` &rarr; `Destination path for 'make install'` (`PREFIX`) = `../nfsroot`

Now run `make install` to install *BusyBox* in this directory.

```console
$ cp .config ../busybox-static.config
$ make install
```

Try to boot your new system on the board. You should now reach a command line prompt, allowing you to execute the commands of your choice.

```console
$ cd "$LAB_PATH/../bootloader/"
$ ./qemu
```

``` title="QEMU - BusyBox" hl_lines="4"
    ...
can't run '/etc/init.d/rcS': No such file or directory

Please press Enter to activate this console.
```

You can press ++enter++ to enter a *root* login shell.
Ignore any warning messages for now.


## Virtual filesystems

Within the target shell, run the `ps` command. You can see that it complains that the `/proc` directory does not exist. The `ps` command and other process-related commands use the `proc` *virtual filesystem* to
get their information from the kernel.

```console title="QEMU - BusyBox" hl_lines="3"
# ps
  PID USER       VSZ STAT COMMAND
ps: can't open '/proc': No such file or directory
```

From the command line of the target, create the `proc`, `sys` and `etc` directories in your
*root* filesystem:

```console title="QEMU - BusyBox"
# cd /
# mkdir proc sys etc
```

Now mount the `proc` virtual filesystem. Now that `/proc` is available, test again the `ps` command.

```console title="QEMU - BusyBox"
# mount -t proc proc /proc
# ps
  PID USER       VSZ STAT COMMAND
    1 0          512 S    /sbin/init
    2 0            0 SW   [kthreadd]
    3 0            0 IW<  [rcu_gp]
    4 0            0 IW<  [rcu_par_gp]
    5 0            0 IW<  [slub_flushwq]
    ...
```

Note that you can also now *halt* your target with the `halt` command, thanks to `proc` being mounted.<br/>
The `halt` command can find the list of mounted filesystems in `/proc/mounts`, and *unmount* them in a clean way before shutting down.

```console title="QEMU - BusyBox" hl_lines="12"
# halt
starting pid 78, tty '': 'umount -a -r'
umount: devtmpfs busy - remounted read-only
starting pid 79, tty '': 'swapoff -a'
swapoff: can't open '/etc/fstab': No such file or directory
The system is going down NOW!
Sent SIGTERM to all processes
Sent SIGKILL to all processes
Requesting system halt
Flash device refused suspend due to active operation (state 20)
Flash device refused suspend due to active operation (state 20)
reboot: System halted
```

You can now quit *QEMU* as usual (++ctrl+a++ then ++x++), this time more safely.


## System configuration and startup

The first user space program that gets executed by the kernel is `/sbin/init`, whose configuration
file is `/etc/inittab`.

In the *BusyBox* sources, read details about `/etc/inittab` in the `examples/inittab` file (press ++q++ to quit from `less`). Let's create it from the example template; we'll tweak it soon.<br>
Perform some housekeeping with permissions, since we messed with the *root* shell from *QEMU*.

```console
$ cd $LAB_PATH/nfsroot/
$ less ../busybox/examples/inittab
$ sudo chown -R tftp:tftp .
$ sudo chmod o-w *
$ cp ../busybox/examples/inittab etc/inittab
$ nano etc/inittab
```

Comment out any *getty respawn* from `etc/inittab`, otherwise those shells would keep respawning in our emulated board. Save (++ctrl+o++) and exit (++ctrl+x++).

```sh title="File: $LAB_PATH/nfsroot/etc/inittab - respawning getty commented out" hl_lines="3-4"
    ...
# /sbin/getty invocations for selected ttys
#tty4::respawn:/sbin/getty 38400 tty5
#tty5::respawn:/sbin/getty 38400 tty6
    ...
```

If you enter the login shell after startup, you should get an annoying message:

```console title="QEMU - BusyBox" hl_lines="8"
Please press Enter to activate this console.
starting pid 59, tty '': '-/bin/sh'


BusyBox v1.35.0 (2023-04-08 14:55:52 CEST) built-in shell (ash)
Enter 'help' for a list of built-in commands.

-/bin/sh: can't access tty; job control turned off
#
```

Without *job control*, we cannot manage jobs, like termination via ++ctrl+c++.<br/>
A quick way is to force `ttyAMA0` (the emulated board debug serial port) as the default login shell:

```sh title="File: $LAB_PATH/nfsroot/etc/inittab - forced login shell device" hl_lines="3"
    ...
# Start an "askfirst" shell on the console (whatever that may be)
ttyAMA0::askfirst:-/bin/sh
    ...
```

> **TODO**: Use some command line tools to comment out the above lines.

Create the standard `/etc/init.d/rcS` startup script, to mount the `/proc` and `/sys` filesystems.

```sh title="File: $LAB_PATH/nfsroot/etc/init.d/rcS"
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sys /sys
```

Quick typing from the shell:

```console
$ mkdir -p etc/init.d
$ cat > etc/init.d/rcS <<'EOF'
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sys /sys
EOF
$ chmod +x etc/init.d/rcS
```

Try again with *QEMU*, and you can notice that those warning messages are now gone.

```console title="QEMU - BusyBox"
VFS: Mounted root (nfs filesystem) on device 0:14.
devtmpfs: mounted
Freeing unused kernel image (initmem) memory: 1024K
Run /sbin/init as init process
starting pid 60, tty '': '/etc/init.d/rcS'

Please press Enter to activate this console.
starting pid 63, tty '/dev/ttyAMA0': '-/bin/sh'


BusyBox v1.35.0 (2023-04-08 14:55:52 CEST) built-in shell (ash)
Enter 'help' for a list of built-in commands.

#
```

You can keep the current *QEMU* instance running for the next paragraphs.
We're going to change files on-the-fly; those changes can be accessed directly from the *QEMU* instance via NFS.

You might want to backup the current `nfsroot`. We're going to use `cpio` to preserve *symlinks*.

```console
$ cd $LAB_PATH/nfsroot/
$ find . -depth -print0 | cpio -ocv0 | xz > "$LAB_PATH/nfsroot-static.cpio.xz"
```


## Switching to shared libraries

Take the `hello.c` program supplied in the lab `data` directory. Cross-compile it for ARM, dynamically-linked with the libraries (just use our `arm-linux` toolchain), and run it on the target.<br/>
You should face a very misleading `not found` error, which is not because the `hello` executable is not found, but because something else was not found while trying to execute this executable.

```console
$ cd $LAB_PATH/data/
$ arm-linux-gcc -o hello hello.c
$ cp hello ../nfsroot/bin/
```

```console title="QEMU - BusyBox"
# hello
-/bin/sh: hello: not found
```

It’s missing the `ld-uClibc.so.0` executable, which is the dynamic linker required to execute any program compiled with shared libraries.<br/>
Using the `find` command, look for any library files in the toolchain install directory, and copy them to the `lib/` directories on the target.
We're going to use a *regex* to find all the matches for possible *shared object* file name extensions.<br/>
Also copy any binary executables (like `ldd`) to their respective fodlers.

```console
$ cd "$TC_BASE/$TC_NAME/sysroot/"
$ so_regex=".+\.so\(\.[0-9]+\)*"
$ find . -regex $so_regex | sort
./lib/ld-uClibc-1.0.39.so
./lib/ld-uClibc.so.0
./lib/ld-uClibc.so.1
./lib/libatomic.so
./lib/libatomic.so.1
./lib/libatomic.so.1.2.0
./lib/libc.so.0
./lib/libc.so.1
./lib/libgcc_s.so
./lib/libgcc_s.so.1
./lib/libitm.so
./lib/libitm.so.1
./lib/libitm.so.1.0.0
./lib/libstdc++.so
./lib/libstdc++.so.6
./lib/libstdc++.so.6.0.29
./lib/libthread_db-1.0.39.so
./lib/libthread_db.so.1
./lib/libuClibc-1.0.39.so
./usr/lib/libc.so
./usr/lib/libthread_db.so
$ mkdir -p "$LAB_PATH/nfsroot/lib/"
$ mkdir -p "$LAB_PATH/nfsroot/usr/lib/"
$ mkdir -p "$LAB_PATH/nfsroot/sbin/"
$ mkdir -p "$LAB_PATH/nfsroot/usr/sbin/"
$ cp lib/libc.so* "$LAB_PATH/nfsroot/lib/"
$ cp lib/ld-uClibc.so* "$LAB_PATH/nfsroot/lib/"
$ cp usr/bin/ldd "$LAB_PATH/nfsroot/usr/bin/"
```

Now `hello` works as expected, and you can also execute `ldd` against it to see the library dependencies:

```console title="QEMU - BusyBox"
# hello
Hello world!
# ldd bin/hello
        libc.so.0 => /lib/libc.so.0 (0x76e83000)
        ld-uClibc.so.1 => /lib/ld-uClibc.so.0 (0x76f0a000)
# ldd bin/busybox
        not a dynamic executable
```

> If you still get the same error message, work, just try again a few seconds later.
> Such a delay can be needed because the NFS client can take a little time (at most 30-60 seconds) before seeing the changes made on the NFS server.

Once the small test program works, we're going to recompile *BusyBox* without the *static* compilation option, so that *BusyBox* takes advantages of the shared libraries that are now present on the target.<br>
Before doing that, measure the size of the busybox executable.

```console
$ cd $LAB_PATH
$ du -h nfsroot/bin/busybox
360K    nfsroot/bin/busybox
$ cd busybox/
$ make menuconfig
```

In `Settings`:

* Disable `Build static binary (no shared libs)` (`STATIC`)

Now `halt` and quit *QEMU*, backup the new configuration, and build *BusyBox* again.<br/>
As you will see, the executable is now smaller, because it's using shared libraries.

```console
$ cp .config ../busybox-dynamic.config
$ make clean
$ make install
$ du -h ../nfsroot/bin/busybox
216K    ../nfsroot/bin/busybox
```

Launch *QEMU* again, reaching the *BusyBox* shell successfully.<br/>
You can confirm that the current `busybox` executable depends on shared libraries:

```console title="QEMU - BusyBox"
# ldd bin/busybox
        libc.so.0 => /lib/libc.so.0 (0x76f1e000)
        ld-uClibc.so.1 => /lib/ld-uClibc.so.0 (0x76fa5000)
```


## Implement a web interface for your device

Replicate `$LAB_PATH/data/www/` to the `/www` directory in your target *root* filesystem.

```console
$ cd $LAB_PATH
$ cp -r data/www/ nfsroot/
```

Then, run the *BusyBox* *http server* from the *BusyBox* shell; it will automatically background itself.

``` title="QEMU - BusyBox"
# /usr/sbin/httpd -h /www/
```

Now, test that your web interface works well by opening
[http://10.0.2.69/index.html](http://10.0.2.69/index.html)
within the host machine (*Lubuntu* VM).

See how the dynamic pages are implemented. Very simple, isn’t it?

> If you use a proxy, configure your host browser so that it doesn’t go through the proxy to connect to the target IP address, or simply disable proxy usage.

Finish by adding the command that starts the web server to your startup script, so that it is always started on your target.

```console
$ cd "$LAB_PATH/nfsroot/"
$ echo "/usr/sbin/httpd -h /www/" >> etc/init.d/rcS
```

You can `reboot` *BusyBox* and see that the web server was started automatically (press ++ctrl+c++ to quit `top`). You can then `halt` *QEMU*.

```console title="QEMU - BusyBox" hl_lines="13"
# top
Mem: 14136K used, 104336K free, 0K shrd, 0K buff, 868K cached
CPU:  0.5% usr  1.5% sys  0.0% nic 97.9% idle  0.0% io  0.0% irq  0.0% sirq
Load average: 0.00 0.00 0.00 1/59 66
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
   66    62 0        R     1016  0.8   0  1.7 top
   31     2 0        IW       0  0.0   0  0.3 [kworker/0:1-eve]
    1     0 0        S     1016  0.8   0  0.0 /sbin/init
   63     1 0        S     1016  0.8   0  0.0 /sbin/init
   64     1 0        S     1016  0.8   0  0.0 /sbin/init
   65     1 0        S     1016  0.8   0  0.0 /sbin/init
   62     1 0        S     1012  0.8   0  0.0 -/bin/sh
   61     1 0        S     1008  0.8   0  0.0 /usr/sbin/httpd -h /www/
    8     2 0        IW       0  0.0   0  0.0 [kworker/u8:0-nf]
   10     2 0        SW       0  0.0   0  0.0 [ksoftirqd/0]
   11     2 0        IW       0  0.0   0  0.0 [rcu_sched]
   55     2 0        IW<      0  0.0   0  0.0 [kworker/u9:2-xp]
   41     2 0        IW<      0  0.0   0  0.0 [kworker/u9:0-xp]
   46     2 0        IW       0  0.0   0  0.0 [kworker/0:2-eve]
   29     2 0        SW       0  0.0   0  0.0 [kdevtmpfs]
   35     2 0        SW       0  0.0   0  0.0 [kcompactd0]
   43     2 0        IW       0  0.0   0  0.0 [kworker/u8:2-ev]
    2     0 0        SW       0  0.0   0  0.0 [kthreadd]
   38     2 0        IW       0  0.0   0  0.0 [kworker/u8:1-nf]
    6     2 0        IW       0  0.0   0  0.0 [kworker/0:0-eve]
```


## Backup and restore

Everything looks fine now, so you're free to make a backup archive of the dynamic `nfsroot`:

```console
$ cd "$LAB_PATH/nfsroot/"
$ find . -depth -print0 | cpio -ocv0 | xz > "$LAB_PATH/nfsroot-dynamic.cpio.xz"
```

In case you wish to restore the snapshot:

```console
$ cd $LAB_PATH
$ rm -rf nfsroot/
$ mkdir -p nfsroot/
$ xzcat nfsroot-dynamic.cpio.xz | cpio -iduv -D nfsroot/
```


## Boot with *initramfs*

It's usual to run an *initramfs* instead of a boot from NFS: the *initramfs* copies data from the SD card into RAM, and executes a preliminary boot from there.

> FYI: [https://landley.net/writing/rootfs-howto.html](https://landley.net/writing/rootfs-howto.html)

Configure your kernel to include the contents of the `nfsroot` directory as an *initramfs* image archive file `/initrd.cpio.gz`.

```console
$ cd "$LAB_PATH/../kernel/linux/"
$ export ARCH=arm
$ make menuconfig
$ cp .config "$LAB_PATH/kernel-initramfs.config"
```

In `General setup`:

* Enable `Initial RAM filesystem and RAM disk (initramfs/initrd) support` (`BLK_DEV_INITRD`)

* Set `Initramfs source file(s)` (`INITRAMFS_SOURCE`) = `../../tinysystem/nfsroot`.

* Enable `Support initial ramdisk/ramfs compressed using gzip` (`RD_GZIP`).

Now `<Save>` the `.config` and save a backup copy.

Before building and running the updated kernel, in the toplevel directory you have to create an `/init` link pointing to `/sbin/init`.
This is required because the kernel will try to execute the `/init` executable &mdash; we simply redirect it to the standard one.<br/>
You can then archive your *initrams* image archive.

```console
$ cd "$LAB_PATH/nfsroot/"
$ ln -s sbin/init init
```

You should also mount `devtmpfs` from the `/etc/init.d/rcS` script, because it cannot be mounted automatically by the kernel when booting from an *initramfs*.

```console
$ cat > etc/init.d/rcS <<'EOF'
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devtmpfs dev /dev
/usr/sbin/httpd -h /www/
EOF
```

You can now rebuild the kernel, `make` will take care of the changes to rebuild.<br/>
Copy the newly generated kernel image to the TFTP server. You can see the difference in size between the two `zImage` versions, since *initramfs* includes our `nfsroot`.

```console
$ cd "$LAB_PATH/../kernel/linux/"
$ make
    ...
  Kernel: arch/arm/boot/zImage is ready
$ mv /srv/tftp/zImage /srv/tftp/zImage-without-initramfs
$ cp arch/arm/boot/zImage /srv/tftp/zImage-with-initramfs
$ cp arch/arm/boot/zImage /srv/tftp/zImage
$ du -h /srv/tftp/zImage*
5.6M    /srv/tftp/zImage
5.6M    /srv/tftp/zImage-with-initramfs
4.8M    /srv/tftp/zImage-without-initramfs
```

You can now launch the *QEMU* machine, which this time is booting the kernel from *initramfs* instead of NFS.<br/>
You won’t need to modify your `root=/srv/nfs` setting in the kernel command line (`bootargs`), because it will just be ignored for an *initramfs*.

```console title="QEMU - BusyBox" hl_lines="2-3"
    ...
Freeing unused kernel image (initmem) memory: 2048K
Run /init as init process

Please press Enter to activate this console.
```

Let's archive the current state for the future.

```console
$ cd "$LAB_PATH/nfsroot/"
$ find . -depth -print0 | cpio -ocv0 | xz > "$LAB_PATH/nfsroot-initramfs.cpio.xz"
$ cd /srv/tftp
$ tar cfJv "$LAB_PATH/zImage-initramfs.tar.xz" zImage
```

In case you wish to restore the snapshot:

```console
$ cd $LAB_PATH
$ rm -rf nfsroot/
$ mkdir -p nfsroot/
$ xzcat nfsroot-initramfs.cpio.xz | cpio -iduv -D nfsroot/
$ tar xfv zImage-initramfs.tar.xz
$ mv zImage /srv/tftp
```

Now go back to booting the system through NFS, which is more convenient for these labs: an *initiramfs* requires to be rebuilt for each change in your *root* filesystem, while NFS does it automatically.

```console
$ cp /srv/tftp/zImage-without-initramfs /srv/tftp/zImage
```


## Licensing

This document is an extension to: [*Embedded Linux System Development - Practical Labs - QEMU Variant*](https://bootlin.com/doc/training/embedded-linux-qemu/)
 &mdash; &copy; 2004-2023, *Bootlin* [https://bootlin.com/](https://bootlin.com), [`CC-BY-SA-3.0`]((https://creativecommons.org/licenses/by-sa/3.0/)) license.
