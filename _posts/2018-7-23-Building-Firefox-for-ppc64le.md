---
layout: post
title: Building Firefox for ppc64le
---

Firefox has bitrotted on PowerPC a little. I installed an Ubuntu 18.04 (and
then upgraded to the in-development 18.10) ppc64le on a Power9 workstation, and
attempted to run Firefox 61. It crashed (trimmed backtrace):

```
Program received signal SIGSEGV, Segmentation fault.
arena_t::SplitRun (this=0x7ffff7700000, aRun=0x18, aSize=65535, aLarge=false,
    aZero=false)
    at /home/joel/src/mozilla-central/memory/build/mozjemalloc.cpp:2336
2336	  old_ndirty = chunk->ndirty;
(gdb) bt
#0  0x0000000010005734 in arena_t::SplitRun
#1  0x000000001000672c in arena_t::AllocRun
#2  0x0000000010007c34 in arena_t::GetNonFullBinRun
#4  0x000000001000e3c4 in BaseAllocator::calloc
#5  0x000000001000e3c4 in Allocator<MozJemallocBase>::calloc
#6  0x0000000010009388 in Allocator<ReplaceMallocBase>::calloc
#7  0x0000000010009388 in calloc
#8  0x00007ffff5aa4620 in g_malloc0 () at /usr/lib/powerpc64le-linux-gnu/libglib-2.0.so.0
#9  0x00007ffff5aa49d0 in g_malloc0_n
#10 0x00007ffff5a75b2c in  ()
#11 0x00007ffff5a7726c in g_hash_table_insert
#12 0x00007ffff5abb178 in g_intern_static_string
```

## Building Firefox

Before reporting this to the mozilla project, I decided to reproduce with the
latest master.

[https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Build_Instructions/Simple_Firefox_build/Linux_and_MacOS_build_preparation](Firefox build instructions) are straight forward, but quickly blew up as the bootstrap.py script attempts to run an x86 binary to install rust:

```
Could not find a Rust compiler.
Will try to install Rust.
Downloading rustup-init... Ok
Running rustup-init...
Traceback (most recent call last):
  File "bootstrap.py", line 176, in <module>
    sys.exit(main(sys.argv))
  File "bootstrap.py", line 167, in main
    dasboot.bootstrap()
  File "/tmp/tmp7eR7Tt/mozboot/bootstrap.py", line 285, in bootstrap

  File "/tmp/tmp7eR7Tt/mozboot/base.py", line 636, in ensure_rust_modern
  File "/tmp/tmp7eR7Tt/mozboot/base.py", line 679, in install_rust
  File "/usr/lib/python2.7/subprocess.py", line 185, in check_call
    retcode = call(*popenargs, **kwargs)
  File "/usr/lib/python2.7/subprocess.py", line 172, in call
    return Popen(*popenargs, **kwargs).wait()
  File "/usr/lib/python2.7/subprocess.py", line 394, in __init__
    errread, errwrite)
  File "/usr/lib/python2.7/subprocess.py", line 1047, in _execute_child
    raise child_exception
OSError: [Errno 8] Exec format error
```

To avoid this, install Rust from the distro repository (`apt-get install rustc
cargo`) and `bootstrap.py` could continue.

## Assembly syntax

The next issue was relating to assembly syntax in the `xpcom` directory,
affecting `xptcinvoke_asm_ppc64_linux.S` and `xptcstubs_asm_ppc64_linux.S`. The
assembly there uses this syntax:

```
    std     r29,-24(r1)
    std     r30,-16(r1)
    std     r31,-8(r1)
```

This causes warnings when building with clang (and GCC I think):

```
 error: invalid memory operand
        ld 31,-8(r1)
                 ^
```

To resolve this, the syntax can be changed to the following:

```
    std     r29,-24(%r1)
    std     r30,-16(%r1)
    std     r31,-8(%r1)
```

An alternative could be to pass `-mregnames` to the flags used by the assembler.

## Cache size detection

There's one other assembly related issue in
`security/nss/lib/freebl/mpi/mpcpucache.c`, where `s_mpi_getProcessorLineSize`
attempts to guess the size of the L1 data cache. This is mid-2000 era code that
is no longer the best way to determine this information, so instead of fixing the
assembly to compile, we could rework the code.

The Power9 processor I'm using has a 128 byte cache line. AFAIK this is the
same for Power8, and the G5, which are the most likely ppc64 platforms that
someone might be running Firefox on. So for now I've changed this function to
return 128 on ppc64 platforms.


## Page Size

I've also noticed that the distros carry a pach to jemalloc that indicate the platform has a static page size:

```
diff --git a/memory/build/mozjemalloc.cpp b/memory/build/mozjemalloc.cpp
--- a/memory/build/mozjemalloc.cpp
+++ b/memory/build/mozjemalloc.cpp
@@ -181,7 +181,7 @@ using namespace mozilla;
 // Debug builds are opted out too, for test coverage.
 #ifndef MOZ_DEBUG
 #if !defined(__ia64__) && !defined(__sparc__) && !defined(__mips__) &&         \
-  !defined(__aarch64__)
+  !defined(__aarch64__) && !defined(__powerpc__)
 #define MALLOC_STATIC_PAGESIZE 1
 #endif
 #endif
```

This is true for the distros shipping for power.

## Bug report

After fixing these issues, the crash was still present, so I opened this bug:

 [https://bugzilla.mozilla.org/show_bug.cgi?id=1477176]
