---
layout: post
title: Running Arm binaries on x86 with binfmt-misc and qemu
---

Here's how to set up Debian (and probably Ubuntu) to run a armhf binary.

## Install and configure
```
sudo dpkg --add-architecture armhf
sudo apt update
sudo apt install libc6:armhf qemu-user-static
sudo ln -s / /etc/qemu-binfmt/arm
```

The symlink should point to the location of the elf interpreter and is passed
as the -L argument to qemu.  The binfmt-misc setup uses `/etc/qemu-binfmt/%M`
where `%M` is the machine name.

## Test it out

Test it out by building a C program and running it.

```
$ echo -e '#include <stdio.h>\nint main() {printf("Hello, World!\\n");}' > hello.c
$ arm-linux-gnueabihf-gcc  -o hello{,.c}
$ file hello
hello: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, BuildID[sha1]=5dd9005818049bb8b57d7b38bfc05568945638f6, for GNU/Linux 3.2.0, not stripped
$ ./hello
Hello, World!
```

## Other libraries

Install libraries as you would for native packages. Here's an example of using openssl:

```
sudo apt install libssl-dev:armhf
```
```
#include <openssl/rand.h>

int main()
{
        unsigned char buffer[8];
        RAND_bytes(buffer, sizeof(buffer));

        for (int i = 0; i < sizeof(buffer); i++)
                printf("%02x ", buffer[i]);
        printf("\n");
}
```
```
$ gcc -o random{,.c}  -lcrypto
$ ./random
b3 ba b9 34 fe 20 99 69
```
```
$ arm-linux-gnueabihf-gcc -o random{,.c}  -lcrypto
$ ./random
e4 6b 2f 47 47 80 fa 9d
```

## Notes

### Third party repos

Third party repositories are unlikely to have metadata for the foreign
architecture, resulting this warning when running `apt update`:

```
N: Skipping acquire of configured file 'main/binary-armhf/Packages' as repository 'https://packages.riot.im/debian default InRelease' doesn't support architecture 'armhf'
```

We can tell apt a repository only supports a given architecture by adding
`arch=amd64` (or whatever your native arch is) to the
`/etc/apt/sources.list.d/` config:

```
deb [signed-by=/usr/share/keyrings/riot-im-archive-keyring.gpg arch=amd64] https://packages.riot.im/debian/ default main
```

As space can be used if there's already an option there.

### Ports

If you're on Ubuntu and trying to use an architecture that is hosted on the
ports mirror instead of the main mirror, such as ppc64el, you will need an
extra step before running `apt update`:

```
sudo sed -i -e 's/deb http/deb [arch=amd64] http/g' /etc/apt/sources.list
sudo dd of=/etc/apt/sources.list.d/ppc64el.list <<EOF
deb [arch=ppc64el] http://ports.ubuntu.com/ $release main universe restricted"
deb [arch=ppc64el] http://ports.ubuntu.com/ $release-updates main universe restricted"
EOF
```

This marks the existing repos as amd64 only, and adds the ports mirror for
ppc64el only. This is not required on Debian, as ppc64el is hosted in the main
mirror.

Thanks to Jeremy for the sed and dd oneliners. They're used in the [libnvme CI](https://github.com/linux-nvme/libnvme/commit/eaa989057f296a9107d4a40f24565af068897214).
