---
layout: post
title: Petitboot hacks
---

A safe way to try out a new petitboot release on your machine is to add it to
your disk, and chainload from the in-flash version to the one on disk. This
uses the same well-tested mechanim as booting your operating system, but
doesn't require touching flash. If something goes wrong, just reboot!

If you're running offical firmware releases on your OpenPower machine, you are
probably missing out on the latest and greatest features, bugfixes and user
experience that the master branch contains. HEAD or dead, as someone said the
other day.

## Building your own image

You should grab op-build and create your own image and build `opal_defconfig`.
This builds a reduced configuration that contains just the kernel and petitboot
bits we require, without rebuilding hostboot etc.

```
git clone --recurse-submodules https://github.com/open-power/op-build
cd op-build
. op-build-env
op-build opal_defconfig
op-build
sudo cp output/build/linux-*/vmlinux /boot/vmlinux-5000-petitboot
sudo cp output/images/rootfs.cpio.xz /boot/vmlinux-5000-petitboot
sudo update-grub
```

## Try a pre-built image

If you want to try this with a pre-built image, here's one from today that I created:

```
wget https://ozlabs.org/~joel/pb/vmlinux-4.19.30-petitboot
wget https://ozlabs.org/~joel/pb/initrd.img-4.19.30-petitboot
sudo mv vmlinux-4.19.30-petitboot initrd.img-4.19.30-petitboot /boot/
sudo update-grub
```

## Booting
Once you've run `update-grub`, reboot. When your machine comes up, there will
be a new boot entry with 'petitboot' in the version that you can select.

![Petitboot menu]({{ site.url }}/images/pb-custom.png)

This has been tested on my Ubuntu system. Please get in touch if it works on
Fedora or your distro of choice.

## Removing your changes

To undo this:

```
sudo rm /boot/vmlinux-4.19.30-petitboot /boot/initrd.img-4.19.30-petitboot  
sudo update-grub
```
