---
layout: post
title: Building OpenROAD tool on Debian/Ubuntu
---

[OpenROAD](https://github.com/The-OpenROAD-Project/OpenROAD/) is an open source
tool for building chips. This post describes how to build using system
dependencies on Debian.

### Background on building tools

Like most bits of software it contains instructions on how to install and build
the required dependencies in order to build the tool itself. They use a
combination of distro packages and hand built software installed to /usr/local.

I value software that has simple setup process. This means it can be installed
without placing packages on in the system paths (as to not interfere with
other bits of software I'm using), and avoids the other extreme of installing
it's own copy of the world using Conda. The happy place is using the system
libraries and toolchains.

As OpenROAD seeks to [support older distros](https://github.com/The-OpenROAD-Project/OpenROAD/blob/7c40272b26b9af338a3bd5cd58118a9d003b894e/etc/DependencyInstaller.sh#L270) - Centos7 and
Ubuntu 20.04 - this is not an option. But for those of us running something
recent such as Debian 10, we can get away with just the distro packages (as of
yesterday, when I [fixed a issue with lemon](https://github.com/The-OpenROAD-Project/OpenROAD/issues/1186#issuecomment-1103383986)).

### Installing dependencies

```
apt install liblemon-dev libspdlog-dev swig libboost-dev \
	libtcl flex bison clang g++ gcc git lcov python3-dev \
	libpcre3-dev libreadline-dev tcl-dev tcllib zlib1g-dev \
	qt5-image-formats-plugins qtbase5-dev
```

### Build and install

```
git clone --recurse-submodules https://github.com/The-OpenROAD-Project/OpenROAD
cd OpenROAD
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=$HOME/tools/openroad/
make -j`nproc`
```

### Usage

Ensure the binaries are in your PATH:
```
PATH=$PATH:$HOME/tools/openroad/bin/
```

Note that this requires Yosys in your path. Debian unstable has 0.15, which is
new enough to build the design:
```
apt install -t unstable yosys
```

Download and run the OpenROAD flow to build microwatt:
```
git clone https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts
cd OpenROAD-flow-scripts/flow
make DESIGN_CONFIG=designs/sky130hd/microwatt/config.mk
```
