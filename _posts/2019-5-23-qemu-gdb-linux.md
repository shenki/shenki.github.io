---
layout: post
title: Linux kernel logs with GDB with Qemu
---

When your Linux kernel is misbehaving, you often look to the output of 'dmesg'
to see what went wrong. If you can't get to userspace, hopefully you have a
serial port to see the latest output.

When neither of those are options, you can instead use Qemu and gdb to extract
the kernel log from memory. The kernel source tree contains scripts to help you
do this:

```
 ./scripts/config -e CONFIG_GDB_SCRIPTS
 make
```

Launch your kernel in Qemu with the `-s` option, which means start a gdb server
listening on TCP port 1234.

```
qemu-system-ppc64 -M powernv -smp 2 -m 2G -nographic -kernel /path/to/kernel/vmlinux -s
```

Connect to the gdb server using gdb. If you're running a Debian based distro,
gdb-multiarch will be useful, as you need to ensure the gdb you use supports
the target architecture. 

From there you can use the 'lx' family of commands that vmlinux-gdb.py has
added to your GDB environment.


```
cd /path/to/kernel
echo add-auto-load-safe-path $PWD/scripts/gdb/vmlinux-gdb.py >> ~/.gdbinit
$ gdb-multiarch vmlinux --silent -ex "target remote localhost:1234"
Reading symbols from vmlinux...done.
(gdb) lx-dmesg 
[    0.000000] dt-cpu-ftrs: setup for ISA 2070
[    0.000000] dt-cpu-ftrs: not enabling: subcore (unknown and unsupported by kernel)
[    0.000000] dt-cpu-ftrs: final cpu/mmu features = 0x000000ff8f5db1a7 0x3c006001
[    0.000000] hash-mmu: Page sizes from device-tree:
[    0.000000] hash-mmu: base_shift=12: shift=12, sllp=0x0000, avpnm=0x00000000, tlbiel=1, penc=0
[    0.000000] hash-mmu: base_shift=12: shift=16, sllp=0x0000, avpnm=0x00000000, tlbiel=1, penc=7
[    0.000000] hash-mmu: base_shift=12: shift=24, sllp=0x0000, avpnm=0x00000000, tlbiel=1, penc=56
[    0.000000] hash-mmu: base_shift=16: shift=16, sllp=0x0110, avpnm=0x00000000, tlbiel=1, penc=1
[    0.000000] hash-mmu: base_shift=16: shift=24, sllp=0x0110, avpnm=0x00000000, tlbiel=1, penc=8
[    0.000000] hash-mmu: base_shift=24: shift=24, sllp=0x0100, avpnm=0x00000001, tlbiel=0, penc=0
[    0.000000] hash-mmu: base_shift=34: shift=34, sllp=0x0120, avpnm=0x000007ff, tlbiel=0, penc=3
[    0.000000] Page orders: linear mapping = 24, virtual = 16, io = 16, vmemmap = 24
[    0.000000] Using 1TB segments
[    0.000000] hash-mmu: Initializing hash mmu with SLB
[    0.000000] Linux version 5.2.0-rc1-00165-g54dee406374c (joel@voyager) (gcc version 8.3.0 (Debian 8.3.0-2)) #1 SMP Thu May 23 15:17:40 ACST 2019
[    0.000000] OPAL: Found non-mapped LPC bus on chip 0
[    0.000000] Using PowerNV machine description
[    0.000000] printk: bootconsole [udbg0] enabled
[    0.000000] CPU maps initialized for 1 thread per core
[    0.000000]  (thread shift is 0)
[    0.000000] Allocated 3208 bytes for 1 pacas
[    0.000000] -----------------------------------------------------
[    0.000000] phys_mem_size     = 0x80000000
[    0.000000] dcache_bsize      = 0x80
[    0.000000] icache_bsize      = 0x80
[    0.000000] cpu_features      = 0x000000ff8f4db1a7
[    0.000000]   possible        = 0x0000fbffcf5fb1a7
```

Here's another example using an ARM machine:
```
CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm make aspeed_g5_defconfig
make
qemu-system-arm -M ast2500-evb -m 512 -nographic -kernel arch/arm/boot/zImage -dtb arch/arm/boot/dts/aspeed-ast2500-evb.dtb -s
```
```
gdb-multiarch vmlinux --silent -ex "target remote localhost:1234"
Reading symbols from vmlinux...done.
(gdb) lx-cmdline
console=ttyS4,115200 earlyprintk
(gdb) lx-dmesg
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.1.3-00115-g49af67f31fd4 (joel@voyager) (gcc version 8.3.0 (Debian 8.3.0-2)) #19 Wed May 22 15:58:57 ACST 2019
[    0.000000] CPU: ARMv6-compatible processor [410fb767] revision 7 (ARMv7), cr=00c5387d
[    0.000000] CPU: VIPT aliasing data cache, unknown instruction cache
[    0.000000] OF: fdt: Machine model: Romulus BMC
[    0.000000] Memory policy: Data cache writeback
[    0.000000] Reserved memory: created CMA memory pool at 0x9d000000, size 16 MiB
[    0.000000] OF: reserved mem: initialized node framebuffer, compatible id shared-dma-pool
[    0.000000] Reserved memory: created CMA memory pool at 0x96000000, size 32 MiB
[    0.000000] OF: reserved mem: initialized node jpegbuffer, compatible id shared-dma-pool
[    0.000000] cma: Reserved 16 MiB at 0x9c000000
[    0.000000] On node 0 totalpages: 110336
[    0.000000]   Normal zone: 990 pages used for memmap
[    0.000000]   Normal zone: 0 pages reserved
[    0.000000]   Normal zone: 110336 pages, LIFO batch:31
[    0.000000] CPU: All CPU(s) started in SVC mode.
[    0.000000] random: get_random_bytes called from start_kernel+0x88/0x4c4 with crng_init=0
[    0.000000] pcpu-alloc: s0 r0 d32768 u32768 alloc=1*32768
[    0.000000] pcpu-alloc: [0] 0
--Type <RET> for more, q to quit, c to continue without paging--
```
