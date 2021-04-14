---
layout: post
title: OpenBMC console
---

The OpenBMC virtual serial console replaces IPMI Serial over LAN (SoL)
as a way to see the UART output of your host system. Where you would have once
run:

```
ipmitool -I lanplus -H server-bmc -U root sol activate
```

You can now connect over ssh to port 2200:

```
ssh -p2200 root@server-bmc
```

## Slow output

Some users have reported that the console output cannot keep up, especially
when dumping many thousands of lines of log messages. There was some work done
by Jeremy on the kernel side in 2018 to implement throttling of the LPC virtual
UART:

 * [serial/aspeed-vuart: Implement quick throttle mechanism](https://git.kernel.org/torvalds/c/5909c0bf9c7a)
 * [serial/aspeed-vuart: Implement rx throttling](https://git.kernel.org/torvalds/c/989983ea849d)
 * [serial/8250: export serial8250_read_char](https://git.kernel.org/torvalds/c/ebbaf9ab9ebd)
 * [serial: Introduce UPSTAT_SYNC_FIFO for synchronised FIFOs](https://git.kernel.org/torvalds/c/c5f78b1fe4e5)

If your BMC has a v4.18 based kernel, or dev-4.13 which contains the same
patches backported, then you have these fixes.

### Workaround

In addition to those fixes, there is a workaround that may improve throughput.
The configuration shipped on some machines defaults to mirroring the vuart UART
output to a real UART on the rear of the chassis. This means the ssh based
console is limited to the speed of the real UART.

To remove this feature, remove the following lines from `/etc/obmc-console.conf`:

```
local-tty = ttyS0
local-tty-baud = 115200
```

Restart the obmc-console service, or reboot your BMC.
