---
layout: post
title: Lenovo firmware updates under Linux
---


```
$ fwupdmgr refresh
Fetching metadata https://cdn.fwupd.org/downloads/firmware.xml.gz
Downloading…           [***************************************]
Fetching signature https://cdn.fwupd.org/downloads/firmware.xml.gz.asc
```

```
$ fwupdmgr get-updates
ignoring 20FCS1B100 System Firmware [c66f3753f0a35b3874d65546d4b7b10896d44ce2] as not updatable
```

```
$ fwupdmgr get-devices
20FCS1B100 System Firmware
  DeviceId:             c66f3753f0a35b3874d65546d4b7b10896d44ce2
  Guid:                 81cba30b-c2d4-4e66-bcf3-69d81ba890b0
  Guid:                 230c8b18-8d9b-53ec-838b-6cfc0383493a
  Plugin:               uefi
  Flags:                internal|require-ac|supported|registered|needs-reboot
  Version:              0.1.37
  VersionLowest:        0.1.31
  Icon:                 computer
  Created:              2018-09-02
  UpdateError:          /usr/lib/fwupd/efi/fwupdx64.efi.signed cannot be found
```

```
$ sudo apt-get install -yy fwupd-amd64-signed
$ dpkg -L  fwupd-amd64-signed
/.
/usr
/usr/share
/usr/share/doc
/usr/share/doc/fwupd-amd64-signed
/usr/share/doc/fwupd-amd64-signed/README.Debian
/usr/share/doc/fwupd-amd64-signed/changelog.gz
/usr/share/doc/fwupd-amd64-signed/copyright
```
