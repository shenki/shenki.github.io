---
layout: post
title: Booting an ASPEED OpenBMC kernel
---

Here are some instructions for booting the OpenBMC 4.13-based kenrel
on the ASPEED ast2500 EVB. They assume you're on Ubuntu 18.04.

Install an ARM compiler and `mkimage`:
```
sudo apt-get install gcc-arm-linux-gnueabi u-boot-tools
```

Get a copy of the kernel and build it:
```
git clone https://github.com/openbmc/linux -b dev-4.13 openbmc-dev
cd openbmc-dev
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
make aspeed_g5_defconfig
make -j $(nproc)
```

This will produce a few build artefacts:
 - `arch/arm/boot/zImage`: compressed kernel image
 - `arch/arm/boot/dts/aspeed-ast2500-evb.dtb`: device tree blob
 - `vmlinux`: kernel ELF for debugging

Download a test initramfs and a script to build a FIT image, and build a FIT
image in `/srv/tftp/evb`:

```
cd openbmc-dev
wget https://ozlabs.org/~joel/evb.its
wget https://ozlabs.org/~joel/broomstick.cpio.xz
mkimage -f evb.its /srv/tftp/evb
```

This assumes you've got a tftp server serving up `/srv/tftp`. If you don't, run
`sudo apt-get install  tftpd` and make sure `/etc/xinetd.d/tftp` contains this:
```
service tftp
{
        protocol        = udp
        port            = 69
        socket_type     = dgram
        wait            = yes
        user            = nobody
        server          = /usr/sbin/in.tftpd
        server_args     = /srv/tftp
        disable         = no
}
```

On the BMC type these commands, assuming your tftp server is at `10.0.2.2`:
```
ast# setenv serverip 10.0.2.2
ast# dhcp evb
ast# bootm
```

It all should look something like this:

```
U-Boot 2016.07 (Jun 07 2018 - 13:21:43 +0000)

       Watchdog enabled
DRAM:  496 MiB
Flash: 32 MiB

In:    serial
Out:   serial
Err:   serial
Net:   aspeednic#0

ast# dhcp evb
aspeednic#0: PHY at 0x00
set_mac_control_register 1453
Found NCSI Network Controller at (0, 0)
Found NCSI Network Controller at (0, 1)
BOOTP broadcast 1
DHCP client bound to address 10.0.2.15 (1 ms)
Using  device
TFTP from server 10.0.2.2; our IP address is 10.0.2.15
Filename 'evb'.
Load address: 0x83000000
Loading: #################################################################
	 #################################################################
	 #################################################################
	 ###########################################################
	 55.7 MiB/s
done
Bytes transferred = 3618975 (37389f hex)
ast# bootm
## Loading kernel from FIT Image at 83000000 ...
   Using 'conf@1' configuration
   Trying 'kernel@1' kernel subimage
     Description:  Linux kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x830000bc
     Data Size:    2460224 Bytes = 2.3 MiB
     Architecture: ARM
     OS:           Linux
     Load Address: 0x80001000
     Entry Point:  0x80001000
   Verifying Hash Integrity ... OK
## Loading ramdisk from FIT Image at 83000000 ...
   Using 'conf@1' configuration
   Trying 'ramdisk@1' ramdisk subimage
     Description:  initramfs
     Type:         RAMDisk Image
     Compression:  Unknown Compression
     Data Start:   0x8325eca8
     Data Size:    1132724 Bytes = 1.1 MiB
     Architecture: ARM
     OS:           Linux
     Load Address: unavailable
     Entry Point:  unavailable
   Verifying Hash Integrity ... OK
## Loading fdt from FIT Image at 83000000 ...
   Using 'conf@1' configuration
   Trying 'fdt@1' fdt subimage
     Description:  device tree
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x83258b9c
     Data Size:    24731 Bytes = 24.2 KiB
     Architecture: ARM
   Verifying Hash Integrity ... OK
   Booting using the fdt blob at 0x83258b9c
   Loading Kernel Image ... OK
   Loading Ramdisk to 9ea84000, end 9eb988b4 ... OK
   Loading Device Tree to 9ea7a000, end 9ea8309a ... OK

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 4.17.0-00082-g39a96bce1561 (joel@aurora) (gcc version 7.3.0 (Ubuntu/Linaro 7.3.0-20ubuntu1)) #3 Fri Jun 8 00:17:21 ACST 2018
[    0.000000] CPU: ARMv6-compatible processor [410fb767] revision 7 (ARMv7), cr=00c5387d
[    0.000000] CPU: VIPT aliasing data cache, unknown instruction cache
[    0.000000] OF: fdt: Machine model: AST2500 EVB
```
