---
layout: post
title: SBE Boot Progress
---

The OpenPower BMC kicks off the host booting by writing to a CFAM register over
FSI. From there, if your system is healthy enough, it will start outputting
characters over the LPC VUART. If it's not, then you get no output, and the
only indication that something went wrong is the host powering itself off.

To further debug, we read some information from the host processor using pdbg.

### FSI2PIB Status Register

First thing to check is the FSI2PIB status register at 0x1007:

```
# pdbg -a getcfam 0x1007
p0: 0x1007 = 0xa02c80f0
```

IBM does not publish the CFAM documentation, but we can use the [headers from Hostboot](https://github.com/open-power/hostboot/blob/393fbe94077cd31670e50a31f72c1390ff8691af/src/import/chips/p9/common/include/p9a_perv_scom_addresses_fld.H#L5658)
to decode the bits in this register:

```
P9A_PERV_FSI2PIB_STATUS_ANY_ERROR                       = 0;
P9A_PERV_FSI2PIB_STATUS_SYSTEM_CHECKSTOP                = 1;
P9A_PERV_FSI2PIB_STATUS_SPECIAL_ATTENTION               = 2;
P9A_PERV_FSI2PIB_STATUS_RECOVERABLE_ERROR               = 3;
P9A_PERV_FSI2PIB_STATUS_CHIPLET_INTERRUPT_FROM_HOST     = 4;
P9A_PERV_FSI2PIB_STATUS_PARITY_CHECK                    = 5;
```

The first nibble is 0xA (IBM bits 0 and 2 are set), which means the processor
has performed a Terminate Immediate (TI) and set the special attention bit.

### SBE Trace Buffer
To debug further we can read the SBE trace buffer.

```
# pdbg -p0 getmempba 0x8208000 64
[==================================================] 100%
0x0000000008208000: 10 30 24 24 24 31 41 42 43 44 45 32 35 24 12 d0
0x0000000008208010: 14 fb
```

(if 0x8208000 doesn't work, try 0xf0208000)

We can use the values in Hostboot's [bootloader trace
header](https://github.com/open-power/hostboot/blob/master/src/include/bootloader/bootloader_trace.H)
to decode the buffer.

We can see that this system has made it to 0x14, but stopped at 0xfb:

```
/** Bootloader main verifyContainer started */
    BTLDR_TRC_MAIN_VERIFY_START          = 0x14,
/** Bootloader main verifyContainer failed */
    BTLDR_TRC_MAIN_VERIFY_FAIL               = 0xFB,
```

In this case, the system has been provisioned with secure boot keys but the SBE
image it's running is not signed by those keys, so boot has halted.

#### Buffer address

Footnote on the address of the buffer: It is composed of HRMOR of the HBBL
-- P9 it was `0x0820_0000` or it might be `0xf020_0000`, as HRMOR was changed
later in the release for OPAL to get out of the linux kernel space. 

The other part of the address is a constant found at [src/bootloader/bl_start.S](https://github.com/open-power/hostboot/blob/393fbe94077cd31670e50a31f72c1390ff8691af/src/bootloader/bl_start.S#L39):
```
.set HBBL_DATA_ADDR_OFFSET, 0x00008000 ;// offset of external data)
```
