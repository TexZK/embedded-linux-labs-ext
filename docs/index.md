# Overview

This document extends the *Practical Labs* provided by [**Bootlin**](https://bootlin.com/).
Their courses for embedded system development are really good, and what I appreciate the most is their open source approach.<br/>
On one hand, their paid trainnig sessions are valuable for professionals, and to have live support from their team, which is expected for the many paid courses available all around the world.<br/>
On the other hand, having the very same training material released as open source really gives more opportunity to spread knowledge on such complex topics even for students, hobbyists, and us amateurs in general!
We all love free open source stuff!

I decided to write this to complement some missing information, updated or alternative procedures while performing the labs.
I'm doing this also because I find it the best way to really learn stuff: read, apply, modify, describe!

I **strongly** recommend to read *Bootlin*'s **slides** before practicing with the labs!

I'm going to follow the
[*Embedded Linux System Development - QEMU variant*](https://bootlin.com/doc/training/embedded-linux-qemu/),
so that anybody can try stuff without buying any physical boards.
The course provided by *Bootlin*, as per March 2023, isn't the most updated one of theirs, which is for the *STM32MP1* boards instead.

Since old and new versions of the course cover variants of the same topics, and I find all those variants valuable knowledge, I'm willing to provide such variants as well.

I own a [*BeagleBone Black rev.C* board](https://beagleboard.org/black),
so I'm willing to extend this document for it &mdash; I'm not going to buy yet another board right now!


## Style and conventions

This is an informal document, so that it can feel closer to the user.
I try my best to be clean and precise about what is describe.
The topics covered here are very complex, and the actual purpose of this document is for me to consolidate my knowledge while writing these notes.
I'm not an expert &mdash; rather, I'm still studying all of this &mdash; so there can be mistakes or unclear parts.
Furthermore, English isn't my native language.

New terms are indicated in *italics* the first time they're introduced, or to distinguish them from plain text.

Important text is marked with **bold**.

The reference *Linux shell* is *bash*. We usually type `~` instead of `$HOME`.
The following is a shell code block:

```console
$ echo "Shell command line from a generic user"
# echo "Shell command line from the root user"
```

Sometimes I'm going to use shell variables to make command lines shorter and with added semantics, easily editable.

```console
$ VARIABLE_USED_BY_FOLLOWING_CODE_BLOCKS="..."
$ variable_used_only_within_this_code_block="..."
```

When using variables I typically enclose string expressions within quotes to improve readability, despite quotes usually being optional.

```console
$ MY_PATH="/this/is/my/path"
$ MY_PATH=/quotes/not/required
$ echo $MY_PATH/direct/use/
$ echo "${MY_PATH}/braces/and/quotes/to/improve/readability"
```

Very long command lines can be wrapped. I leave two spaces before the wrapping `\`, or between command line options, to give a better visual cue.

```console
$ some-command  --option first  --option second  --option third
$ echo Some commands can be split  \
       into multiple lines,  \
       to improve readability
```

When creating small text files, I usually prefer to pass the content via command line, as in:

```console
$ cat  > file.txt  <<'EOF'
Here's some text that's going to
become the content of the
"file.txt" text file,
as soon as the user puts
an 'EOF' on a standalone line.
EOF
$ cat file.txt
Here's some text that's going to
become the content of the
"file.txt" text file,
as soon as the user puts
an 'EOF' on a standalone line.
```

An untitled shell block usually represents a shell running on the Linux *host* machine.<br/>
In case of a shell running on a different machine, like a *QEMU guest* or a *remote* shell, or even file content, that shell is given a title:

```console title="picocomBBB - BusyBox"
$ echo "I'm a BusyBox shell running on the remote BeagleBone Black board!"
$ echo "You can use me because you're connected to me via serial connection with `picocom`!"
```

```console title="ssh - Debian"
$ echo "I'm a BASH shell running on the remote Debian board!"
$ echo "You can use me because you're connected to me via SSH!"
```

``` title="QEMU - U-Boot"
=> echo "I'm an U-Boot prompt running on the emulated target board!"
=> echo "You can use me because you're running QEMU on your host machine!"
```

```sh title="script.sh"
#!/bin/sh
echo "Hello $USER! I'm a shell script file!"
```

Inline code: `int main(int argc, char *argv[]);`.


## Backups

I created a *Google Drive* folder where I put backup copies of relevant outputs, tools, and files mentioned within the documents, in case they cannot be retrieved the standard way (*e.g.* broken *git* reposifory, or unreachable sites).

<https://drive.google.com/drive/folders/1DhCmpQT-Sus44rgsjVs82BA0bJSz8-eV?usp=share_link>
