# Processors

It's useful to take advantage of a multi-core machine. The OS does it automatically, but most toolchains don't.

An agnostic way to know the number of cores is via the `nproc` command from the `coreutils` Ubuntu package:

```console
$ sudo apt install coreutils
$ nproc
```

To let the GNU `make` command leverage the speedup opportunity, provide it via the `-j <JOBS>` command-line option (the space is optional), like in the following generic example:

```console
$ make -j$(nproc)
```

Otherwise, declare the special environment variable named `MAKEFLAGS`, containing that option, to avoid typing it at every call to `make`:

```console
$ export MAKEFLAGS=-j$(nproc)
$ make
```
