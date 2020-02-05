---
layout: post
title: Debugging u-boot in Qemu with GDB after relocation
---

Debugging u-boot in Qemu allows for JTAG-style single stepping without the
hassle of setting up hardware. However, u-boot relocates itself half way
through running, which requires some fiddling in u-boot.


If there's an easier way to do what I'm describing here then please let me
know.

## Install tools

I'm using Debian testing (Bullseye). You can use any distro which ships qemu
4.2 and a gdb that is built for arm.

```
apt install gdb-multiatch qemu-system-arm arm-linux-gnueabi-gcc
```

## Build u-boot

```
git clone git://git.denx.de/u-boot.git
make evb-ast2500_defconfig
CROSS_COMPILE="ccache arm-linux-gnueabi-"  make -j8
```

## Create a SPI FLASH image

```
dd if=/dev/zero of=test.img count=32 bs=1M
dd if=u-boot.bin of=test.img conv=notrunc
```

## Qemu command line

```
qemu-system-arm -M ast2500-evb -nographic \
 -drive file=test.img,format=raw,if=mtd,readonly \
 -s -S
```

## GDB

```
gdb-multiarch u-boot -ex "target remote localhost:1234"

(gdb) break relocate_done
Breakpoint 1 at 0xb28: file arch/arm/lib/relocate.S, line 137.
(gdb) c
Continuing.

Breakpoint 1, relocate_code () at arch/arm/lib/relocate.S:137
137		bx	lr
(gdb) print/x ((gd_t *)$r9)->relocaddr
$1 = 0xbf794000
(geb) add-symbol-file u-boot $1
add symbol table from file "u-boot" at
        .text_addr = 0xbf7a8000
(y or n) y
Reading symbols from u-boot...
```

From there you can set breakpoints, and `continue` execution of u-boot.
