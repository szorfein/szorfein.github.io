---
layout: post-detail
title: Archlinux with qemu in less of 10min.
date: 2017-09-16
categories: archlinux qemu
description: We're create a virtual image of archlinux with qemu in less of 10min.
img-url: https://i.imgur.com/VTg4vo0m.png
comments: true
---

# Download the last iso

We need the official iso to start, download the last on [archlinux]( https://www.archlinux.org/download/).

# Create a virtual Hard disk drive

We create a HDD of 8G here:

    $ qemu-img create -f raw ArchLinux.raw 8G

# Create a little script for QEMU

With our favorite text editor, create a script bellow:

    $ vim archvm

```
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
```

And set it executable.

    $ chmod +x archvm

# Boot on archlinux

Now that we have a script, we simply add two arguments to change the boot order:

    $ archvm -boot d -cdrom archlinux.iso

# Archlinux install

If you don't understand somes command line, please look to the  official [wiki](https://wiki.archlinux.org/index.php/Installation_Guide).

Change keyboard layout for non-english

    # loadkeys fr
    # timedatectl set-ntp true

Partition ArchLinux.raw, the name on qemu is vda bellow sda.  
I just following the archlinux of the guide for BIOS system.  
Partition 1 = boot, 2 = system, 3 = swap.

    # gdisk /dev/vda
        n # partition 1 [enter], from beginning [enter], [+1M], code [EF02]
        n # partition 2 [enter], from beginning [enter], [-256M], code [enter]
        n # partition 3 [enter], from beginning [enter], [enter], code [enter]
        w

Format with ext4, not need a better filesystem.

    # mkfs.ext4 /dev/vda2

And the swap.

    # mkswap /dev/vda3   
    # swapon /dev/vda3

Set mountpoint.

    # mount /dev/vda2 /mnt

Install base system, fstab and chroot on.

    # pacstrap /mnt base linux linux-firmware
    # genfstab -U /mnt >> /mnt/etc/fstab
    # arch-chroot /mnt

Base configuration:

    # ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
    # hwclock --systohc
    # echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
    # locale-gen
    # echo "LANG=en_US.UTF-8" > /etc/locale.conf
    # echo "KEYMAP=fr" > /etc/vconsole.conf
    # echo "archvm" > /etc/hostname

Install grub as bootloader with few tools:

    # pacman -S grub vim
    # grub-install /dev/vda
    # grub-mkconfig -o /boot/grub/grub.cfg

Set your new password before turning off the vm.

    # passwd

And shutdown.

    # exit
    # systemctl poweroff

# Restart with our script

    $ archvm

# Configure the network
With systemd, create a file for the DHCP:

    # cat > /etc/systemd/network/50-dhcp.network
    [Match]
    Name=en*

    [Network]
    DHCP=yes
    ^D

And another file for DNS query:

    # cat >> /etc/systemd/resolved.conf
    [Resolve]
    DNS = 1.1.1.1
    ^D

Start them:

    # systemctl start systemd-networkd
    # systemctl start systemd-resolved

