# Accessing Hardware Devices

*Important: The manipulations in this lab are limited because we’re working with an emulated platform, not with real hardware. It would be best to get your hands on the hardware platforms we support. Our instructions for such platforms really cover practising with internal and external hardware devices.*


## Objectives

* Learn how to access hardware devices.


## Setup

Go to the `$HOME/embedded-linux-qemu-labs/hardware` directory, which provides useful files for
this lab.<br/>
However, we will go on booting the system through NFS, using the root filesystem built by the
previous lab.


## Exploring `/dev`

Start by exploring `/dev/` on your target system (with *QEMU*).

```console title="QEMU - BusyBox"
# ls /dev/
console          ptypc            tty35            ttyAMA2
cpu_dma_latency  ptypd            tty36            ttyAMA3
dri              ptype            tty37            ttyp0
fb0              ptypf            tty38            ttyp1
full             random           tty39            ttyp2
gpiochip0        rtc0             tty4             ttyp3
gpiochip1        snd              tty40            ttyp4
gpiochip2        tty              tty41            ttyp5
gpiochip3        tty0             tty42            ttyp6
hwrng            tty1             tty43            ttyp7
input            tty10            tty44            ttyp8
kmsg             tty11            tty45            ttyp9
mem              tty12            tty46            ttypa
mmcblk0          tty13            tty47            ttypb
mmcblk0p1        tty14            tty48            ttypc
mmcblk0p2        tty15            tty49            ttypd
mmcblk0p3        tty16            tty5             ttype
mtd0             tty17            tty50            ttypf
mtd0ro           tty18            tty51            ubi_ctrl
mtd1             tty19            tty52            urandom
mtd1ro           tty2             tty53            usbmon0
mtdblock0        tty20            tty54            vcs
mtdblock1        tty21            tty55            vcs1
null             tty22            tty56            vcs2
ptmx             tty23            tty57            vcs3
ptyp0            tty24            tty58            vcs4
ptyp1            tty25            tty59            vcsa
ptyp2            tty26            tty6             vcsa1
ptyp3            tty27            tty60            vcsa2
ptyp4            tty28            tty61            vcsa3
ptyp5            tty29            tty62            vcsa4
ptyp6            tty3             tty63            vcsu
ptyp7            tty30            tty7             vcsu1
ptyp8            tty31            tty8             vcsu2
ptyp9            tty32            tty9             vcsu3
ptypa            tty33            ttyAMA0          vcsu4
ptypb            tty34            ttyAMA1          zero
```

Here are a few noteworthy device files that you will see:

* *Terminal devices*: devices starting with `tty`.<br/>
  Terminals are user interfaces taking text as input and producing text as output, and are typically used by interactive shells.<br/>
  In particular, you will find console which matches the device specified through `console=` in the kernel command line (*U-Boot*'s `bootargs`).<br/>
  You will also find the `ttyAMA0` device file we used for the emulated debug serial port.

* *Pseudo-terminal devices*: devices starting with `pty`, used when you connect through SSH for example.<br/>
  Those are virtual devices, but there are so many in `/dev/` that we wanted to give a description here.

* *MMC devices and partitions*: devices starting with `mmcblk`.<br/>
  You should here recognize the MMC devices on your system, and the associated partitions.

Don’t hesitate to explore `/dev/` on your workstation too!

```console
$ ls /dev/
autofs           loop1         rtc0      tty2   tty47      ttyS15   vboxguest
block            loop10        sda       tty20  tty48      ttyS16   vboxuser
bsg              loop11        sda1      tty21  tty49      ttyS17   vcs
btrfs-control    loop12        sdb       tty22  tty5       ttyS18   vcs1
bus              loop13        sdb1      tty23  tty50      ttyS19   vcs2
cdrom            loop14        sdc       tty24  tty51      ttyS2    vcs3
char             loop2         sg0       tty25  tty52      ttyS20   vcs4
console          loop3         sg1       tty26  tty53      ttyS21   vcs5
core             loop4         sg2       tty27  tty54      ttyS22   vcs6
cpu              loop5         sg3       tty28  tty55      ttyS23   vcsa
cpu_dma_latency  loop6         shm       tty29  tty56      ttyS24   vcsa1
cuse             loop7         snapshot  tty3   tty57      ttyS25   vcsa2
disk             loop8         snd       tty30  tty58      ttyS26   vcsa3
dma_heap         loop9         sr0       tty31  tty59      ttyS27   vcsa4
dri              loop-control  stderr    tty32  tty6       ttyS28   vcsa5
ecryptfs         mapper        stdin     tty33  tty60      ttyS29   vcsa6
fb0              mcelog        stdout    tty34  tty61      ttyS3    vcsu
fd               mem           tty       tty35  tty62      ttyS30   vcsu1
full             mqueue        tty0      tty36  tty63      ttyS31   vcsu2
fuse             net           tty1      tty37  tty7       ttyS4    vcsu3
hidraw0          null          tty10     tty38  tty8       ttyS5    vcsu4
hpet             nvram         tty11     tty39  tty9       ttyS6    vcsu5
hugepages        port          tty12     tty4   ttyprintk  ttyS7    vcsu6
hwrng            ppp           tty13     tty40  ttyS0      ttyS8    vfio
i2c-0            psaux         tty14     tty41  ttyS1      ttyS9    vga_arbiter
initctl          ptmx          tty15     tty42  ttyS10     udmabuf  vhci
input            pts           tty16     tty43  ttyS11     uhid     vhost-net
kmsg             random        tty17     tty44  ttyS12     uinput   vhost-vsock
log              rfkill        tty18     tty45  ttyS13     urandom  zero
loop0            rtc           tty19     tty46  ttyS14     userio   zfs
```


## Exploring `/sys`

The next thing you can explore is the *sysfs* filesystem.

A good place to start is `/sys/class/`, which exposes devices classified by the kernel frameworks which manage them.

```console title="QEMU - BusyBox"
# ls /sys/class/
ata_device    drm           mem           ptp           tty
ata_link      graphics      misc          regulator     ubi
ata_port      hwmon         mmc_host      rtc           usbmon
backlight     i2c-adapter   mtd           scsi_device   vc
bdi           input         net           scsi_disk     virtio-ports
block         leds          power_supply  scsi_host     vtconsole
devlink       mdio_bus      pps           sound         wakeup
```

For example, go to `/sys/class/net/`, and you will see all the networking interfaces on your system, whether they are internal, external or virtual ones.

```console title="QEMU - BusyBox"
# ls /sys/class/net/
eth0  lo
```

Find which subdirectory corresponds to the network connection to your host system, and then check device properties such as:

* `speed`: will show you whether this is a gigabit or hundred megabit interface.

* `address`: will show the device MAC address. No need to get it from a complex command!

* `statistics/rx_bytes` will show you how many bytes were received on this interface.

Don’t hesitate to look for further interesting properties by yourself!

```console title="QEMU - BusyBox"
# ls /sys/class/net/eth0/
addr_assign_type      flags                 phys_port_name
addr_len              gro_flush_timeout     phys_switch_id
address               ifalias               power
broadcast             ifindex               proto_down
carrier               iflink                queues
carrier_changes       link_mode             speed
carrier_down_count    mtu                   statistics
carrier_up_count      name_assign_type      subsystem
dev_id                napi_defer_hard_irqs  testing
dev_port              netdev_group          threaded
device                operstate             tx_queue_len
dormant               phydev                type
duplex                phys_port_id          uevent
# cat /sys/class/net/eth0/speed
100
# cat /sys/class/net/eth0/address
52:54:00:12:34:56
# cat /sys/class/net/eth0/statistics/rx_bytes
963618
```

You can also check whether `/sys/class/thermal/` exists and is not empty on your system.
That’s the thermal framework, and it allows to access temperature measures from the thermal sensors on your system.

```console title="QEMU - BusyBox"
# ls /sys/class/thermal/
ls: /sys/class/thermal: No such file or directory
```

Next, you can now explore all the buses (virtual or physical) available on your system, by checking the contents of `/sys/bus/`.
In particular, go to `/sys/bus/mmc/devices/` to see all the MMC devices on your system.<br/>
Go inside the directory for the first device and check several files (for example):

* `serial`: the serial number for your device.

* `preferred_erase_size`: the preferred erase block for your device. It’s recommended that partitions start at multiples of this size.

* `name`: the product name for your device. You could display it in a user interface or log file, for example.

```console title="QEMU - BusyBox"
# ls /sys/bus/
ac97          cpu           i2c           platform      usb
amba          dp-aux        mdio_bus      scsi          virtio
clockevents   event_source  mmc           sdio          workqueue
clocksource   gpio          mmc_rpmb      serio
container     hid           nvmem         soc
# ls /sys/bus/mmc/devices/
mmc0:4567
# ls /sys/bus/mmc/devices/mmc0:4567/
block                 hwrev                 scr
cid                   manfid                serial
csd                   name                  ssr
date                  ocr                   subsystem
driver                oemid                 type
dsr                   power                 uevent
erase_size            preferred_erase_size
fwrev                 rca
# cat /sys/bus/mmc/devices/mmc0:4567/serial
0xdeadbeef
# cat /sys/bus/mmc/devices/mmc0:4567/preferred_erase_size
4194304
# cat /sys/bus/mmc/devices/mmc0:4567/name
QEMU!
```

Don’t hesitate to spend more time exploring `/sys/` on your system!


## Licensing

This document is an extension to: [*Embedded Linux System Development - Practical Labs - QEMU Variant*](https://bootlin.com/doc/training/embedded-linux-qemu/)
 &mdash; &copy; 2004-2023, *Bootlin* [https://bootlin.com/](https://bootlin.com), [`CC-BY-SA-3.0`]((https://creativecommons.org/licenses/by-sa/3.0/)) license.
