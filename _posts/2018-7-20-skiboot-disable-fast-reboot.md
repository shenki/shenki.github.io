---
layout: post
title: How to disable fast reboot
---

tl;dr: To disable OPAL/skiboot fast reboot on your system:

```
nvram -p ibm,skiboot --update-config fast-reset=0
```

Fast reboot is a way of re-booting a Power8 or Power9 system without going back
through a full "IPL". When you reboot a machine, after Linux has finished
shutting down, an OPAL call is made to power cycle the system. When power comes
back on the SBE starts fetching hostboot and running the hardware procedures
as in a normal cold boot.

When doing fast reboot, instead of telling the BMC or FSP to reboot, skiboot
keeps the lights on and re-runs part of it's initialisation sequence.

Importantly, this means the system does not need to wait for the service
processor, or re-run hostboot, which takes from one minute to many minutes,
depending on your system. The downside is some hardware and software
combinations misbehave when it does not go through the full reboot. If you have
this hardware, you can follow the procedure mentioned to disable fast reboot for
that machine.

