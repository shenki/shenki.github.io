---
layout: post
title: Booting AST2600 using UART5 debug mode
---

The AST2600 has a recovery mode where it can boot a payload from the UART. This
must be enabled by a strapping pin.

If you haven't set up a development machine these tools will get you close:

```
sudo apt install gcc-arm-linux-gnueabi make u-boot-tools git minicom
```

## Build u-boot and SPL

```
git clone https://github.com/openbmc/u-boot -b v2019.04-aspeed-openbmc
cd u-boot
ARCH=ARM CROSS_COMPILE=arm-linux-gnueabi- make ast2600_openbmc_spl_defconfig
ARCH=ARM CROSS_COMPILE=arm-linux-gnueabi- make -j8
```

## Prepare the images

```
tools/gen_uart_booting_image.sh spl/u-boot-spl.bin u-boot-spl-recovery.bin
mkimage -A arm -O u-boot -C none -a 0x83000000 -e 0x83000000 -d u-boot.bin u-boot-recovery.bin
```

## Prepare system
Power off your system. Set the "Boot from UART" strap.  Power it back on.

```
minicom -D /dev/ttyUSB0 -b 115200
```

## Load recovery SPL
```
cat u-boot-spl-recovery.bin > /dev/ttyUSB0
```

You will see:

```
123P

U-Boot SPL 2019.04-05549-g08e425111892 (Apr 24 2020 - 13:17:07 +0930)
Trying to boot from UART
C
```

## Send u-boot with minicom

`ctrl+a s` in minicom to send a file with ymodem. When complete it will jump to
u-boot proper.

```
CCC - CRC mode, 4345(SOH)/0(STX)/0(CAN) packets, 6 retries
Loaded 555824 bytes

U-Boot 2019.04-05549-g08e425111892 (Apr 24 2020 - 13:32:52 +0930)

SOC: AST2600-A1
```

From there you will be in a u-boot environment which can be used to recover the system.
