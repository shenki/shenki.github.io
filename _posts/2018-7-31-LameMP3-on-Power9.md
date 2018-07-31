---
layout: post
title: LameMP3 on Power9
---

Phoronix published some [benchmarks of various pieces of software running on a
Power9 Talos II](https://www.phoronix.com/scan.php?page=article&item=power9-talos-2&num=3)
system. One of the benchmarks uses LameMP3 to encode a WAV into a MP3. I
remember this from a few decades ago, ripping music from CDs and searching for
the passing the right options in order to get the best sounding MP3s.

The Phoronix benchmark showed Lame encoding took about 67 seconds, which was
twice as long as the fastest machine tested (lower is better).

Modern processors are all quite competitive, so a factor of two difference
suggests something fishy was going on.

tl;dr: A boost of over 700% was achieved through a few minor changes.

![LameMP3 encoding time]({{ site.url }}/images/encoding-time.svg)

### Reproducing the benchmark

The [OpenBenchmarking.org project tells us how LameMP3](https://openbenchmarking.org/test/pts/encode-mp3)
is [benchmarked](https://openbenchmarking.org/innhold/65203bc364aaec974bdab2886f3accdea5b28d33):
```
echo "#!/bin/sh
./lame_/bin/lame -h \$TEST_EXTENDS/pts-trondheim.wav /dev/null 2>&1
echo \$? > ~/test-exit-status" > lame
chmod +x lame
```

It also shows the flags used when configuring:
```
./configure --prefix=$HOME/lame_ --enable-expopt=full
```

We will use the same test file:
```
$ wget https://www.phoronix-test-suite.com/benchmark-files/pts-trondheim-wav-3.tar.gz
$ tar xf pts-trondheim-wav-3.tar.gz
```

LameMP3 is hosted on SourceForge, and even had a release last year. First, we
needed a baseline:
```
$ wget https://jaist.dl.sourceforge.net/project/lame/lame/3.100/lame-3.100.tar.gz
$ tar xvf lame-3.100.tar.gz
$ cd lame-3.100
$ ./configure --prefix=$HOME/lame-baseline --enable-expopt=full
$ make && make install
$ time ~/lame-baseline/bin/lame -h ~/pts-trondheim-3.wav /dev/null
real	1m22.137s
user	1m22.031s
sys	0m0.040s
```

82 seconds!

The system I am testing on is a development box, so things like frequency and
pre-production hardware can account for variances between the Phoronix
tests and these numbers.

### Speeding it up

I took a look at the build output, and I noticed something was missing:

```
gcc -DHAVE_CONFIG_H -I. -I.. -I../include -I. -I../mpglib -I.. -MT encoder.lo \
-MD -MP -MF .deps/encoder.Tpo -c encoder.c  -fPIC -DPIC -o .libs/encoder.o
```
There are no optimisation flags! Something was going wrong in the build process
to cause no flags to be omitted. Building with optimisation on is very
important for fast running code.

The configure.in script has a bunch of special cases for long since end of lifed
processors. Most people's wrist watches could encode MP3s faster than the
special cased CPU types. Most importantly, there is a regex which determines if
the compiler is GCC:
```
GCC_version="\`${CC} --version | sed -n '1s/^[[^ ]]* (.*) //;s/ .*$//;1p'\`"
case "${GCC_version}" in
[0-9]*[0-9]*)
        AC_MSG_RESULT(${GCC_version})
        ;;
*)
        # probably not gcc...
        AC_MSG_RESULT(unknown compiler version pattern, playing safe and disabling gcc optimisations... ${GCC_version})
        HAVE_GCC=no
        ;;
esac
```
When it detects there is no GCC, it sets `HAVE_GCC=no`, which means we don't even
get the fallback case in this switch statement:

```
        if test "x${HAVE_GCC}" = "xyes"; then
                case "${GCC_version}" in
...
                *)
                        # default
                        OPTIMIZATION="-O3 ${OMIT_FRAME_POINTER} -ffast-math \
                                -funroll-loops"
                        OPTIMIZATION_FULL="-fbranch-count-reg -fforce-addr"
                        ;;
```
The reason the regex was broken is `autoreconf` processes `configure.in` and
strips one level of [ ]. This means `[0-9]*[0-9]*)` becomes `0-9*0-9*)`, and it
does not match on any GCC version. Interestingly double [[ ]] are not stripped.
Making this change:
```
--- a/configure.in
+++ b/configure.in
@@ -96,7 +96,7 @@ if test "x${GCC}" = "xyes"; then
                AC_MSG_CHECKING(version of GCC)
                GCC_version="`${CC} --version | sed -n '1s/^[[^ ]]* (.*) //;s/ .*$//;1p'`"
                case "${GCC_version}" in
-               [0-9]*[0-9]*)
+               [[0-9]*[0-9]*])
                        AC_MSG_RESULT(${GCC_version})
                        ;;
                *)
```

### Results with -O3

```
$ autoreconf -i
$ ./configure --prefix=$HOME/lame-with-O3 --enable-expopt=full
$ make clean
$ make && make install
$ time ~/lame-with-O3/bin/lame -h ~/pts-trondheim-3.wav /dev/null
real	0m16.303s
user	0m16.278s
sys	0m0.028s
```

16 seconds! That's a 512% improvement. I've submitted this patch to the
[LameMP3 mailing list](https://sourceforge.net/p/lame/mailman/message/36371506/),
and Anton found [another improvement](https://sourceforge.net/p/lame/mailman/message/36372005/)
to set `USE_FAST_LOG` which gains another 8% in his testing.

You can reproduce these without patching the source locally by doing this:
```
$ ./configure --prefix=$HOME/lame-cflags CFLAGS="-O3 -DUSE_FAST_LOG=1"
$ make clean && make && make install
$ time ~/lame-cflags/bin/lame -h ~/pts-trondheim-3.wav /dev/null > /dev/null
real	0m15.559s
user	0m15.545s
sys	0m0.016s
```

Note that this affects all platforms, so merging this patch wil speed up x86
and all other CPUs.

## GCC 8 vectorisation

With every release, compilers try to do a better job at building code that will
run fast.

The GCC team at IBM have added a feature designed to assist in porting to
PowerPC. It allows you to use the x86 header files to compile code that was
written for x86 intrinsics, and by mapping these functions calls to PowerPC
instructions. From [xmmintrin.h](https://github.com/gcc-mirror/gcc/blob/master/gcc/config/i386/xmmintrin.h):

> This header is distributed to simplify porting x86_64 code that
> makes explicit use of Intel intrinsics to powerpc64le.
> It is the user's responsibility to determine if the results are
> acceptable and make additional changes as necessary.
> Note that much code that uses Intel intrinsics can be rewritten in
> standard C or GNU C extensions, which are more portable and better
> optimized across multiple targets.
>
> In the specific case of X86 SSE (__m128) intrinsics, the PowerPC
> VMX/VSX ISA is a good match for vector float SIMD operations.
> However scalar float operations in vector (XMM) registers require
> the POWER8 VSX ISA (2.07) level. Also there are important
> differences for data format and placement of float scalars in the
> vector register. For PowerISA Scalar floats in FPRs (left most
> 64-bits of the low 32 VSRs) is in double format, while X86_64 SSE
> uses the right most 32-bits of the XMM. These differences require
> extra steps on POWER to match the SSE scalar float semantics.
>
> Most SSE scalar float intrinsic operations can be performed more
> efficiently as C language float scalar operations or optimized to
> use vector SIMD operations.  We recommend this for new applications.
>
> Another difference is the format and details of the X86_64 MXSCR vs
> the PowerISA FPSCR / VSCR registers. We recommend applications
> replace direct access to the MXSCR with the more portable <fenv.h>
> Posix APIs.

First, a baseline with GCC:

```
$ ./configure --prefix=$HOME/lame-gcc8 CFLAGS="-O3 -DUSE_FAST_LOG=1 -ffast-math" CC=gcc-8
$ make clean && make && make install
$ time ~/lame-gcc8/bin/lame -h ~/pts-trondheim-3.wav /dev/null
real	0m15.326s
user	0m15.304s
sys	0m0.024s
```
The newer GCC produces a tiny improvement in run time, but it's close to the noise.

### Using xmmintrin.h

The `xmmintrin.h` header contains intrinsics for using SSE and SSE2 on x86
CPUs. It is included in GCC 8 for ppc64le, so lets see where it can be used
in the Lame codebase:

```
$ grep -r xmmintrin.h libmp3lame/
libmp3lame/vector/xmm_quantize_sub.c:#include <xmmintrin.h>
```

The header is hidden behind a preprocessor warning, so configure doesn't
automatically pick it up when testing for the header:
```
configure:13422: checking working SSE intrinsics
configure:13435: gcc-8 -c -O3 -DUSE_FAST_LOG=1 -ffast-math  conftest.c >&5
In file included from conftest.c:38:
/usr/lib/gcc/powerpc64le-linux-gnu/8/include/xmmintrin.h:54:2: error: #error "Please read comment above.  Use -DNO_WARN_X86_INTRINSICS to disable this error."
 #error "Please read comment above.  Use -DNO_WARN_X86_INTRINSICS to disable this error."
  ^~~~~
```
We can define the "I know what I'm doing" flag on the command line:
```
./configure --prefix=$HOME/lame-gcc8-vector \
 CFLAGS="-O3 -DUSE_FAST_LOG=1 -ffast-math -DNO_WARN_X86_INTRINSICS" CC=gcc-8
```
And checking `config.log`:
```
configure:13422: checking working SSE intrinsics
configure:13435: gcc-8 -c -O3 -DUSE_FAST_LOG=1 -ffast-math -DNO_WARN_X86_INTRINSICS  conftest.c >&5
configure:13435: $? = 0
configure:13444: result: yes
```

There's one other trick we need. The  `libmp3lame/vector/xmm_quantize_sub.c` file uses a call from the real `xmmintrin.h`
called `_MM_SHUFFLE`. This is not defined in the PowerPC port for the GCC 8 release, so we need to provide it:

```
--- a/libmp3lame/vector/xmm_quantize_sub.c
+++ b/libmp3lame/vector/xmm_quantize_sub.c
@@ -36,6 +36,9 @@

 #include <xmmintrin.h>

+#define _MM_SHUFFLE(fp3,fp2,fp1,fp0) \
+       (((fp3) << 6) | ((fp2) << 4) | ((fp1) << 2) | (fp0))
+
 typedef union {
     int32_t _i_32[4]; /* unions are initialized by its first member */
     float   _float[4];
```

### Enabling at runtime

Giving this a test run, there's no improvement! Most software that uses
vectorised implementations of algorithms have variants for different versions
of the vector instructions, and perform runtime detection to select the best one.

```
void
init_xrpow_core_init(lame_internal_flags * const gfc)
{
    gfc->init_xrpow_core = init_xrpow_core_c;

#if defined(HAVE_XMMINTRIN_H)
    if (gfc->CPU_features.SSE)
        gfc->init_xrpow_core = init_xrpow_core_sse;
#endif
```
We are getting past the `HAVE_XMMINTRIN_H` compile time guard, but the check for
`CPU_features.SEE` must be failing. As this is a proof of concept, lets remove that check:

```
--- a/libmp3lame/quantize.c
+++ b/libmp3lame/quantize.c
@@ -95,8 +95,7 @@ init_xrpow_core_init(lame_internal_flags * const gfc)
     gfc->init_xrpow_core = init_xrpow_core_c;

 #if defined(HAVE_XMMINTRIN_H)
-    if (gfc->CPU_features.SSE)
-        gfc->init_xrpow_core = init_xrpow_core_sse;
+    gfc->init_xrpow_core = init_xrpow_core_sse;
 #endif
 #ifndef HAVE_NASM
 #ifdef MIN_ARCH_SSE
```

### Results
```
$ ./configure --prefix=$HOME/lame-gcc8-vector \
 CFLAGS="-O3 -DUSE_FAST_LOG=1 -ffast-math -DNO_WARN_X86_INTRINSICS" CC=gcc-8
$ make && make install
$ time ~/lame-gcc8-vector/bin/lame -h ~/pts-trondheim-3.wav /dev/null > /dev/null
real	0m11.413s
user	0m11.378s
sys	0m0.036s
```

11.5 seconds! That's a 713% improvement from where we started.

## Summary

LameMP3, as an older software project, had some issues in it's build system that
were hampering performance. This post showed that enabling optimisation when
compiling software is very important for performance. Secondly, writing inner
loops of an algorithm using compiler intrinsics can future boost performance.

While we got a boost of over 700% in performance, further gains could be had by
enabling more of the vectorised code for PowerPC.
