# System integration


## Objectives

* Get familiar with the `systemd` init system.

Compared to the previous lab, we go on increasing the complexity of the system, this time by using the systemd init system, and by taking advantage of it to add a few extra features, in particular ones that will be useful for debugging in the next lab.


## Setup

Since `systemd` requires the GNU C library, we are going to make a new *Buildroot* build in a new working directory, and using a different cross-compiling toolchain.

So, create the `$HOME/embedded-linux-bbb-labs/integration/` directory and go inside of it.

```console
$ LAB_PATH="$HOME/embedded-linux-bbb-labs/integration"
$ mkdir -p $LAB_PATH
$ cd $LAB_PATH
```

Make a new clone of *Buildroot* from the existing local *git* repository, and checkout our `embedded-linux-bbb` branch:

```console
$ git clone ../buildroot/buildroot/
$ cd buildroot/
$ git switch embedded-linux-bbb
```


## Root filesystem overlay

Remove `etc/init.d/` from the *root* filesystem overlay, because it was for *BusyBox*, not *systemd*.

```console
$ rm -rf board/bootlin/training/rootfs-overlay/etc/init.d/
```

## Buildroot configuration

Let's make a new *Buildroot* configuration from scratch.

```console
$ cd "$LAB_PATH/buildroot/"
$ make distclean
$ make menuconfig
$ cp .config ../buildroot.config
```


In `Target options`:

* `Target Architecture` = `ARM (little endian)`.

* `Target Architecture Variant` = `cortex-A8`.

* `Target ABI` = `EABIhf`.

* `Floating point strategy` = `VFPv3-D16`.


In `Toolchain`:

* `Toolchain type` = `External toolchain`.

* `Toolchain` = `Bootlin toolchains`.<br/>
  This time, we will use a *Bootlin* ready-made toolchain for `glibc`, as this is necessary for using *systemd*.

* `Toolchain origin` = `Toolchain to be downloaded and installed`.

* `Bootlin toolchain variant` = `armv7-eabihf glibc bleeding-edge 2021.11-1`.

* Enable `Copy gdb server to the Target`.


In `System configuration`:

* `Init system` = `systemd`.

* `Root filesystem overlay directories` = `board/bootlin/training/rootfs-overlay`.


In `Kernel`:

* Enable `Linux Kernel`.

* `Kernel version` = `Latest version (5.15)`.

* `Custom kernel patches` = `board/bootlin/training/0001-Custom-DTS-for-Bootlinlab.patch`.

* `Kernel configuration` = `Using a custom (def)config file`.

* `Configuration file path` = `board/bootlin/training/linux.config`.

* Enable `Build a Device Tree Blob (DTB)`.

* `In-tree Device Tree Source file names` = `am335x-boneblack-custom`.


In `Target packages`:

* `Audio and video applications`:

    * Enable `mpd`, and in the submenu:

        * Keep only `alsa`, `vorbis`, and `tcp sockets`.

    * Enable `mpd-mpc`.

* `Hardware handling`:

    * Enable `nunchuk driver`.

* `Networking applications`:

    * Enable `dropbear`, a lightweight *SSH* server used instead of *OpenSSH* in most embedded devices.<br/>
      Disable `client programs`, which are not needed.


In `Filesystem images`:

* Enable `tar the root filesystem`.


## Build and test

Now build the full system.

```console
$ make
    ...
```

Once the build is over, generate the dependency graph again and find out the new dependencies
introduced by using systemd.

```
$ make graph-depends
    ...
$ evince output/graphs/graph-depends.pdf
$ cp output/graphs/graph-depends.pdf ../graph-depends.pdf
```

To test the new system, create a new `nfsroot` directory, extract then new *root* filesystem into it, and boot your board on it through NFS.

```console
$ mkdir -p "$LAB_PATH/nfsroot/"
$ cd "$LAB_PATH/nfsroot/"
$ tar xfv "../buildroot/output/images/rootfs.tar"
$ sudo rm -f /srv/nfs
$ sudo ln -snv "$LAB_PATH/nfsroot/" /srv/nfs
'/srv/nfs' -> '/home/me/embedded-linux-bbb-labs/integration/nfsroot/'
$ sudo chown -R tftp:tftp /srv/nfs
$ sudo exportfs -ar
$ sudo systemctl restart nfs-kernel-server
```

You should see the system booting through *systemd*, with all the *systemd* targets and system services starting one by one, with a total boot time which looks slower than before.<br/>
That's because the system configuration is more complex, but also more versatile, being ready to run more complex services and applications.

You can ask systemd to show you the various services which were started:

```console
$ systemctl status
    **TODO**
```

You can also check all the mounted filesystems and be impressed:

```console
# mount
    **TODO**
```


## Inspecting the system

On the *target*, look at the contents of `/lib/systemd/`.
You will see the implementation of most *systemd* targets and services.

```console title="picocomBBB - systemd"
# ls /lib/systemd/
    **TODO**
```

In particular, check out `/lib/systemd/user/`, containing some unnecessary targets in our case, such as `bluetooth.target`.

```console title="picocomBBB - systemd"
# ls /lib/systemd/user/
    **TODO**
```

However, check the `mpd.service` file for our *MPD* server.
This should help you to realize all the options provided by *systemd* to start and control system services, while keeping the system secure, and their resources under control.
You won't be able to match this level of control and security in a *hand-made* system.

```console title="picocomBBB - systemd"
# less /lib/systemd/user/mpd.service
    **TODO**
```


## Automatic module loading

Check the currently loaded modules on your system.<br/>

```console title="picocomBBB - systemd"
# lsmod
    **TODO**
```

Surprise: both the *Nunchuk* and USB audio modules are already loaded.
We didn't have anything to set up and *systemd* automatically load the modules associated to connected hardware.<br/>
Let's find out why.

On the target, go to `/lib/udev/rules.d/`.
You will find all the standard rules for *Udev*, the part of *systemd* which handles hardware events, takes care of the permissions and ownership of device files, notifies other userspace programs, and among others, loads kernel modules.

Open `80-drivers.rules`, which is the rule allowing *Udev* to load kernel modules for detected devices.<br/>
Here is its most important line:

```sh title="File: 80-drivers.rules - MODALIAS line"
ENV{MODALIAS}=="?*", RUN{builtin}+="kmod load '$env{MODALIAS}'"
```

This is when the `modules.alias` file comes into play.<br/>
When a new device is found, the kernel passes a `MODALIAS` environment variable to *Udev*, containing which bus this happened on, and the attributes of the device on this bus.
Thanks to the module aliases, the right module gets loaded.
We already explained that [in the lectures](hardware.md) when talking about the output of `make modules_install`.

Find where the `modules.alias` file is located, and you will find the two lines that allowed to load
our `snd_usb_audio` and `nunchuk` modules:

``` title="File: modules.alias"
    ...
alias usb:v*p*d*dc*dsc*dp*ic01isc01ip*in* snd_usb_audio
alias usb:v2B53p0031d*dc*dsc*dp*ic*isc*ip*in* snd_usb_audio
    ...
alias of:N*T*Cnintendo,nunchuk nunchuk
    ...
```

For `snd_usb_audio`, there are many possible matching values, so it isn't straighforward to be
sure which matched your particular device.

However, you can find in *sysfs* which `MODALIAS` was emitted for your device:

```console title="picocomBBB - systemd"
# cd /sys/class/sound/card0/device
# ls -la
# cat modalias
usb:v1B3Fp2008d0100dc00dsc00dp00ic01isc01ip00in00
```

With a bit of patience, you could find the matching linewithin the `modules.alias` file.

If you want to see the information sent to Udev* by the kernel when a new device is plugged in, here are a few debugging commands.

First unplug your device and run:

```console title="picocomBBB - systemd"
# udevadm monitor
    **TOD**
```

Then plug in your headset again. You will find all the events emitted by the kernel, and with the
same string (with `UDEV` instead of `KERNEL`), the time when *Udev* finished processing each event.

```console title="picocomBBB - systemd"
    **TODO**
```

You can also see the MODALIAS values carried by these events:

```console title="picocomBBB - systemd"
# udevadm monitor --env
    **TODO**
```

As far as the *Nunchuk* is concerned, we cannot easily remove it from the *Device Tree* and add it back, but it's easier to find its `MODALIAS` value:

```console title="picocomBBB - systemd"
# cd /sys/bus/i2c/devices/
# ls -la
    **TODO**
```

Here you will recognize our *Nunchuk* device through its `0x52` address.

```console title="picocomBBB - systemd"
# cd 1-0052
# ls -la
# cat modalias
of:NjoystickT(null)Cnintendo,nunchuk
```

Here the bus is `of`, meaning *Open Firmware*, which was the former name of the *Device Tree*.<br/>
When an event was emitted by the kernel with this `MODALIAS` string, the `nunchuk` module got loaded by *Udev* thanks to the matching alias.

This actually happened when *systemd* ran the *coldplugging* operation: at system startup, it asked the kernel to emit *hotplug* events for devices already present when the system booted:

```
[ OK ] Finished Coldplug All udev Devices.
```

On non-*x86* platforms, that's typically for devices described in the *Device Tree*.<br/>
This way, both *static* and *hotplugged* devices can be handled in the same way, using the same *Udev* rules.


## Testing

Make sure that audio playback still works on your system:

```console title="picocomBBB - systemd"
# mpc update
# mpc add /
# mpc play
```

If it doesn't, look at the *systemd* logs in your serial console history (`dmesg`).
*systemd* should let you know about the failing services and the commands to run to get more details.
