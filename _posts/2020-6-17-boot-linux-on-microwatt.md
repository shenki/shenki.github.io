---
layout: post
title: Preview Linux on the Microwatt PowerPC CPU
---

This describes how to boot Linux on
[Microwatt](https://github.com/antonblanchard/microwatt/), running on the Arty
A7 FPGA.

![]({{ site.url }}/images/microwatt-title.png)

WARNING: The SoC is undergoing rapid development. These instructions work
today, with microwatt 12a257f01ed, but by tomorrow the firmware, memory layout,
kernel or some other feature will have been added making these instructions out
of date.

Note that this is the Microwatt SoC and not a [LiteX SoC](https://github.com/enjoy-digital/litex/)
with microwatt inside of it. The LiteX SoC is capable of booting Linux but
support is not yet enabled.

Step zero is to install the Xilinx Vivardo toolchain with support for the Artix 7 FPGAs.

## Build the FPGA image
```
pip install fusesoc
git clone https://github.com/antonblanchard/microwatt
cd microwatt
cat <<EOF > fusesoc.conf
[library.microwatt]
location = $PWD
sync-uri = $PWD
sync-type = local
auto-sync = true
EOF
```

Attach the Arty via USB and run the following:

```
source ~/tools/Xilinx/Vivado/2019.1/settings64.sh
fusesoc run --target=arty_a7-35 microwatt --no_bram --memory_size=0
```

This will build the bitstream and load it to your board. You will see the following:

```
Welcome to Microwatt !

 Soc signature: f00daa5500010001
  Soc features: UART DRAM BRAM SPIFLASH
          BRAM: 16 KB
          DRAM: 256 MB
     DRAM INIT: 0 KB
           CLK: 100 MHz
  SPI FLASH ID: 20ba18 Micron [quad IO mode]
 SPI FLASH OFF: 0x300000 bytes

LiteDRAM built from Migen b1b2b29 and LiteX 20ff2462
Initializing SDRAM...
SDRAM now under software control
SDRAM now under software control
Read leveling:
...
best: m1, b09 delays: 20+-07
SDRAM now under hardware control
Memtest OK
Memspeed Writes: 457Mbps Reads: 524Mbps
Booting from DRAM at 0
```

As we do not have any code loaded, it will then reset and print the same message forever.

This is where firmware would take over, and allow you the option of booting
from Flash, UART, network or BRAM or SDRAM. However that feature is a work in
progress, so instead we will build a Linux image that can be written to the SPI
Flash and booted from there.

### Building Linux

Linux needs support for the microwatt platform which is not yet upstream. The
[patches have been posted](https://lore.kernel.org/linuxppc-dev/20200509050103.GA1464954@thinks.paulus.ozlabs.org/)
to linuxppc-dev and expanded upon in a git tree on kernel.org.

Importantly this tree allows us to build an image that contains a [device tree
for the system](https://git.kernel.org/pub/scm/linux/kernel/git/joel/microwatt.git/tree/arch/powerpc/boot/dts/microwatt.dts?h=microwatt-5.7),
as the firmware has not yet been written to provide one to Linux. The
resulting dtbimage contains a copy of the device tree, and can be
loaded to anywhere in memory above address 0.

If the CPU jumps to this location, the kernel will the extract itself to
address 0 and set up the machine state to enter the kernel image proper (sets
r3 to the location of the device tree, and r5 to zero to indicate there is no
openfirmware to call into).

```
apt install gcc-powerpc64le-linux-gnu
git clone -b microwatt-5.7 https://git.kernel.org/pub/scm/linux/kernel/git/joel/microwatt.git
cd microwatt
wget https://ozlabs.org/~paulus/rootfs.cpio.xz
unxz rootfs.cpio.xz
CROSS_COMPILE="ccache powerpc64le-linux-gnu-" ARCH=powerpc make -j8 O=microwatt microwatt_defconfig
CROSS_COMPILE="ccache powerpc64le-linux-gnu-" ARCH=powerpc make -j8 O=microwatt
```

### Flashing the Arty

```
apt install openocd
cd microwatt
$ python3 openocd/flash-arty -f a35 -t bin build/microwatt_0/arty_a7-35-vivado/microwatt_0.bit -a 0x0
init; jtagspi_init 0 {microwatt/openocd/bscan_spi_xc7a35t.bit}; ; jtagspi_program {build/microwatt_0/arty_a7-35-vivado/microwatt_0.bit} 0x0 bin; fpga_program; exit
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
none separate
adapter speed: 25000 kHz
Info : auto-selecting first available session transport "jtag". To override use 'transport select <transport>'.
fpga_program
Info : ftdi: if you experience problems at higher adapter clocks, try the command "ftdi_tdo_sample_edge falling"
Info : clock speed 25000 kHz
Info : JTAG tap: xc7.tap tap/device found: 0x0362d093 (mfg: 0x049 (Xilinx), part: 0x362d, ver: 0x0)
loaded file microwatt/openocd/bscan_spi_xc7a35t.bit to pld device 0 in 0s 133343us
Info : JTAG tap: xc7.tap tap/device found: 0x0362d093 (mfg: 0x049 (Xilinx), part: 0x362d, ver: 0x0)
Info : Found flash device 'micron n25q128' (ID 0x0018ba20)
flash 'jtagspi' found at 0x00000000
auto erase enabled
Info : Found flash device 'micron n25q128' (ID 0x0018ba20)
Info : Found flash device 'micron n25q128' (ID 0x0018ba20)
Info : Found flash device 'micron n25q128' (ID 0x0018ba20)
Info : sector 0 took 247 ms
Info : sector 1 took 257 ms
Info : sector 2 took 259 ms
Info : sector 3 took 263 ms
...
wrote 1572864 bytes from file build/microwatt_0/arty_a7-35-vivado/microwatt_0.bit in 16.513203s (93.016 KiB/s)
```

The same to flash the kernel:
```
$ python3 openocd/flash-arty -f a35 -t bin -a 0x300000 ~/microwatt/microwatt/arch/powerpc/boot/dtbImage.microwatt.elf
init; jtagspi_init 0 {microwatt/openocd/bscan_spi_xc7a35t.bit}; ; jtagspi_program {/home/joel/dev/microwatt/arch/powerpc/boot/dtbImage.microwatt.elf} 0x300000 bin; fpga_program; exit
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
none separate
adapter speed: 25000 kHz
Info : auto-selecting first available session transport "jtag". To override use 'transport select <transport>'.
fpga_program
Info : ftdi: if you experience problems at higher adapter clocks, try the command "ftdi_tdo_sample_edge falling"
Info : clock speed 25000 kHz
Info : JTAG tap: xc7.tap tap/device found: 0x0362d093 (mfg: 0x049 (Xilinx), part: 0x362d, ver: 0x0)
loaded file microwatt/openocd/bscan_spi_xc7a35t.bit to pld device 0 in 0s 134233us
Info : JTAG tap: xc7.tap tap/device found: 0x0362d093 (mfg: 0x049 (Xilinx), part: 0x362d, ver: 0x0)
Info : Found flash device 'micron n25q128' (ID 0x0018ba20)
flash 'jtagspi' found at 0x00000000
auto erase enabled
Info : Found flash device 'micron n25q128' (ID 0x0018ba20)
Info : Found flash device 'micron n25q128' (ID 0x0018ba20)
Info : Found flash device 'micron n25q128' (ID 0x0018ba20)
Info : sector 48 took 232 ms
Info : sector 49 took 264 ms
Info : sector 50 took 289 ms
Info : sector 51 took 266 ms
Info : sector 52 took 265 ms
...
wrote 2555904 bytes from file microwatt/arch/powerpc/boot/dtbImage.microwatt in 27.095570s (92.118 KiB/s)
  flash verify_bank bank_id filename offset

```

From here you can reset the board using the PROG button next to the USB cable.
You should see the following:

```
Welcome to Microwatt !

 Soc signature: f00daa5500010001
  Soc features: UART DRAM SPIFLASH
          DRAM: 256 MB
     DRAM INIT: 0 KB
           CLK: 100 MHz
  SPI FLASH ID: 20ba18
  TEST DUAL: 00 09 0f f0
  TEST QUAD: 00 09 0f f0
  TEST MAP : f00f0900
 SPI FLASH OFF: 0x300000 bytes

LiteDRAM built from Migen b1b2b29 and LiteX 20ff2462
Initializing SDRAM...
SDRAM now under software control
SDRAM now under software control
Read leveling:
m0, b00: |00000000000000000000000000000000| delays: -
m0, b01: |00000000000000000000000000000000| delays: -
m0, b02: |00000000000000000000000000000000| delays: -
m0, b03: |00000000000000000000000000000000| delays: -
m0, b04: |00000000000000000000000000000000| delays: -
m0, b05: |00000000000000000000000000000000| delays: -
m0, b06: |00000000000000000000000000000000| delays: -
m0, b07: |00000000000000000000000000000000| delays: -
m0, b08: |11111111111100000000000000000000| delays: 06+-06
m0, b09: |00000000000011111111111111100000| delays: 19+-07
m0, b10: |00000000000000000000000000001111| delays: 30+-02
m0, b11: |00000000000000000000000000000000| delays: -
m0, b12: |00000000000000000000000000000000| delays: -
m0, b13: |00000000000000000000000000000000| delays: -
m0, b14: |00000000000000000000000000000000| delays: -
m0, b15: |00000000000000000000000000000000| delays: -
best: m0, b09 delays: 19+-07
m1, b00: |00000000000000000000000000000000| delays: -
m1, b01: |00000000000000000000000000000000| delays: -
m1, b02: |00000000000000000000000000000000| delays: -
m1, b03: |00000000000000000000000000000000| delays: -
m1, b04: |00000000000000000000000000000000| delays: -
m1, b05: |00000000000000000000000000000000| delays: -
m1, b06: |00000000000000000000000000000000| delays: -
m1, b07: |00000000000000000000000000000000| delays: -
m1, b08: |11111111111100000000000000000000| delays: 06+-06
m1, b09: |00000000000001111111111111110000| delays: 20+-07
m1, b10: |00000000000000000000000000000111| delays: 30+-01
m1, b11: |00000000000000000000000000000000| delays: -
m1, b12: |00000000000000000000000000000000| delays: -
m1, b13: |00000000000000000000000000000000| delays: -
m1, b14: |00000000000000000000000000000000| delays: -
m1, b15: |00000000000000000000000000000000| delays: -
best: m1, b09 delays: 20+-07
SDRAM now under hardware control
Memtest OK
Memspeed Writes: 710Mbps Reads: 721Mbps
Trying flash...
Copy segment 0 (0x261008 bytes) to 0x500000
Booting from DRAM at 500000

zImage starting: loaded at 0x0000000000500000 (sp: 0x0000000000763eb0)
Allocating 0x474478 bytes for kernel...
Decompressing (0x0000000000000000 <- 0x0000000000510000:0x0000000000760bef)...
Done! Decompressed 0x439d80 bytes

Linux/PowerPC load:
Finalizing device tree... flat tree at 0x764b40
[    0.000000] printk: bootconsole [udbg0] enabled
 -> early_setup(), dt_ptr: 0x764b40
[    0.000000] dt-cpu-ftrs: setup for ISA 3000
[    0.000000] dt-cpu-ftrs: final cpu/mmu features = 0x00000087800391e1 0x3c006041
[    0.000000] radix-mmu: Page sizes from device-tree:
[    0.000000] radix-mmu: Page size shift = 12 AP=0x0
[    0.000000] radix-mmu: Page size shift = 16 AP=0x5
[    0.000000] radix-mmu: Page size shift = 21 AP=0x1
[    0.000000] radix-mmu: Page size shift = 30 AP=0x2
[    0.000000] radix-mmu: Mapped 0x0000000000000000-0x0000000000600000 with 2.00 MiB pages (exec)
[    0.000000] radix-mmu: Mapped 0x0000000000600000-0x0000000010000000 with 2.00 MiB pages
 <- early_setup()
[    0.000000] Linux version 5.7.0-00012-g42ff5cfb8300 (joel@voyager) (gcc version 9.3.0 (Debian 9.3.0-13), GNU ld (GNU Binutils for Debian)0
[    0.000000] Using microwatt machine description
[    0.000000] -----------------------------------------------------
[    0.000000] phys_mem_size     = 0x10000000
[    0.000000] dcache_bsize      = 0x40
[    0.000000] icache_bsize      = 0x40
[    0.000000] cpu_features      = 0x00000087800391e1
[    0.000000]   possible        = 0x0003fbefcb5fb1a5
[    0.000000]   always          = 0x00000003800081a1
[    0.000000] cpu_user_features = 0xc4002102 0x88800000
[    0.000000] mmu_features      = 0x3c006041
[    0.000000] firmware_features = 0x0000000000000000
[    0.000000] vmalloc start     = 0xc008000000000000
[    0.000000] IO start          = 0xc00a000000000000
[    0.000000] vmemmap start     = 0xc00c000000000000
[    0.000000] -----------------------------------------------------
[    0.000000] Top of RAM: 0x10000000, Total RAM: 0x10000000
[    0.000000] Memory hole size: 0MB
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000000000000-0x000000000fffffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000000000-0x000000000fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000000000-0x000000000fffffff]
[    0.000000] On node 0 totalpages: 65536
[    0.000000]   Normal zone: 896 pages used for memmap
[    0.000000]   Normal zone: 0 pages reserved
[    0.000000]   Normal zone: 65536 pages, LIFO batch:15
[    0.000000] pcpu-alloc: s0 r0 d32768 u32768 alloc=1*32768
[    0.000000] pcpu-alloc: [0] 0
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 64640
[    0.000000] Kernel command line:
[    0.000000] Dentry cache hash table entries: 32768 (order: 6, 262144 bytes, linear)
[    0.000000] Inode-cache hash table entries: 16384 (order: 5, 131072 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 236424K/262144K available (1984K kernel code, 208K rwdata, 616K rodata, 1516K init, 233K bss, 25720K reserved, 0K cma)
[    0.000000] SLUB: HWalign=128, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 16
[    0.000000] time_init: decrementer frequency = 100.000000 MHz
[    0.000000] time_init: processor frequency   = 100.000000 MHz
[    0.000159] time_init: 64 bit decrementer (max: 7fffffffffffffff)
[    0.006153] clocksource: timebase: mask: 0xffffffffffffffff max_cycles: 0x171024e7e0, max_idle_ns: 440795205315 ns
[    0.016399] clocksource: timebase mult[a000000] shift[24] registered
[    0.022776] clockevent: decrementer mult[cccccd] shift[27] cpu[0]
[    0.028769] printk: console [hvc0] enabled
[    0.028769] printk: console [hvc0] enabled
[    0.037080] printk: bootconsole [udbg0] disabled
[    0.037080] printk: bootconsole [udbg0] disabled
[    0.046362] pid_max: default: 4096 minimum: 301
[    0.052425] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.058083] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.091572] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.098960] futex hash table entries: 16 (order: -4, 384 bytes, linear)
[    0.136909] clocksource: Switched to clocksource timebase
[    0.938534] workingset: timestamp_bits=62 max_order=16 bucket_order=0
[    1.044815] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 254)
[    1.049723] io scheduler mq-deadline registered
[    1.527837] brd: module loaded
[    1.586247] loop: module loaded
[    1.586796] drmem: No dynamic reconfiguration memory found
[    1.592584] random: get_random_bytes called from init_oops_id+0x34/0x60 with crng_init=0
[    1.616216] Freeing unused kernel memory: 1516K
[    1.618253] This architecture does not have kernel memory protection.
[    1.624550] Run /init as init process
[    1.628396]   with arguments:
[    1.631281]     /init
[    1.633611]   with environment:
[    1.636813]     HOME=/
[    1.639237]     TERM=linux
Starting syslogd: OK
Starting klogd: OK
Running sysctl: OK
Saving random seed: [    2.784155] random: dd: uninitialized urandom read (512 bytes read)
OK
Starting network: ip: socket: Function not implemented
ip: socket: Function not implemented
FAIL

Welcome to Buildroot
buildroot login:
```
