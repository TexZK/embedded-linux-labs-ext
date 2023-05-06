# App - Development

**TODO: THIS DOCUMENT IS STILL JUST A DRAFT!**


## Objectives

* Compile an application against a *Buildroot* build space and debug it remotely.


## Required tools

* Ubuntu packages:

    `snap`
    and those from the previous labs.

* *snap* packages:

    `code` (*Visual Studio Code*)

* The *root* filesystem and *NFS* setup from the [*integration course*](integration.md).


## Setup

We will continue to use the same root filesystem.

Our goal is to compile and debug our own *MPD* client. This client will be driven by the Nunckuk to switch between audio tracks, and to adjust the playback volume.

However, this client will be used together with mpc, as it won't be able to create the playlist and start the playback. It will just be used to control the volume and switch between songs.
So, you need to run `mpc` commands first before trying the new client:

```console title="picocomBBB - systemd"
# mpc update
# mpc add /
# mpc pause
```

We will use the new client to resume playback.


## Compile your own application

Go to the `$HOME/embedded-linux-bbb-labs/appdev` directory.

```console
$ LAB_PATH="$HOME/embedded-linux-bbb-labs/appdev"
```

In the lab directory the file `nunchuk-mpd-client.c` contains an application which implements a simple *MPD* client based on the libmpdclient library.
As *MPC* is also based on this library, *Buildroot* already compiled it and added it to our *root* filesystem.
What's special in this application is that it allows to drive music playback through our *Nunchuk*.

*Buildroot* has generated toolchain wrappers in `output/host/bin/`, which make it easier to use the toolchain, since these wrappers pass some mandatory flags, especially the `--sysroot gcc`
flag, which tells *GCC* where to look for headers and libraries.
This way, we can compile our application outside *Buildroot*, as often as we want.

Let's add this directory to our `PATH`, and try to compile the application.

```console hl_lines="5"
$ export PATH="$LAB_PATH/../integration/buildroot/output/host/bin:$PATH"
$ cd $LAB_PATH
$ arm-linux-gcc -o nunchuk-mpd-client nunchuk-mpd-client.c
    ...
nunchuk-mpd-client.c:(.text+0x28): undefined reference to `mpd_connection_get_error_message'
    ...
collect2: error: ld returned 1 exit status
```

The compiler complains about undefined references to some symbols in `libmpdclient`.
This is normal, since we didn't tell the compiler to link with this library.<br/>
So, let's use `pkg-config` to query its database about the list of libraries needed to build an application against `libmpdclient`. Again, `output/host/bin/` has a special `pkg-config` that automatically knows where to look, so it already knows the right paths to find `.pc` files and their *sysroot*.

```console hl_lines="5"
$ arm-linux-gcc -o nunchuk-mpd-client nunchuk-mpd-client.c  \
    $(pkg-config --libs libmpdclient)
```

Copy the `nunchuk-mpd-client` executable to the `/root` directory of the *root* filesystem, and
then *strip* it.

```console hl_lines="5"
$ NFSROOT="$LAB_PATH/../integration/nfsroot"
$ cp nunchuk-mpd-client "$NFSROOT/root/"
$ arm-linux-strip "$NFSROOT/root/nunchuk-mpd-client"
```

Back to target system, try to run the program:

```console title="picocomBBB - systemd" hl_lines="2"
# /root/nunchuk-mpd-client
ERROR: didn't manage to find the Nunchuk device in /dev/input. Is the Nunchuk driver loaded?
```


## Enable debugging tools

In order to debug our application, let's make *Buildroot* build some debugging tools for our *root* filesystem.
This is also an opportunity to enable `perf`, that we are using later on during this lab.

Go back to the *Buildroot* configuration interface and enable the following options:

* In `Kernel` &rarr; `Linux Kernel Tools`:

    * Enable `perf`.

* In `Target packages` &rarr; `Debugging, profiling and benchmark`:

    * Enable `ltrace`.
    * Enable `strace`.

Then rebuild and update your *root* filesystem.

```console
$ BUILDROOT="$LAB_PATH/../integration/buildroot"
$ cd $BUILDROOT
$ make menuconfig
$ cp .config "$LAB_PATH/buildroot-appdev.config"
$ make
$ cp output/images/zImage /srv/tftp/
$ cd $NFSROOT
$ sudo rm -rf *
$ tar xfv "$BUILDROOT/output/images/rootfs.tar"
$ cp "$LAB_PATH/nunchuk-mpd-client" "$NFSROOT/root/"
```


## Using strace

Let's run the program through the `strace` command to find out why this happens.

```console title="picocomBBB - systemd"
# strace /root/nunchuk-mpd-client
    ...
openat(AT_FDCWD, "/dev/event0", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/dev/event1", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/dev/event2", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/dev/event3", O_RDONLY) = -1 ENOENT (No such file or directory)
write(2, "ERROR: didn't manage to find the"..., 93ERROR: didn't manage to find the Nunchuk device in /dev/input. Is the Nunchuk driver loaded?
) = 93
exit_group(1)                           = ?
+++ exited with 1 +++
```

You should see that it's trying to access files that don't exist: it's looking for `/dev/event0`, while the actual file is at `/dev/input/event0`!<br/>
So, let's fix the source code from `"/dev/%s"` to `"/dev/input/%s"`:

```c title="File: nunchuk-mpd-client.c - fixed input device path" hl_lines="12"
/*...*/
int main(int argc, char ** argv)
{
    /*...*/
        /* Find Nunchuk input device */

        ndev = scandir("/dev/input", &namelist, is_event_device, alphasort);
    /*...*/
        for (i = 0; i < ndev; i++)
        {
            /*...*/
                snprintf(fname, sizeof(fname), "/dev/input/%s", namelist[i]->d_name);
            /*...*/
        }
    /*...*/
}
/*...*/
```

```console
$ cd $LAB_PATH
$ sed -i 's|"/dev/%s"|"/dev/input/%s"|' nunchuk-mpd-client.c
$ arm-linux-gcc  -o nunchuk-mpd-client  nunchuk-mpd-client.c  $(pkg-config --libs libmpdclient)
$ cp nunchuk-mpd-client "$NFSROOT/root/"
```

Rebuild the program and run it again (no need to *strip* for our debug sessions):

```console title="picocomBBB - systemd"
# /root/nunchuk-mpd-client
ERROR: didn't manage to find the Nunchuk device in /dev/input. Is the Nunchuk driver loaded?
```

Ouch, same problem again!

You can run the program again through `strace`, and check that the right paths are now accessed, but the cause of the issue might be tricky to find.

```console title="picocomBBB - systemd" hl_lines="7"
# strace /root/nunchuk-mpd-client
    ...
openat(AT_FDCWD, "/dev/input/event0", O_RDONLY) = 3
ioctl(3, EVIOCGNAME(256), "tps65217_pwrbutton\0") = 19
close(3)                                = 0
openat(AT_FDCWD, "/dev/input/event1", O_RDONLY) = 3
ioctl(3, EVIOCGNAME(256), "Wii Nunchuk\0") = 12
close(3)                                = 0
openat(AT_FDCWD, "/dev/input/event2", O_RDONLY) = 3
ioctl(3, EVIOCGNAME(256), "Logitech Inc. Logitech USB Heads"...) = 57
close(3)                                = 0
openat(AT_FDCWD, "/dev/input/event3", O_RDONLY) = 3
ioctl(3, EVIOCGNAME(256), "Logitech Inc. Logitech USB Heads"...) = 40
close(3)                                = 0
write(2, "ERROR: didn't manage to find the"..., 93ERROR: didn't manage to find the Nunchuk device in /dev/input. Is the Nunchuk driver loaded?
) = 93
exit_group(1)                           = ?
+++ exited with 1 +++
```

If you didn't spot the bug, `ltrace` might help you!


## Using ltrace

Let's run the program through `ltrace` now. We will be able to see the shared library calls.
Take your time to study the ltrace output. That's interesting information!
Back to our issue, the last lines of output should make the issue pretty obvious.

```console title="picocomBBB - systemd" hl_lines="7"
# ltrace /root/nunchuk-mpd-client
    ...
snprintf("/dev/input/event1", 256, "/dev/input/%s", "event1") = 17
free(0x49b228)                                   = <void>
open("/dev/input/event1", 0, 022230010)          = 3
ioctl(3, -2130688762, 0xbef8fb9c)                = 12
strcmp("Wii Nunchuck", "Wii Nunchuk")            = -8
close(3)
    ...
```

The *for* loop within the program should stop at the first instance of a device named `"Wii Nunchuck"`... and indeed, there are none, because it's called `"Wii Nunchuk"` instead!
This was clearly a typo, let's fix it!

```console
$ cd $LAB_PATH
$ sed -i "s/Nunchuck/Nunchuk/" nunchuk-mpd-client.c
$ arm-linux-gcc  -o nunchuk-mpd-client  nunchuk-mpd-client.c  $(pkg-config --libs libmpdclient)
$ cp nunchuk-mpd-client "$NFSROOT/root/"
```

You should now be able to use the new client, driving the server through the following Nunchuk inputs:

* Joystick up: volume up 5%
* Joystick down: volume down 5%
* Joystick left: previous song
* Joystick right: next song
* Z (big) button: pause / play
* C (small) button: quit client!

```console title="picocomBBB - systemd" hl_lines="5 7"
# /root/nunchuk-mpd-client
# /root/nunchuk-mpd-client
Connection successful
Play/Pause
Quit
[ 2804.673441] audit: type=1701 audit(1683378416.277:7): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=235 comm="nunchuk-mpd-cli" exe="/root/nunchuk-mpd-client" sig=11 res=1
Segmentation fault
```

Have fun with the new client. You'll just realize that quitting causes the program to crash with a segmentation fault. Let's debug this too, this time via `gdbserver`.


## Using gdbserver

We are going to use `gdbserver` to understand why the program segfaults.

Compile `nunchuk-mpd-client.c` again with the `-g` (`g` means `gdb`) option to include debugging symbols.<br/>
This time, just keep it on your workstation, as you already have the version without debugging symbols on your target.

```console
$ cd $LAB_PATH
$ arm-linux-gcc  -g  -o nunchuk-mpd-client  nunchuk-mpd-client.c  $(pkg-config --libs libmpdclient)
```

Then, on the target side, run the program under `gdbserver`. `gdbserver` will listen on a TCP port for a connection from `gdb` on the host, and will control the execution of `nunchuk-mpd-client` according to the `gdb` commands.

```console title="picocomBBB - systemd"
# gdbserver localhost:2345 /root/nunchuk-mpd-client
Process /root/nunchuk-mpd-client created; pid = 179
Listening on port 2345
```

On the host side, run `arm-linux-gdb` (also found in your toolchain):

```console
$ arm-linux-gdb nunchuk-mpd-client
GNU gdb (GDB) 11.1
    ...
Reading symbols from nunchuk-mpd-client...
(gdb)
```

`gdb` starts and loads the debugging information from the `nunchuk-mpd-client` binary (in the `appdev` directory), which was compiled with `-g`.

Then, we need to tell where to find our libraries, since they are not present in the default `/lib/` and `/usr/lib/` directories on your workstation.<br/>
This is done by setting the `sysroot` variable (on one line; replace `<user>` with your user name).<br/>
Then, tell `gdb` to connect to the remote system.

```
(gdb) set sysroot /home/<user>/embedded-linux-bbb-labs/integration/buildroot/output/staging
(gdb) target remote 192.168.0.69:2345
Remote debugging using 192.168.0.69:2345
Reading symbols from /home/me/embedded-linux-bbb-labs/integration/buildroot/output/staging/lib/ld-linux-armhf.so.3...
0xb6fc88c0 in _start ()
   from /home/me/embedded-linux-bbb-labs/integration/buildroot/output/staging/lib/ld-linux-armhf.so.3
```

Then, use `gdb` as usual to set breakpoints, look at the source code, run the application step by
step, etc.<br/>
In our case, we'll just start the program and press the *C* (small) button to quit and trigger the *segmentation fault*:

``` hl_lines="4"
(gdb) continue
Continiung.

Program received signal SIGSEGV, Segmentation fault.
0xb6fb3690 in mpd_settings_free () from /home/me/embedded-linux-bbb-labs/integration/buildroot/output/staging/lib/libmpdclient.so.2
```

After the segmentation fault, you can ask for a backtrace to see where this happened:

``` hl_lines="6"
(gdb) backtrace
#0  0xb6fb3690 in mpd_settings_free ()
   from /home/me/embedded-linux-bbb-labs/integration/buildroot/output/staging/lib/libmpdclient.so.2
#1  0xb6fa8548 in mpd_connection_free ()
   from /home/me/embedded-linux-bbb-labs/integration/buildroot/output/staging/lib/libmpdclient.so.2
#2  0x00401300 in main (argc=1, argv=0xbefffe24) at nunchuk-mpd-client.c:167
```

This will tell you that the *segmentation fault* occurred in a function of the `libmpdclient`, called by our program.<br/>
You will also get the number of the line in the program which caused this. This should help you to find the bug in our application.

```c title="File: nunchuk-mpd-client.c" hl_lines="11"
/*...*/
                        case BTN_C:
                                if (event.value == 1) {
                                        printf("Quit\n");
                                        quit = 1;
                                        free(conn);
                                }
                                break;
/*...*/
        /* Close connection */
        mpd_connection_free(conn);  /* __LINE__ == 167 */
/*...*/
```

Obviously, the `mpd_connection_free()` function was provided a pointer to some memory that was already unallocated with `free()`.<br/>
We're going to fix this bug later on, after learning how to use more tools.

You can now quit `gdb`.

```
(gdb) quit
A debugging session is active.

        Inferior 1 [process 179] will be killed.

Quit anyway? (y or n) y
```


## Post mortem analysis

Configure your shell on the *target* to get a *core file* dumped when you run `nunchuk-mpd-client` again.

```console title="picocomBBB - systemd" hl_lines="7 9"
# ulimit -c unlimited
# echo "nunchuk-mpd-client.core" > /proc/sys/kernel/core_pattern
# /root/nunchuk-mpd-client
Connection successful
Quit
[ 1617.932592] audit: type=1701 audit(1683388582.214:2): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=194 comm="nunchuk-mpd-cli" exe="/root/nunchuk-mpd-client" sig=11 res=1
Segmentation fault (core dumped)
# du -h nunchuk-mpd-client.core*
124.0K  nunchuk-mpd-client.core.194
# chmod a+r nunchuk-mpd-client.core.194
```

Once you have such a file, inspect it with `arm-linux-gdb` on the *host*, set the `sysroot` setting, and then generate a *backtrace* to see where the program crashed.<br/>
This way, you can have information about the crash without running the program through the debugger.

Because the executable on the *target* does not contain debug information (it was compiled without the `-g` option), and because the core dump holds the *absolute paths* of the *target*, we have to emulate the state of the *target sysroot* somehow.<br/>
One way works by copying the *debug* executable and the *core dump* to the corresponding places within the `staging` folder by *Buildroot*.<br/>
With `set sysroot`, `gdb` sees the debug files as if they were there on the fake *target*.

``` title="Folder structure excerpts to analyze our core dump"
[TARGET]
/                                       <<<  host NFS: $NFSROOT
├── lib/
|   └── libmpdclient.so.2
└── root/
    ├── nunchuk-mpd-client              <<<  without debug symbols
    └── nunchuk-mpd-client.core.194

[HOST]
buildroot/output/staging/               <<<  gdb: sysroot
├── lib/
|   └── libmpdclient.so.2
└── root/
    ├── nunchuk-mpd-client              <<<  with debug symbols, copied from: $LAB_PATH/data/
    └── nunchuk-mpd-client.core.194     <<<  copied from: $NFSROOT/root/
```

Here's the sequence of operations, switching between the shell and `gdb`:

```console
$ cd "$BUILDROOT/output/staging/root/"
$ cp "$LAB_PATH/nunchuk-mpd-client" ./
$ cp "$NFSROOT/root/nunchuk-mpd-client.core.194" ./
$ arm-linux-gdb  -ex "set sysroot $BUILDROOT/output/staging"
    ...
```

``` hl_lines="1 3 10"
(gdb) file nunchuk-mpd-client
Reading symbols from nunchuk-mpd-client...
(gdb) core-file nunchuk-mpd-client.core.194
warning: core file may not match specified executable file.
[New LWP 194]
Core was generated by `/root/nunchuk-mpd-client'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0xb6f6c690 in mpd_settings_free ()
   from /home/me/embedded-linux-bbb-labs/appdev/../integration/buildroot/output/staging/lib/libmpdclient.so.2
(gdb) backtrace full
#0  0xb6f6c690 in mpd_settings_free ()
   from /home/me/embedded-linux-bbb-labs/appdev/../integration/buildroot/output/staging/lib/libmpdclient.so.2
No symbol table info available.
#1  0xb6f61548 in mpd_connection_free ()
   from /home/me/embedded-linux-bbb-labs/appdev/../integration/buildroot/output/staging/lib/libmpdclient.so.2
No symbol table info available.
#2  0x00421300 in main (argc=1, argv=0xbea46e24) at nunchuk-mpd-client.c:167
        conn = 0x4331c0
        i = 1
        ndev = 4
        ret = 16
        fd = 3
        quit = 1
        num_events = 1
        event = {time = {tv_sec = 1683388582, tv_usec = 224672}, type = 1, code = 306, value = 1}
        namelist = 0x43b1b
(gdb) set confirm off
(gdb) quit
```

```console
$ cd "$BUILDROOT/output/staging/root/"
$ rm nunchuk-mpd-client*
```

You can even see the value of all variables in the different function contexts of your program.
This way, you can have a lot of information about the crash without running the program
through the debugger.


## Visual Studio Code

### Installing software

We are going to use *Visual Studio Code* to do the remote debugging again, and eventually fix and recompile our program.

The first thing to do is install *VSC*. This package is only available as a *snap package*:

```console
$ sudo apt install snap
$ sudo snap install --classic code
```


### Accessing your board through SSH

We will use *Visual Studio Code* to modify and recompile our client program, and also to update and run the binary on the target.
Of course, we will use a simple solution, as we won't be able to spend too much time learning about all the possibilities offered by *VSC*.

For our purpose, a good solution is *SSH*, which allows to copy files (via the `scp` command) and to run remote commands.
We already included the *Dropbear* SSH server in our *root* filesystem.

We just need to implement password-less SSH access, to keep things simple:

Generate a password-less one with the `ssh-keygen` command, named `id_rsa_empty`.<br/>
We're creating two files in `~/.ssh/`: `id_rsa_empty` (*private* key) and `id_rsa_empty.pub` (*public* key).

```console hl_lines="3 5 6"
$ ssh-keygen -t rsa -f ~/.ssh/id_rsa_empty
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/me/.ssh/id_rsa_empty
Your public key has been saved in /home/me/.ssh/id_rsa_empty.pub
The key fingerprint is:
SHA256:MMNVi/JBFlFTv6DpuCpx6zj/WGceGDr7Q6k0ThcBUqo me@vm
    ...
```

Then, create the `/root/.ssh/` directory on the *target*, and in it create an `authorized_keys` file with the line in `id_rsa_empty.pub`.

```console
$ cd "$NFSROOT/root/"
$ mkdir -p .ssh/
$ cp ~/.ssh/id_rsa_empty.pub .ssh/
```

Finally, fix permissions on the *target*, as *Dropbear* is quite strict about them:

```console title="picocomBBB - systemd"
# chmod -R go-rwx /root
# chown -R root.root /root
```

Then, you can test that SSH works without a password:

```console
$ ssh root@192.168.0.69
```

**TODO**

### Compiling and debugging

**TODO**


## Profiling the application with perf

**TODO**


## What to remember

**TODO**


## Packaging with Meson

**TODO**


## Backup and restore

```console
$ cd "$BUILDROOT/output/images/"
$ tar cfJv "$LAB_PATH/appdev-images.tar.xz" *
```


## Licensing

This document is an extension to: [*Embedded Linux System Development - Practical Labs - BeagleBone Black Variant*](https://bootlin.com/doc/training/embedded-linux-bbb/)
 &mdash; &copy; 2004-2023, *Bootlin* [https://bootlin.com/](https://bootlin.com), [`CC-BY-SA-3.0`]((https://creativecommons.org/licenses/by-sa/3.0/)) license.
