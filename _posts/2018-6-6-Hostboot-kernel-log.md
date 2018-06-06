---
layout: post
title: Hostboot kernel log
---

On an OpenPower system, the hostboot kernel can be fetched from the host using
[pdbg](https://github.com/open-power/pdbg) to read host memory.

You will need a copy of the symbols contained in hbicore in order to find the address of 
`kernel_printk_buffer`. If you've built locally, that could be found in `hostboot/img/hibcore.elf`:

```
$ nm hostboot/img/hbicore.elf | grep kernel_printk_buffer
0000000000040278 D kernel_printk_buffer
```

If you've grabbed a test build from eg. [openpower.xyz](http://github.com/open-power/op-build/), download the corresponding `host_fw_debug.tar`,
and extract it in a directory as follows:

```
$ tar xf ~/Downloads/host_fw_debug.tar hbicore.syms
$ grep kernel_printk_buffer  hbicore.syms
V,00040278,00000000,00005000,kernel_printk_buffer
```

We now have the offset from the hostboot base address: `0x00040278`.

Hostboot runs at `0x8000000`, so the address of interest for this build is
`0x80040278`. (FWIW, this address appears to be fixed between a bunch of
hostboot builds that I've looked at).

On your BMC:

```
$ pdbg -p 0 getmem 0x8040278 500 > /tmp/hb-log
$ cat /tmp/hb-log
Booting Hostboot kernel...
CPU=Nimbus  PIR=0
Valid BlToHbData found at 0x80E2000
Version=900000007
lpc=6030000000000, xscom=603FC00000000
iv_lpc=6030000000000, iv_xscom=603FC00000000, iv_data=0x454a8
HRMOR = 8000000
Hostboot base image ends at 0x98000...
PageManager end of preserved area at 0XE7000
PageManager page table offset at 0X100000
729 pages.
Starting init!
Bringing up VFS...done.
Initializing modules.
        Initing module libtrace.so...done.
        Initing module liberrl.so...done.
        Initing module libdevicefw.so...done.
        Initing module libscom.so...done.
        Initing module libxscom.so...done.
        Initing module libinitservice.so...done.
        Initing module libsecureboot_base.so...done.
        Initing module liblpc.so...done.
        Initing module libpnor.so...done.
        Initing module libvfs.so...done.
        Initing module libsio.so...done.
Modules initialized.
init_main: Starting Initialization Service...
InitService entry.
Addr [0], magic[50415254] checksum[0]
forced to true - l_isActiveTOC=1
SECUREBOOT::enabled() state:0
ExtInitSvc entry.
ast2400: SUART config in process
UART: Device initialized.
IStepDispatcher entry.
Set MchChk Xstop: 0x800603fc05012000=0000000100000000
Winkle threads - 0123.... - Awake!
```
