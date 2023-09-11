---
layout: post
title: Testing and OpenBMC Kernel on eMMC
---


This guide provides notes on how to put a new kernel FIT on a OpenBMC system
that uses the eMMC layout. It assumes you've built an OpenBMC FIT containing
the device tree, initrd and kernel zImage for testing.

The OpenBMC file system layout as used by IBM's creatively named "p10bmc"
system has the following layout:

```
# parted /dev/mmcblk0 p
Model: MMC S0J56X (sd/mmc)
Disk /dev/mmcblk0: 15.9GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 1      20.5kB  1069kB  1049kB               primary  msftdata
 2      1069kB  68.2MB  67.1MB  ext4         boot-a
 3      68.2MB  135MB   67.1MB  ext4         boot-b
 4      135MB   1209MB  1074MB  ext4         rofs-a
 5      1209MB  2283MB  1074MB  ext4         rofs-b
 6      2283MB  9799MB  7516MB  ext4         rwfs
 7      9799MB  15.2GB  5369MB  ext4         hostfw
```

By default, only these filesystems are mounted:
```
# mount |grep mmc
/dev/mmcblk0p4 on / type ext4 (ro,relatime)
/dev/mmcblk0p6 on /var type ext4 (rw,relatime)
/dev/mmcblk0p6 on /home type ext4 (rw,relatime)
/dev/mmcblk0p7 on /media/hostfw type ext4 (rw,relatime)
```

It doesn't mount the boot filesystems as they are not required to be read or
written at normal runtime.

The system has a/b updates for the rootfs and kernel FIT. If we want to test
our kernel, first we must pick which side is current:

```
# fw_printenv bootside
bootside=a
```

And mount it:
```
# mount /dev/disk/by-partlabel/boot-a /boot/
```

The u-boot environment is configured to look for a file called fitImage at root of the boot partition:

```
ext4load mmc 0:${bootpart} ${loadaddr} fitImage
```

If there's a risk that the image might be broken, or if you want to be polite
and replace the old image once you're done testing, back up the old image:

```
# mv /boot/fitImage /boot/fitImage.backup
```

And copy over your new image:
```
rsync -P myAwesomeKernelFit root@bmc:/boot/fitImage
```

From there, check the image copied successfully by checking the size or the
md5sum, and reboot the BMC.
