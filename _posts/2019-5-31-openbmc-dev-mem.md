---
layout: post
title: Using /dev/mem on OpenBMC
---

OpenBMC's kernel configuration disables `/dev/mem` by default. In the early
days of the project some functionality was implemented using simple userspace
programs that read and wrote straight to peripherals, instead of kernel drivers.
Now that it has kernel drivers userspace access to memory is restricted.

Sometimes you need this access, and to cover that use case there's a kernel
option that can be set to re-enable `/dev/mem`. To enable this from a shell on
the bmc, first take note of the currently set command line:

```
# fw_printenv  bootargs
bootargs=console=ttyS4,115200n8
```
And then add the devmem parameter:
```
fw_setenv bootargs console=ttyS4,115200n8 mem.devmem=1
```

After rebooting the `/dev/mem` character device will be back. This will persist
until the command line is changed back to the default.
