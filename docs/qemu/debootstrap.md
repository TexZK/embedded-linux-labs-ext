# `debootstrap`

It can be interesting to emulate a *Debian* *ARM* *target* system directly on your *host* machine. On the other chapters, we tried with:
barebone [*BusyBox*](tinysystem.md),
[*Buildroot*](buildroot.md),
and custom *Yocto* (**TODO**).<br/>
Within this document, we're focusing on the creation of a *fake rootfs*, to mimic an actual *rootfs* of our emulated *target* machine.

As explained within the
[*Exploring BeagleBone Black*](http://exploringbeaglebone.com/),
there can be two common ways: the old-school via *chroot*, or via a modern *systemd-nspawn*.


## Required tools

* Ubuntu packages:

    `debian-archive-keyring`
    `debootstrap`
    `qemu-user-static`
    `systemd-container`


## `debootstrap` + `chroot`

Let's enter our workspace.

```console
$ LAB_PATH="$HOME/embedded-linux-qemu-labs/debootstrap"
$ mkdir -p $LAB_PATH
```

In order to use `debootstrap`, first install it, along with `qemu-user-static` (our emulation engine). On *Ubuntu*, we also need to install the *keyring* expected by *Debian*'s *debootstrap*.

```console
$ sudo apt install debian-archive-keyring debootstrap qemu-user-static
```

The `debootstrap` installs a basic *Debian*-based *rootf*, with minimal distro and tools. We have to specify the CPU architecture, the *Debian* release (*bullseye* is the current one), and the destination folder.<br/>
Since a *rootfs* contains sensible data (such as the `/dev/` and `/sys/` folders), it must be issued with *root* access.<br/>
We're using the default *Debian keyring* to take pa

```console
$ cd $LAB_PATH
$ sudo debootstrap --arch=armhf --verbose --foreign bullseye rootfs
I: Checking Release signature
    ...
$ ls rootfs/
bin  boot  debootstrap  dev  etc  home  lib  proc  root  run  sbin  sys  tmp  usr  var
```
Let's run `chroot` with *root* privileges on or newly-created *rootfs*.<br/>
Once entered, the shell is confined to the content of this *rootfs*. To exit from this *chroot jail*, just launch `exit` command.

```console
$ sudo chroot rootfs/
I have no name!@vm:/# uname -a
Linux vm 5.19.0-38-generic #39~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Fri Mar 17 21:16:15 UTC 2 armv7l GNU/Linux
I have no name!@vm:/# whoami
whoami: cannot find name for user ID 0: No such file or directory
```

As you can see, our *target* machine is actually an emulated *ARM* architecture, but we aren't finished yet! Indeed, our user isn't even registered yet, strangely named `I have no name!`.<br/>
This is because we need to conclude a *second stage* installation phase for `debootstrap`, with the `--second-stage` argument within the *target* itself.
The *second stage* resumes the package installation, and finalizes it to make the *target* system ready for proper use the next time it's run. We'd better set the *root password* then (let's just set it to `pass`).

```console
I have no name!@vm:/# /debootstrap/debootstrap --second-stage
I: Installing core packages...
    ...
I: Base system installed successfully.
I have no name!@vm:/# exit
exit
$ sudo chroot rootfs/
root@vm:/# whoami
root
root@vm:/# passwd -d root
root@vm:/# passwd
New password: pass
Retype new password: pass
passwd: password updated successfully
I have no name!@vm:/# exit
exit
```

Congratulations! Your fake minimal *Debian* for *ARM* is ready!


## `systemd` container

To work with the `systemd` container, you need to install its package first.<br/>
It's recommended to enable *unpriviledged user namespaces* on the *host*.<br/>
You can restart the `systemd` service then.

```console
$ sudo apt install systemd-container
$ sudo sh -c 'echo "kernel.unprivileged_userns_clone=1" > /etc/sysctl.d/nspawn.conf'
$ sudo systemctl restart systemd-sysctl.service
```

It's now time to setup the *rootfs* we already created for usage with `systemd`.
We're going to re-use the *rootfs* made with `chroot` previously into the `rootfs` folder, calling the machine container `fake`.<br/>
To be able to log in, we need to make sure the `dbus` package is installed on the *target*. After installation, let's `exit` from the container.

```console
$ MACHINE_NAME="fake"
$ sudo systemd-nspawn -D rootfs -U -M $MACHINE_NAME
Spawning container fake on /home/me/embedded-linux-qemu-labs/debootstrap/rootfs.
Press ^] three times within 1s to kill container.
Selected user namespace base 1736310784 and range 65536.
Failed to correct timezone of container, ignoring: Value too large for defined data type
root@fake:~# apt install dbus
    ...
root@fake:~# exit
logout
Container fake exited successfully.
```

Ok, our `chroot` replacement is working! But, we can do even more with this `systemd` service: we can launch containers!<br/>
The `systemd` containers must be running on *machine images*, usually from the `/usr/lib/machines/` folder. We already have a *machine*, we just need to put it into place, or rather, *bind* a mount point to our `rootfs` folder (*symlinks* don't seem to work properly for this service).

```console
$ sudo mkdir -p "/var/lib/machines/$MACHINE_NAME"
$ sudo mount --bind "$LAB_PATH/rootfs" "/var/lib/machines/$MACHINE_NAME"
```

To test that the container works, we can now *spawn* a new contaier against the *machine* we've just created, check its status, log in, shutdown, and stop the container.

```console
$ sudo machinectl start $MACHINE_NAME
$ machinectl list
MACHINE CLASS     SERVICE        OS     VERSION ADDRESSES
fake    container systemd-nspawn debian 11      -

1 machines listed.
$ sudo systemctl status systemd-nspawn@$MACHINE_NAME
● systemd-nspawn@fake.service - Container fake
     Loaded: loaded (/lib/systemd/system/systemd-nspawn@.service; disabled; vendor preset: enabled)
     Active: active (running) since Tue 2023-04-11 18:41:18 CEST; 33s ago
       Docs: man:systemd-nspawn(1)
   Main PID: 16568 (systemd-nspawn)
     Status: "Container running: Startup finished in 2.569s."
      Tasks: 18 (limit: 16384)
     Memory: 69.0M
        CPU: 5.384s
     CGroup: /machine.slice/systemd-nspawn@fake.service
    ...
$ sudo machinectl login $MACHINE_NAME
Connected to machine fake. Press ^] three times within 1s to exit session.

Debian GNU/Linux 11 vm pts/1

vm login: root
Password: pass
Linux vm 5.15.0-67-generic #74-Ubuntu SMP Wed Feb 22 14:14:39 UTC 2023 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@vm:~# shutdown -h now

Machine fake terminated.
$ sudo machinectl stop $MACHINE_NAME
```

Finally, we can unmount out machine mounting point.

```console
$ sudo umount "/var/lib/machines/$MACHINE_NAME"
```


## References

* <https://help.ubuntu.com/community/DebootstrapChroot/>

* <https://wiki.ubuntu.com/ARM/RootfsFromScratch/QemuDebootstrap/>

* <https://wiki.debian.org/nspawn/>

* <https://www.baeldung.com/linux/bind-mounts/>

* <https://blog.entek.org.uk/technology/2020/06/06/building-debian-vms-with-debootstrap.html>

* <http://exploringbeaglebone.com/>
