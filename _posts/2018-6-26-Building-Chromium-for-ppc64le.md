Download gclient and do the initial fetc of source by following the official
instuctions. Stop at `Install additional build dependencies`:

https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md

Before running the `Install additional build dependencies` stop and apply this
patch. If you have already run it, run `sudo dpkg --remove-architecture i386`
to un-break your system:

```
--- a/build/install-build-deps.sh
+++ b/build/install-build-deps.sh
@@ -194,7 +194,7 @@ dev_list="\

 # 64-bit systems need a minimum set of 32-bit compat packages for the pre-built
 # NaCl binaries.
-if file -L /sbin/init | grep -q 'ELF 64-bit'; then
+if file -L /sbin/init | grep -q 'x86-64'; then
   dev_list="${dev_list} libc6-i386 lib32gcc1 lib32stdc++6"
 fi

@@ -483,7 +483,7 @@ fi
 # When cross building for arm/Android on 64-bit systems the host binaries
 # that are part of v8 need to be compiled with -m32 which means
 # that basic multilib support is needed.
-if file -L /sbin/init | grep -q 'ELF 64-bit'; then
+if file -L /sbin/init | grep -q 'x86-64'; then
   # gcc-multilib conflicts with the arm cross compiler (at least in trusty) but
   # g++-X.Y-multilib gives us the 32-bit support that we need. Find out the
   # appropriate value of X and Y by seeing what version the current
```

With the patch applied run the script with the following arguments:

```
build/install-build-deps.sh  --unsupported --no-syms --no-arm --no-chromeos-fonts --no-nacl --no-backwards-compatible
```

export GYP_DEFINES="disable_nacl=1":q
