---
layout: post-detail
title: Make a 32 bit chroot on gentoo
date: 2019-06-11
categories: chroot
description: Make a 32 bit chroot on a no-multilib profile.
img-url: 
comments: true
---

If you use a `no-multilib` profile and want for example use wine to play, you should install a 32 bit environment.

## KERNEL SOURCE
If not alrealy enable, allow 32 bit binaries to run:

    Binary Emulations --->
    [*] IA32 Emulation

Recompile your kernel and restart to activate this.

## Download last stage3 for i686
Get the last stage 3, `hardened`, `systemd` or not at a [gentoo mirror](https://www.gentoo.org/downloads/mirrors/):

    $ w3m https://mirrors.evowise.com/gentoo/releases/x86/autobuilds/

Control the gpg signature and the checksum:

    $ gpg --keyserver pool.sks-keyservers.net --recv-key 0x2D182910
    $ gpg --verify stage3-i686-*.DIGEST.asc

Result:
```
gpg: Signature made Fri 07 Jun 2019 11:04:35 PM AKDT
gpg:                using RSA key 534E4209AB49EEE1C19D96162C44695DB9F6043D
gpg: Good signature from "Gentoo Linux Release Engineering (Automated Weekly Release Key) <releng@gentoo.org>" [unknown]
```
Checksums:

    $ awk '/SHA512 HASH/{getline;print}' stage3-i686-*.DIGEST.asc | sha512sum --check

Result:
    
    stage3-i686-hardened-20190607T214502Z.tar.xz: OK
    stage3-i686-hardened-20190607T214502Z.tar.xz.CONTENTS: OK

## Switch to root

    $ sudo su -

## Create the directory
If you use `zfs`, you can create a dataset like this, replace `zfsforninja` by your pool name:

    # zfs create -o mountpoint=/usr/local/gentoo32 -o devices=on -o setuid=on -o exec=on zfsforninja/usr/gentoo32

Else, create a basic directory:

    # mkdir /usr/local/gentoo32
    # cd /usr/local/gentoo32

Unpack the stage 3:

    # tar xvJpf stage3-*.tar.{bz2,xz} --xattrs-include='*.*' --numeric-owner
    # rm stage3*

## Copy the necessary files

    # cp -L /etc/resolv.conf etc/
    # cp -L /etc/passwd etc/
    # cp -L /etc/portage/make.conf etc/portage/make.conf

Change or add the `CHOST` value to `i686-pc-linux-gnu` !

    # vim etc/portage/make.conf
    CHOST="i686-pc-linux-gnu"

You don't need to customize package use.

## Mount 

    # mount -o bind /dev dev
    # mount -o bind /sys sys
    # mount -o bind /proc proc
    # mount -o bind /run run

Mount portage:

    # mkdir usr/portage
    # mount -o bind /usr/portage usr/portage

## Chroot

    $ sudo linux32 chroot /usr/local/gentoo32 /bin/bash
    # source /etc/profile && env-update

To verify the 32-bit environment:

    # uname -n
    
## Compile
You have to compile all i686 programs you need.

    # emerge -auvDN @world
    # emerge -av wine-staging lutris

## Recreate your home dir
If you use zsh, you have to install:

    # emerge -av zsh

Your home dir is normally void, so:

    # mkdir /home/username
    # chown -R username:username /home/username
    # su username

## Script
Finally, we can write a little script:

```sh
#!/bin/sh

CHRD="/usr/local/gentoo32"

start() {
  mount -o bind /dev $CHRD/dev >/dev/null
  mount -o bind /dev/pts $CHRD/dev/pts >/dev/null
  mount -o bind /dev/shm $CHRD/dev/shm >/dev/null
  mount -o bind /proc $CHRD/proc >/dev/null
  mount -o bind /sys $CHRD/sys >/dev/null
  mount -o bind /tmp $CHRD/tmp >/dev/null
  [[ ! -d $CHRD/usr/portage ]] && mkdir -p $CHRD/usr/portage
  mount -o bind /usr/portage $CHRD/usr/portage/ >/dev/null
  echo "Copying 32bit chroot files"
  cp -pf /etc/resolv.conf $CHRD/etc >/dev/null
  cp -pf /etc/passwd $CHRD/etc >/dev/null
  cp -pf /etc/shadow $CHRD/etc >/dev/null
  cp -pf /etc/group $CHRD/etc >/dev/null
  cp -pf /etc/gshadow $CHRD/etc >/dev/null
  cp -pf /etc/hosts $CHRD/etc > /dev/null
  cp -Ppf /etc/localtime $CHRD/etc >/dev/null
  echo "you can start with : "
  echo "sudo linux32 chroot $CHRD /bin/bash"
  echo "source /etc/profile && env-update"
}

stop() {
  umount -f $CHRD/dev/pts >/dev/null
  umount -f $CHRD/dev/shm >/dev/null
  umount -f $CHRD/dev >/dev/null
  umount -f $CHRD/proc >/dev/null
  umount -f $CHRD/sys >/dev/null
  umount -f $CHRD/tmp >/dev/null
  umount -f $CHRD/usr/portage/ >/dev/null
}

if [[ $1 == "start" ]] ; then
  echo "call with start : $1";
  start;
  xhost local:localhost;
  exit 0
elif [[ $1 == "stop" ]] ; then
  echo "call with stop : $1";
  stop;
  exit 0
else
  echo "need arg start or stop";
  exit 1
fi
```
Usage: `chroot32.sh start`, `chroot32.sh stop`.
