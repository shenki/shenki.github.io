---
layout: post
title: Building a Linux powernv kernel with clang
---

These instructions are current as of v4.19-rc2 running Ubuntu 18.10 with
clang-7 (7.0.0-+rc2-1~exp3). This will work on a ppc64le box or cross compiled
from another machine.

```
export ARCH=powerpc CROSS_COMPILE=powerpc64le-linux-gnu-
make powernv_defconfig
```

1. Disable `PPC_DISABLE_WERROR` due to `-Wduplicate-decl-specifier` warnings in
   `arch/powerpc/include/asm/uaccess.h` (see [https://github.com/linuxppc/linux/issues/185])

2. Disable `FTRACE` due to this enabling the `-mno-sched-epilog` option which
   clang does not know about.

3. Disable `MD_RAID456` and `BTRFS_FS` as `lib/raid6/altivec1.c` causes clang to crash

4. Apply this patch ([https://patchwork.ozlabs.org/patch/966825/]):

```
diff --git a/arch/powerpc/Makefile b/arch/powerpc/Makefile
index 11a1acba164a..a70639482053 100644
--- a/arch/powerpc/Makefile
+++ b/arch/powerpc/Makefile
@@ -238,7 +238,7 @@ cpu-as-$(CONFIG_4xx)                += -Wa,-m405
 cpu-as-$(CONFIG_ALTIVEC)       += $(call as-option,-Wa$(comma)-maltivec)
 cpu-as-$(CONFIG_E200)          += -Wa,-me200
 cpu-as-$(CONFIG_E500)          += -Wa,-me500
-cpu-as-$(CONFIG_PPC_BOOK3S_64) += -Wa,-mpower4
+cpu-as-$(CONFIG_PPC_BOOK3S_64) += $(call as-option,-Wa$(comma)-mpower8,-Wa$(comma)-mpower4)
 cpu-as-$(CONFIG_PPC_E500MC)    += $(call as-option,-Wa$(comma)-me500mc)

 KBUILD_AFLAGS += $(cpu-as-y)
```

```
make CC=clang-7 CLANG_TRIPLE=powerpc64le-linux-gnu
```

And then the link step will fail with this:

```
ld: kernel/irq/manage.o: in function `local_bh_enable':
manage.c:(.text+0x3ccc): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+3cb8
ld: kernel/rcu/srcutree.o: in function `local_bh_enable':
srcutree.c:(.text+0x258c): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+2578
ld: kernel/time/hrtimer.o: in function `local_bh_enable':
hrtimer.c:(.text+0x242c): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+2418
ld: kernel/bpf/cpumap.o: in function `local_bh_enable':
cpumap.c:(.text+0x134c): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+1338
ld: drivers/net/ethernet/broadcom/tg3.o: in function `local_bh_enable':
tg3.c:(.text+0x87dc): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+87c8
ld: drivers/net/ethernet/intel/e1000/e1000_main.o: in function `local_bh_enable':
e1000_main.c:(.text+0xc13c): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+c128
ld: net/core/sock.o: in function `local_bh_enable':
sock.c:(.text+0x5a6c): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+5a58
ld: net/core/gen_estimator.o: in function `local_bh_enable':
gen_estimator.c:(.text+0x2ec): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+2d8
ld: net/core/dev.o: in function `local_bh_enable':
dev.c:(.text+0x8a6c): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+8a58
ld: net/core/neighbour.o: in function `local_bh_enable':
neighbour.c:(.text+0x1f2c): relocation truncated to fit: R_PPC64_ADDR16_HA against `.text'+1f18
ld: net/sched/sch_generic.o: in function `local_bh_enable':
sch_generic.c:(.text+0x392c): additional relocation overflows omitted from the output
```
