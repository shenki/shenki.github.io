---
layout: post
title: Custom kernel on OpenBMC eMMC system
---

This post has instructions for creating an ITS and using it to build a signed
FIT for booting on a OpenBMC system that requires a signed kernel FIT.


## Background

Some of the Power10 systems from IBM use OpenBMC for their management
processors. These run the AST2600 BMC, and have eMMC as the boot media and
root file system.

The system contains two sides, a and b, that are selected by a u-boot script depending on
the u-boot environment variable 'bootside'. This maps to the following GPT disk layout:


```
  --- - ----- -------- -------- -------- -------- ------ -------- -------
 |GPT| | env | boot-a | boot-b | rofs-a | rofs-b | rwfs | hostfw |GPT-sec|
  --- - ----- -------- -------- -------- -------- ------ -------- -------
         1MB    64MB     64MB     1GB      1GB     7GB     5GB
```

(From (https://github.com/openbmc/openbmc/blob/master/meta-aspeed/wic/emmc-aspeed.wks.in))

In addition, OpenBMC now uses the u-boot 'vboot' feature to verify the image it
loads. u-boot contains a public key that is used to verify the signatures
present in the FIT image.

For the public OpenBMC tree, a hardcoded key pair is used for development. This
means we can build our own kernels and sign them with that key, and load them
on machines that run development builds of OpenBMC.

## Prerequisites

To do this we need an Image Signing Tree (.its) that describes how to sign the
image. We can use the one from the [build artifacts](https://jenkins.openbmc.org/view/latest/job/latest-master/label=docker-builder,target=p10bmc/lastSuccessfulBuild/artifact/openbmc/build/tmp/deploy/images/p10bmc/fitImage-its-obmc-phosphor-initramfs-p10bmc-p10bmc) or write our own:


```
/dts-v1/;

/ {
        description = "How good are computers?";
        #address-cells = <1>;

        images {
                kernel-1 {
                        description = "Linux kernel";
                        data = /incbin/("arch/arm/boot/zImage");
                        type = "kernel";
                        arch = "arm";
                        os = "linux";
                        compression = "none";
                        load = <0x80001000>;
                        entry = <0x80001000>;
                        hash-1 {
                                algo = "sha512";
                        };
                };
                fdt-aspeed-bmc-ibm-rainier.dtb {
                        description = "Flattened Device Tree blob";
                        data = /incbin/("arch/arm/boot/dts/aspeed-bmc-ibm-rainier.dtb");
                        type = "flat_dt";
                        arch = "arm";
                        compression = "none";
                        hash-1 {
                                algo = "sha512";
                        };
                };
                ramdisk-1 {
                        description = "obmc-phosphor-initramfs";
                        data = /incbin/("obmc-phosphor-initramfs-p10bmc.cpio.xz");
                        type = "ramdisk";
                        arch = "arm";
                        os = "linux";
                        compression = "none";
                        hash-1 {
                                algo = "sha512";
                        };
                };
	};

        configurations {
                default = "conf-aspeed-bmc-ibm-rainier.dtb";
                conf-aspeed-bmc-ibm-rainier.dtb {
			description = "1 Linux kernel, FDT blob, ramdisk";
			kernel = "kernel-1";
			fdt = "fdt-aspeed-bmc-ibm-rainier.dtb";
			ramdisk = "ramdisk-1";
                        hash-1 {
                                algo = "sha512";
                        };
                        signature-1 {
                                algo = "sha512,rsa4096";
                                key-name-hint = "rsa_oem_fitimage_key";
				sign-images = "kernel", "fdt", "ramdisk";
                        };
                };
	};
};
```

Place this as `p10bmc.its` in your kernel build tree.

Also download the [signing key](https://github.com/openbmc/openbmc/raw/master/meta-aspeed/recipes-kernel/linux/linux-aspeed/rsa_oem_fitimage_key.key) from the openbmc repository, and an [initramfs](https://jenkins.openbmc.org/view/latest/job/latest-master/label=docker-builder,target=p10bmc/lastSuccessfulBuild/artifact/openbmc/build/tmp/deploy/images/p10bmc/obmc-phosphor-initramfs-p10bmc.cpio.xz) from CI.

Place `rsa_oem_fitimage_key.key` and `obmc-phosphor-initramfs-p10bmc.cpio.xz` in your kernel build tree too.

## Signing

With all of these pieces in place we can produce a signed FIT:

```
mkimage -k . -f p10bmc.its image.fit
```

The output should look as follows:

```
FIT description: How good are computers?
Created:         Fri Sep  3 12:04:12 2021
 Image 0 (kernel-1)
  Description:  Linux kernel
  Created:      Fri Sep  3 12:04:12 2021
  Type:         Kernel Image
  Compression:  uncompressed
  Data Size:    3812160 Bytes = 3722.81 KiB = 3.64 MiB
  Architecture: ARM
  OS:           Linux
  Load Address: 0x80001000
  Entry Point:  0x80001000
  Hash algo:    sha512
  Hash value:   1b1e6428c8f0b18dd2b92ddabe751fa99e786e301ea0017a65823c81db77ed8be66798599d9673d76a03c712aedcbddd44d63ce955ca1b92e44033f22805762e
 Image 1 (fdt-aspeed-bmc-ibm-rainier.dtb)
  Description:  Flattened Device Tree blob
  Created:      Fri Sep  3 12:04:12 2021
  Type:         Flat Device Tree
  Compression:  uncompressed
  Data Size:    57116 Bytes = 55.78 KiB = 0.05 MiB
  Architecture: ARM
  Hash algo:    sha512
  Hash value:   1e9d2c7ab1df61e42ba91b020634b0d4fc1e8998433126f7a11166a10fb8651a269ef9a3498602e7f8ddf17bd7cbaa3e5ae17a8698cb89c3350a599b5ae67753
 Image 2 (ramdisk-1)
  Description:  obmc-phosphor-initramfs
  Created:      Fri Sep  3 12:04:12 2021
  Type:         RAMDisk Image
  Compression:  uncompressed
  Data Size:    2883560 Bytes = 2815.98 KiB = 2.75 MiB
  Architecture: ARM
  OS:           Linux
  Load Address: unavailable
  Entry Point:  unavailable
  Hash algo:    sha512
  Hash value:   b7dc3392eec60dfb4c25cc678d2bb00e01c404ccf1212a3a08beedd051cf7ad533c46d11bf5651c9a06a8f116f41e47bbcdf56107f65339ae27ee49816066c0d
 Default Configuration: 'conf-aspeed-bmc-ibm-rainier.dtb'
 Configuration 0 (conf-aspeed-bmc-ibm-rainier.dtb)
  Description:  1 Linux kernel, FDT blob, ramdisk
  Kernel:       kernel-1
  Init Ramdisk: ramdisk-1
  FDT:          fdt-aspeed-bmc-ibm-rainier.dtb
  Hash algo:    sha512
  Hash value:   unavailable
  Sign algo:    sha512,rsa4096:rsa_oem_fitimage_key
  Sign value:   <big number>
  Timestamp:    Fri Sep  3 12:04:12 2021
```

### Tips
I modify my its to point to a location where I keep copies of the initramfs so
I can test various kernel trees without copying files around. Similarly I
modify the `-k` key argument to point to a local checkout of the OpenBMC source
tree to save copying it around.

## Booting

The easiest method of using your shiny signed FIT is to install it in `/boot` on a live machine.

Check to see which side is the current:
```
fw_printenv bootside
bootside=a
```

Copy it over, mount the boot partition and copy in the new FIT:
```
scp p10bmc.fit bmc:
ssh bmc
mount /dev/disk/by-partlabel/boot-a /boot
mv /boot/fitimage{,.old}
mv p10bmc.fit /boot/fitImage
reboot
```
