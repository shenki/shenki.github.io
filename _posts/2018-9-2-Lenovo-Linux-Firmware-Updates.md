---
layout: post
title: Lenovo firmware updates under Linux
---



Phoronix published some [benchmarks of various pieces of software running on a
Power9 Talos II](https://www.phoronix.com/scan.php?page=article&item=power9-talos-2&num=5)
system. I wrote about my efforts to speed up LameMP3 earlier, and to continue
that trend I took a look at Scikit Learn.

[Scikit Learn](http://scikit-learn.org/) is a set of tools for machine
learning, written in Python. Some of the Python modules it uses wrap C or
Fortran libraries.

tl;dr: Installing the optimised PowerPC libblas libraries increases benchmark
performace by 1453%.

![Scikit Learn GaussianRandomProjection benchmark]({{ site.url }}/images/scikit-time.svg)

## Reproduce the results

The [Phoronix benchmark is on
OpenBenchmarking.org](https://openbenchmarking.org/test/pts/scikit-learn), and
we can see how the benchmark itself is run:

```
unzip -o scikit-learn-20160921.zip

echo "#!/bin/sh
cd scikit-learn-master/benchmarks/
python bench_random_projections.py
echo \$? > ~/test-exit-status" > scikit-learn
chmod +x scikit-learn
```

We can reproduce this. First we need to install the distro `python-sklearn`
package. Notice that some of the dependencies include `libblas3` and
`liblapack3`, which are Fortran libraries.  We can now run the benchmark:

```
$ wget https://raw.githubusercontent.com/scikit-learn/scikit-learn/master/benchmarks/bench_random_projections.py
$ time python bench_random_projections.py

Transformer                    |     fit      |  transform
-------------------------------|--------------|--------------
GaussianRandomProjection       |   4.2146s    |   60.4167s
SparseRandomProjection         |   0.1326s    |   0.6146s

real	5m27.319s
user	5m26.922s
sys	0m0.252s
```

The 5m26s, or 327 seconds, is a bit slower than the 230 seconds from Phoronix.
As mentioned in the LameMP3 analysis, this could be explained by the
pre-production hardware and slower clock speeds that are being used for this
test.

## Understanding the benchmark

The benchmark is actually a set of benchmarks, averaged over five runs. For the
rest of the post we will look at GaussianRandomProjection as this took the
majority of runtime, and set the number of iterations to one.

```
$ time python bench_random_projections.py --n-times 1 \
    --transformers=GaussianRandomProjection
Transformer                    |     fit      |  transform  
-------------------------------|--------------|--------------
GaussianRandomProjection       |   3.6705s    |   60.3699s  

real	1m4.227s
user	1m4.166s
sys	0m0.060s
```

We can use `perf` to understand where the execution time is going:

```
$ sudo perf record -o sklearn-baseline.data \
    python bench_random_projections.py --n-times 1 --transformers=GaussianRandomProjection
$ sudo chown $USER
$ perf report -q --stdio --percent-limit=1 -i sklearn-baseline.data

92.78%  python   libblas.so.3.7.1                          [.] dgemm_
 2.23%  python   libm-2.27.so                              [.] __log_finite
 1.98%  python   mtrand.powerpc64le-linux-gnu.so           [.] rk_gauss
 1.23%  python   mtrand.powerpc64le-linux-gnu.so           [.] rk_random
```

Here we can see that the function `dgemm_` from `libblas` is taking up most of
the CPU time. This is a matrix multiply function that operates on double
precession floating point numbers, which sounds like something a machine
learning application would do.

## libblas in detail

Looking dgemm looks a bit like this:

```
*
*           Form  C := alpha*A*B + beta*C.
*
              DO 90 j = 1,n
                  IF (beta.EQ.zero) THEN
                      DO 50 i = 1,m
                          c(i,j) = zero
   50                 CONTINUE
                  ELSE IF (beta.NE.one) THEN
                      DO 60 i = 1,m
                          c(i,j) = beta*c(i,j)
   60                 CONTINUE
                  END IF
                  DO 80 l = 1,k
                      temp = alpha*b(l,j)
                      DO 70 i = 1,m
                          c(i,j) = c(i,j) + temp*a(i,l)
   70                 CONTINUE
   80             CONTINUE
   90         CONTINUE
          ELSE
```

Mmmm Fortran. First year maths is coming back to haunt me.

So what we're benchmarking here is the Fortran library. The [Ubuntu package](https://launchpad.net/ubuntu/+source/lapack/3.7.1-4ubuntu1/+build/13787724)
[build logs](https://launchpadlibrarian.net/347394316/buildlog_ubuntu-bionic-ppc64el.lapack_3.7.1-4ubuntu1_BUILDING.txt.gz)
show that dgemm.f is built with `-O2`:

```
$ powerpc64le-linux-gnu-gfortran -fPIC -g -fdebug-prefix-map=/<<PKGBUILDDIR>>=. \
    -fstack-protector-strong -O2 -frecursive -c -o dgemm.o dgemm.f
```

Looking at debian/rules, ppc64le is special cased to disable O3:

```
ifeq ($(DEB_HOST_ARCH),ppc64el)
BUILDFLAGS_ENV += DEB_CFLAGS_MAINT_STRIP="-O3" DEB_CFLAGS_MAINT_APPEND="-O2"
BUILDFLAGS_ENV += DEB_FFLAGS_MAINT_STRIP="-O3" DEB_FFLAGS_MAINT_APPEND="-O2"
endif
```

This appears to be due to a [test case failure](https://bugs.launchpad.net/ubuntu/+source/lapack/+bug/1708735).

### Rebuilding with -Ofast

I modified `DEB_FFLAGS_MAINT_APPEND="-Ofast"`. This is like `-O3`, but with more
shortcuts. Depending on your application those shortcuts might not make sense

> Disregard strict standards compliance. -Ofast enables all -O3 optimizations.
> It also enables optimizations that are not valid for all standard-compliant
> programs. It turns on -ffast-math and the Fortran-specific -fstack-arrays,
> unless -fmax-stack-var-size is specified, and -fno-protect-parens.

```
$ debian/rules DEB_BUILD_OPTIONS=nocheck build
$ fakeroot debian/rules DEB_BUILD_OPTIONS=nocheck binary
```

Installing the deb that this produces and re-running the benchmark:

```
$ time python bench_random_projections.py --n-times 1 \
    --transformers=GaussianRandomProjection

Transformer                    |     fit      |  transform  
-------------------------------|--------------|--------------
GaussianRandomProjection       |   3.2686s    |   32.6166s  

real	0m36.098s
user	0m39.093s
sys	0m0.100s
```

This is an improvement of 178%.

## Other implementations

If we look at the dependencies for `python-numpy`

```
$ apt-cache show python-numpy | grep Depends
Depends: python (<< 2.8), python (>= 2.7~), python2.7:any, python:any (<< 2.8),
python:any (>= 2.7.5-5~), libblas3 | libblas.so.3, libc6 (>= 2.22), liblapack3
| liblapack.so.3
```

The package depends on `libblas3`, or anything that provides a shared object
called `libblas.so.3`.

```
$ apt-file search libblas.so.3
libatlas3-base: /usr/lib/powerpc64le-linux-gnu/atlas/libblas.so.3
libatlas3-base: /usr/lib/powerpc64le-linux-gnu/atlas/libblas.so.3.10.3
libblas3: /usr/lib/powerpc64le-linux-gnu/blas/libblas.so.3
libblas3: /usr/lib/powerpc64le-linux-gnu/blas/libblas.so.3.7.1
libopenblas-base: /usr/lib/powerpc64le-linux-gnu/openblas/libblas.so.3
```

This means we can use `libatlas3-base` or `libopenblas-base` as alternative
implementations of `libblas`. By installing it after `libblas`, Debian/Ubuntu's
alternatives system prefers this one over the `libblas` version.

If you're playing along at home you may already have mutliple `libblas.so.3`
options installed. To pick the one you're trying to test without resorting to
removing packages:
```
$ sudo update-alternatives --config libblas.so.3-powerpc64le-linux-gnu
There are 3 choices for the alternative libblas.so.3-powerpc64le-linux-gnu (providing /usr/lib/powerpc64le-linux-gnu/libblas.so.3).

  Selection    Path                                                  Priority   Status
------------------------------------------------------------
* 0            /usr/lib/powerpc64le-linux-gnu/openblas/libblas.so.3   40        auto mode
  1            /usr/lib/powerpc64le-linux-gnu/atlas/libblas.so.3      35        manual mode
  2            /usr/lib/powerpc64le-linux-gnu/blas/libblas.so.3       10        manual mode
  3            /usr/lib/powerpc64le-linux-gnu/openblas/libblas.so.3   40        manual mode

Press <enter> to keep the current choice[*], or type selection number:
```

### Using libatlas

```
$ sudo apt-get install libatlas3-base
...
update-alternatives: using /usr/lib/powerpc64le-linux-gnu/atlas/libblas.so.3
to provide /usr/lib/powerpc64le-linux-gnu/libblas.so.3 (libblas.so.3-powerpc64le-linux-gnu)
in auto mode
```

```
$ time python bench_random_projections.py --n-times 1 \
    --transformers=GaussianRandomProjection

Transformer                    |     fit      |  transform
-------------------------------|--------------|--------------
GaussianRandomProjection       |   3.2718s    |   4.8506s

real	0m8.291s
user	0m8.247s
sys	0m0.044s
```

64.2s down to 8.2 seconds! A 773% reduction in runtime, and this time no code
change was required.

```
$ sudo perf record -o sklearn-altas.data \
    python bench_random_projections.py --n-times 1 --transformers=GaussianRandomProjection
$ sudo chown $USER sklearn-altas.data
$ perf report -q --stdio --percent-limit=1 -i sklearn-libatlas.data
    51.81%  python   libblas.so.3.10.3                       [.] ATL_dJIK24x24x24TN24x24x0_a1_b1
    13.76%  python   libm-2.27.so                            [.] __log_finite
    12.73%  python   mtrand.powerpc64le-linux-gnu.so         [.] rk_gauss
     7.65%  python   mtrand.powerpc64le-linux-gnu.so         [.] rk_random
     4.79%  python   mtrand.powerpc64le-linux-gnu.so         [.] rk_double
     2.15%  python   libblas.so.3.10.3                       [.] ATL_dupNBmm0_4_0_b1
     1.61%  python   mtrand.powerpc64le-linux-gnu.so         [.] _init
     1.14%  python   libblas.so.3.10.3                       [.] ATL_dcol2blk_a1
```

We still have a `libblas` matrix function at the top, but it's consuming a
smaller proportion of the execution time.

### Using libopenblass

```
$ sudo apt-get install libopenblas-base
...
update-alternatives: using /usr/lib/powerpc64le-linux-gnu/openblas/libblas.so.3
to provide /usr/lib/powerpc64le-linux-gnu/libblas.so.3
(libblas.so.3-powerpc64le-linux-gnu) in auto mode
```

```
$ sudo perf record -o sklearn-openblas.data \
   python bench_random_projections.py --n-times 1 --transformers=GaussianRandomProjection
$ sudo chwon $USER sklearn-openblas.data
$ perf report -q --stdio --percent-limit=1 -i sklearn-openblas.data 
    27.96%  python   libopenblas_power8p-r0.2.20.so          [.] dgemm_kernel
    15.63%  python   libopenblas_power8p-r0.2.20.so          [.] 0x00000000001609c0
    15.42%  python   libopenblas_power8p-r0.2.20.so          [.] 0x0000000000160be0
     5.73%  python   libpthread-2.27.so                      [.] __pthread_mutex_unlock
     5.17%  python   libopenblas_power8p-r0.2.20.so          [.] 0x00000000001609c8
     5.04%  python   libpthread-2.27.so                      [.] __pthread_mutex_lock
     4.75%  python   libopenblas_power8p-r0.2.20.so          [.] 0x0000000000160be8
     2.91%  python   libm-2.27.so                            [.] __log_finite
     2.53%  python   mtrand.powerpc64le-linux-gnu.so         [.] rk_gauss
     1.87%  python   libc-2.27.so                            [.] pthread_mutex_lock
     1.62%  python   mtrand.powerpc64le-linux-gnu.so         [.] rk_random
     1.54%  python   libopenblas_power8p-r0.2.20.so          [.] 0x000000000020fd44
     1.44%  python   libopenblas_power8p-r0.2.20.so          [.] 0x0000000000160ea0
     1.39%  python   libopenblas_power8p-r0.2.20.so          [.] 0x000000000020fd40
     1.13%  python   libopenblas_power8p-r0.2.20.so          [.] 0x00000000001609c4
```

Now we're talking! This is using a Power8 optimised version of the blas routines.

```
$ time python bench_random_projections.py --n-times 1 \
    --transformers=GaussianRandomProjection

Transformer                    |     fit      |  transform
-------------------------------|--------------|--------------
GaussianRandomProjection       |   3.5704s    |   0.6145s

real	0m4.421s
user	0m41.257s
sys	0m0.052s
```

With `libopenblas` the runtime is roughly halved again, compared to the
`libatlas` implementation.

## Results

There are packaging issues in Ubuntu and Debian that affect the potential
performance of the machine learning libraires. Fixing libblas to be built with
the correct optimisation settings would provide a 2x improvement. Having the
default dependency be the fastest library would provide a 14x improvement.
