---
layout: post-detail
title: Make your own iso with archiso and ZFS support
date: 2018-04-26
categories: zfs
description: Create your own iso with archiso and QEMU, include your own tools and scripts.
img-url: ''
comments: true
---


For the rest of this wiki, i will use QEMU, you can switch on virtualbox or what you want. I've alrealy create the archlinux system in an other post [here](https://szorfein.github.io/archlinux/qemu/archlinux-with-qemu/).  

So, i run my script for start the vm.

    $ archvm

# Network

Control the network, launch dhcpcd if need:

    # ping -c 1 gentoo.org
    # dhcpcd ens3

# Dependencies

Next step is to install somes dependencies:
  
    # pacman -S archiso git

# Clone Gentoo-ZFS

My repository contain the necessary to install zfs-lts and somes script i need to autoformat a hdd with zfs.

    # git clone https://github.com/szorfein/Gentoo-ZFS

# Copy files

Copy the `releng` directory from archlinux to build our custome image.

    # cp -r /usr/share/archiso/configs/releng archlive
    # cd archlive
    # vim pacman.conf
    [archzfs]
    Server = https://archzfs.com/$repo/$arch

Edit the `packages.x86_64` to include the good kernel, add:

    archzfs-linux

And update somes files with my repository.

    # cp -a Gentoo-ZFS/archiso/* archlive

# Repo ArchZFS

Before build iso, you need import the gpg key for `pacman` from [archzfs](https://github.com/archzfs/archzfs/wiki#using-the-archzfs-repository).

    # pacman-key -r DDF7DB817396A49B2A2723F7403BD972F75D9D76 --keyserver hkp://pool.sks-keyservers.net:80
    # pacman-key --lsign-key DDF7DB817396A49B2A2723F7403BD972F75D9D76 --keyserver hkp://pool.sks-keyservers.net:80
    # pacman -Syy

# Build ISO

    # cd archlive
    # mkarchiso -v -o out .

# Rebuild ISO

If you need rebuild iso, then, do it:

    # cd archlive
    # rm -v work/*

And rebuild iso normally.

# Retrieve the iso from qemu image

So, close your vm if it's run.

    # systemctl poweroff

Next, check `Archvm.raw` with `fdisk` utility.

    $ sudo fdisk -l Archvm.raw
    
```
Disque vms/Arch.raw : 10 GiB, 10737418240 octets, 20971520 secteurs
Unités : secteur de 1 × 512 = 512 octets
Taille de secteur (logique / physique) : 512 octets / 512 octets
taille d'E/S (minimale / optimale) : 512 octets / 512 octets
Type d'étiquette de disque : gpt
Identifiant de disque : F49B432F-0EC9-4B4A-86E1-B3E1D6B8646D

Périphérique    Début      Fin Secteurs Taille Type
vms/Arch2.raw1   2048     6143     4096     2M Amorçage BIOS
vms/Arch2.raw2   6144   530431   524288   256M Système de fichiers Linux
vms/Arch2.raw3 530432 20971486 20441055   9,8G Système de fichiers Linux
```

You only need sector size, 512 here, and the beginning of partition 3: 530432.

    $ bc 
    530432*512 = 271581184

You have to mount this partition for retrieve the iso image:

    $ sudo mount -o loop,offset=271581184 Archvm.raw /mnt
    $ cp -a /mnt/root/archlive/out/build/archiso.iso ~
    $ sudo umount /mnt

## Troubleshooting

**`spl-linux-lts requires linux-lts=X.XX.XX`**  

With this error, you have nothing to do, just wait some days, it's happen when `linux-lts` is update from archlinux, the team of archzfs have to update somes file like [src/kernels/linux-lts.sh](https://github.com/archzfs/archzfs/blob/8e5583632b26cef809abc91eb28afb950dafa989/src/kernels/linux-lts.sh).
Once the repository is update, remove older build packages and rebuild iso: 

    # cd archlive
    # rm -rf out/*
    # ./build.sh -v

Post an issue to [github](https://github.com/szorfein/szorfein.github.io/issue).
