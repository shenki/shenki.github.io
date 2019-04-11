---
layout: post
title: Setting Petitboot kernel command line
---

The Petitboot kernel contains a default command line set in the skiroot [kernel configuration](https://github.com/open-power/op-build/blob/v2.2/openpower/configs/linux/skiroot_defconfig#L44):

```
CONFIG_CMDLINE="console=tty0 console=hvc0 ipr.fast_reboot=1 quiet"
```

You can add options to this by setting a nvram variable:

```
nvram -p ibm,skiboot --update-config bootargs="xmon=on"
```

This can also be used for blacklisting problematic kernel drivers, or enabling
extra debugging when chasing down a bootloader kernel issue.
