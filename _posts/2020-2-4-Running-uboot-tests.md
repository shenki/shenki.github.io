---
layout: post
title: Running u-boot's test suite
---

u-boot has a test suite that can run against Qemu, but it's intimidating to
get going. Here's some minimal instructions:

## Get the code

```
git clone git://git.denx.de/u-boot.git
https://github.com/swarren/uboot-test-hooks
```

### Install the python dependencies

You can choose which of these two work for you. I prefer installing the
packages from Debian's package manager (bullseye/testing works for me).

```
sudo apt install python3-atomicwrites python3-attr python3-coverage
python3-extras python3-fixtures python3-importlib-metadata python3-linecache2
python3-more-itertools python3-packaging python3-pbr python3-pluggy
python3-pyparsing python3-pytest python3-mimeparse python3-subunit python3-six
python3-testtools python3-traceback2 python3-unittest2 python3-wcwidth
python3-zipp
```

```
virtualenv -p /usr/bin/python3 /tmp/venv
. /tmp/venv/bin/activate
cd u-boot
pip install -r test/py/requirements.txt
```

## Get a ARM compiler

The easiest is to use one from your distro. Other options are to use buildman
to download one, or get one from https://toolchains.bootlin.com/

```
apt install gcc-arm-linux-gnueabihf
```

## Set up paths

I cloned the repos into to $SRC. If you set SRC to where you put them, you can
cut/paste the lines below.

```
PYTHONPATH=$SRC/uboot-test-hooks/py/travis-ci
PATH=$PATH:$SRC/uboot-test-hooks/bin/
```

uboot-test-tools uses your hostname to determine which scripts to use. Lets pretend we're running on travis, to debug CI failures:

```
ln -s $SRC/uboot-test-hooks/bin/travis-ci $SRC/uboot-test-hooks/bin/`hostname`
```

Another option would be to copy the `travis-ci` directory, so you can configure
the scripts to be appropriate for your setup.

## Go!

As we're passing `--build`, this will build u-boot in a directory called `build`
the current working directory.

```
CROSS_COMPILE="ccache arm-linux-gnueabi-" ./test/py/test.py --bd evb-ast2500 --id qemu --build-dir build --build
```

The board chosen is the ast2500-evb, which has upstream support in u-boot,
uboot-test-tools and in Qemu as of 4.2.


## Advanced

If you are testing qemu changes against u-boot, you can set the qemu binary as follows:

```
cd $SRC/uboot-test-hooks
sed -i 's|qemu-system-arm|$HOME/qemu/arm-softmmu/$1/' bin/travis-ci/conf.evb-ast2400_qemu
```

This is where creating your own set of scripts instead of relying on the
travis-ci configuration would make sense.
