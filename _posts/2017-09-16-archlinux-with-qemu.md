---
layout: post-detail
title: Archlinux with qemu in less of 10min.
date: 2017-09-16
categories: archlinux qemu
description: We're create a virtual image of archlinux with qemu in less of 10min.
img-url: https://i.imgur.com/VTg4vo0m.png
comments: true
---

### 1 - Download the last iso.

We need an iso, download the last on official archlinux [site]( https://www.archlinux.org/download/).

### 2 - Create a virtual Hard disk drive.

We create a HDD of 8G here:

    $ qemu-img create -f raw ArchLinux.raw 8G

### 3 - Create a script for qemu.

With our favorite text editor, create a script bellow:

    $ vim archvm
    #!/bin/sh
    exec qemu-system-x86_64 \
    -enable-kvm \
    -cpu host \
    -display sdl \
    -drive file=ArchLinux.raw,format=raw,if=virtio \
    -netdev bridge,id=vmnic,br=br0 -device virtio-net-pci,netdev=vmnic \
    -m 512M \
    -monitor stdio \
    -name "Arch VM" \
    $@

And set it executable.

    $ chmod +x archvm

### 3 - Boot on the iso file.

Now than the script is write, we must simply add two arguments for change boot order:

    $ archvm -boot d -cdrom archlinux.iso

### 4 - Archlinux install

Sry for beginners but i will just write all commands without detailled explanation.  
If you don't understand somes command line, please look to the  official [wiki](https://wiki.archlinux.org/index.php/Installation_Guide).


Change keyboard layout for non-english

    # loadkeys fr
    # timedatectl set-ntp true

Partition ArchLinux.raw, the name on qemu is vda bellow sda.

    # gdisk /dev/vda
        n # partition 1 [enter], from beginning [enter], [+2M], code [EF02]
        n # partition 2 [enter], from beginning [enter], [+256M], code [enter]
        n # partition 3 [enter], from beginning [enter], [enter], code [enter]
        w

Format with ext4, not need the better filesystem.

    # mkfs.ext4 /dev/vda2
    # mkfs.ext4 /dev/vda3

Set mountpoint.

    # mount /dev/vda3 /mnt
    # mkdir /mnt/boot
    # mount /dev/vda2 /mnt/boot

Install base system, fstab and chroot on.

    # pacstrap /mnt base
    # genfstab -U /mnt >> /mnt/etc/fstab
    # arch-chroot /mnt

Base configuration:

    # ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
    # hwclock --systohc
    # echo "fr_FR.UTF-8 UTF-8" >> /etc/locale.gen
    # locale-gen
    # echo "LANG=fr_FR.UTF-8" > /etc/locale.conf
    # echo "LC_COLLATE=C" >> /etc/locale.conf
    # echo "KEYMAP=fr-latin9" > /etc/vconsole.conf
    # echo "archvm" > /etc/hostname
    # echo "127.0.1.1	archvm.localdomain	archvm" >> /etc/hosts

Install grub as bootloader:

    # pacman -S grub
    # grub-install --target=i386-pc /dev/vda
    # grub-mkconfig -o /boot/grub/grub.cfg

Set your new password before turning of the virtual system.

    # passwd

And exit to chroot and shutdown.

    # exit
    # systemctl poweroff

### Restart on your new system with our script.

    $ archvm
