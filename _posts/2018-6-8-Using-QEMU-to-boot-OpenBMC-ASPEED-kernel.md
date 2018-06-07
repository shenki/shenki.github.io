---
layout: post
title: Using QEMU to boot OpenBMC ASPEED kernel 
---

The LTC team at IBM have created a useful model of the ASPEED BMC SoCs and
upstreamed it to QEMU. If you're on a recent distro, you can skip building it
and `sudo apt-get install qemu-system-arm` instead.

To build:
```
git clone https://github.com/legoater/qemu -b aspeed-2.12 qemu-aspeed
cd qemu-aspeed
./configure --target-list=arm-softmmu
make -j $(nproc)
```

Download a flash image:
```
wget https://openpower.xyz/job/openbmc-build/distro=ubuntu,target=evb-ast2500/lastSuccessfulBuild/artifact/deploy/images/evb-ast2500/flash-evb-ast2500
```

This will boot from an emulated flash image:
```
$ ./arm-softmmu/qemu-system-arm -M ast2500evb -m 512 \
 -drive file=flash-evb-ast2500,format=raw,if=mtd \
 -nodefaults -nographic  -serial stdio -net nic \
 -net user,hostname=qemu,ftp=/path/to/tftp/
```

If you have a kernel FIT or u-boot image you would like to test booting. In
this example your image is in `/srv/tftp`, and is called `evb`.
```
$ ./arm-softmmu/qemu-system-arm -M ast2500evb -m 512 \
 -drive file=flash-ast2500evb,format=raw,if=mtd \
 -nodefaults -nographic  -serial stdio -net nic \
 -net user,hostname=qemu,ftp=/srv/tftp/
```

Press any key when u-boot prompts you, and type these three commands:
```
ast# setenv ethaddr DE:AD:CA:FE:BE:EF
ast# dhcp evb
ast# bootm
```
