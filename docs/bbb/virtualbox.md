# Host VM

**TODO: THIS DOCUMENT IS STILL JUST A DRAFT FROM THE QEMU VARIANT!**

I chose to use a *virtual machine* as the *host machine*, so that I won't mess with my physical machines.
Moreover, I can bring it around on an USB hard disk!

I chose
[**VirtualBox**](https://www.virtualbox.org)
as my VM, simply because it's a freely available VM I've used for a long time.
My current version is **7.0.6**, but it shouldn't matter much as far as I know.
I installed this VirtualBox on several *Windows 10* machines successfully; this is the *VM host OS*.

As the *VM guest OS* I chose
[**Lubuntu 22.04 LTS**](https://lubuntu.me/).
I know that many prefer a standard [*Debian*](https://www.debian.org/),
but I find Ubuntu easier to setup &mdash; I don't want to waste too much time configuring  the host machine.

Lubuntu is rather lightweight and quick, with no frills, while still being an officially supported Linux distro.

The LTS was chosen because it's in the middle of its planned life, so it's mature enough to get stable results for enough time.

Furthermore, 22.04 LTS is officially supported by several tools and frameworks, including
[*Yocto*](https://docs.yoctoproject.org/ref-manual/system-requirements.html).


## Required Tools

* [VirtualBox 7.0.6](https://www.virtualbox.org/wiki/Downloads)
  or later

* [Lubuntu 22.04 LTS (*Jammy Jellyfish*)](https://cdimage.ubuntu.com/lubuntu/releases/22.04/release/)
  or another *Ubuntu 22.04 LTS*

* [Training lab data](https://bootlin.com/doc/training/embedded-linux-bbb/embedded-linux-bbb-labs.tar.xz)


## VM guest OS installation

First, install *VirtualBox*, which should be very straightforward.

Then, create a new virtual machine.

Important VM specs:

* Hard Disk &ge; 32 GB
    * *Dynamically allocated* is fine
    * Enable `Use Host I/O cache` to avoid hiccups

* CD drive with the Lubuntu *ISO file* as a `Live CD/DVD`

* RAM &ge; 4 GB
    * With less RAM, some toolchains might get stuck in *swap hell*

* Processors = number of physical CPU cores
    * VM host OS still usable
    * good performance
    * might not leverage *hyperthreading*
    * I'm using `Enable PAE/NX` as an added bonus
    * suggested &ge; 4 *physical* cores of the host machine

* Network via *NAT*
    * No need for a proper network as with physical boards

Launch the VM and wait for it to load the *Live* OS straight from the emulated DVD drive.

When available, let's run the installer from the desktop.
No particular options have to be set, so leave the defauit ones if in doubt.

When prompted for formatting options, just let the installer format the whole partition with *swap* on file.

When asked for names and passwords, I'd opt for covnenience:

* Machine name: `vm`
* User name: `me`
* Password: *none*
* Automatic login: *enabled*

After a few minutes the installer should have terminated, and the VM can be restarted to enter the installed *VM guest OS*.


## VM guest OS configuration

Once the installed *VM guest OS* is run, it should install any updates from the internet.

For shell commands I typically use the default terminal application provided by the OS.
In the case of Lubuntu, it's *QTerminal*, which can be accessed either via the *start menu*, or by pressing ++ctrl+alt+t++.

So, let's update the system, with the canonical commands for *Debian*-based Linux distros:

```console
$ sudo apt update
$ sudo apt dist-upgrade
```

It most probably installed some Linux kernel updates, so let's reboot to make changes effective:

```console
$ sudo reboot
```

This time it's best to add *VirtualBox Guest Additions* for the best VM experience.

From the `Devices` menu, select `Insert Guest Additions CD Image...`.
Ignore any automatic actions; we're going to install them from the shell:

```console
$ cd /media/me/VBox_GAs_7.0.6/
$ sudo ./VBoxLinuxAdditions.run
```

> You can usually type `cd /media` and press the ++tab++ key repeatedly for automatic completion; the same for the run script.
>
> If you double-press ++tab++, the shell prints some suggestions. It's often handy to click the desired suggestion with the central mouse button to automatically append it to the command line.

After the installation has finished, it's best to reboot again.

You're now free to resize the window, or enter *seamless mode*.
I usually select the latter, so that I can open a PDF in the host OS, and superimpose Lubuntu shell windows seamlessly.


## Networking

The *Bootlin* course relies on the *virtual Ethernet* provided by the *BBB*.
This allows the course support both the standard *BeagleBone Black*, as well as the *BeagleBone Black Wireless*.

Instead, this personal variant of the course adopts just the standard *BBB*, so I'm allowed to use the dedicated *RJ45 Ethernet* connection, which is more akin to an actual connected product.<br/>
This requires some more setup on the *host* side, because the *BBB* is going to be an actual member of our local network, as well as the *VM guest OS*.

The conventions for this tutorial are the following:

* Network mask: `255.255.255.0` = `/24`, the 24 most significant bits.
* *Host* machine IP: `192.168.0.15`, the *VM guest OS*.
* *Target* machine IP: `192.168.0.69`, the *BeagleBone Black* board.

Of course, you're free to change them accordingly to your preferences and network availability &mdash; just avoid *DHCP* for our *host* and *target* machines!

To help being more consistent across the course, I use some environment variables, so that you don't need to change too many command lines and scripts within the course itself.

```console
$ NET_MASK="255.255.255.0"
$ NET_IP="192.168.0.0"
$ HOST_IP="192.168.0.15"
$ TARGET_IP="192.168.0.69"
```

An important point is that you should change the *VirtualBox guest* network interface to be *bridged*, so that the *VM guest OS* acts as an actual node of your LAN.

For *Lubuntu* VM, you can edit the network settings with the graphical *Network Manager*.

Make sure your *VM guest OS* can reach the internet in a fast and reliable way.


## Training lab data

You should retrieve the training lab data from its *Bootlin* page:

```console
$ cd ~
$ wget https://bootlin.com/doc/training/embedded-linux-bbb/embedded-linux-bbb-labs.tar.xz
$ tar xfv embedded-linux-bbb-labs.tar.xz
```

You can give a quick look to the folder structure with the `tree` command:

```console
$ tree ~/embedded-linux-bbb-labs/
/home/me/embedded-linux-bbb-labs/
├── appdev
│   ├── nunchuk-mpd-client.c
│   └── prep-debug.sh
├── bootloader
│   └── data
│       ├── MLO
│       ├── MLO-2022.07
│       ├── u-boot-2022.07.img
│       └── u-boot.img
├── buildroot
│   └── data
│       ├── mpd.conf
│       └── music
│           ├── 1-sample.ogg
│           ├── 2-arpent.ogg
│           ├── 3-chronos.ogg
│           ├── 4-land-of-pirates.ogg
│           ├── 5-ukulele-song.ogg
│           ├── 6-le-baguette.ogg
│           ├── 7-fireworks.ogg
│           └── README.txt
├── hardware
│   └── data
│       └── nunchuk
│           ├── Makefile
│           └── nunchuk.c
├── tinysystem
│   └── data
│       ├── busybox-1.35.config
│       ├── hello.c
│       └── www
│           ├── cgi-bin
│           │   ├── cpuinfo
│           │   ├── list
│           │   ├── reboot
│           │   ├── upload
│           │   ├── upload.c
│           │   ├── upload.cfg
│           │   └── uptime
│           ├── gohome.png
│           ├── index.html
│           ├── kshutdown.png
│           └── upload
│               ├── BadPage.html
│               ├── files
│               │   ├── adult-small.png
│               │   ├── brick.png
│               │   ├── linux-blackfin.jpg
│               │   ├── linux-kernel-dev-book.jpg
│               │   └── lkn-small.jpg
│               └── OkPage.html
└── toolchain
    └── hello.c

16 directories, 37 files
```
