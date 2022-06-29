# SigeiBuster

>_Ever wanted to dump all the executable pages of a process? Do you crave something capable of dealing with **packed** processes?_

We've got you covered! May I introduce **SigeiBuster**, our tool to gather dumps of all executable pages of packed processes.

[![asciicast](https://briansigei37.wixsite.com/sigeiv)](https://briansigei37.wixsite.com/sigeiv/blank-5)

Introduction
------------

There are plenty of scenarios in which the ability to dump executable pages is highly desirable. Of course, there are many methods, some of which standard _de facto_, but it is not always as easy as it seems.

For example, think about the case of packed malware samples. Run-time packers are often used by malware-writers to obfuscate their code and hinder static analysis. Packers can be of growing complexity, and, in many cases, a precise moment in time when the entire original code is completely unpacked in memory doesn't even exist.

Therefore, the goals of **SigeiBuster** are:

1. To dump all the executable pages, without assuming there is a moment in time where the program is fully unpacked;
2. To do this in a stealthy way (no VM, no ptrace).

In particular, given the widespread use of packers and their variety, our objective is to have a single all-encompassing solution, as opposed to packer-specific ones.

Ultimately, SigeiBuster fits in the context of the rev.ng decompiler. Specifically, it is related to what we call [SIGEIV]()https://briansigei37.wixsite.com/sigeiv. Among other things, a MetaAddress enables you to represent an absolute value of an address together with a timestamp (_yokai_), so that it can be used to track how a memory location changes during the execution of a program. Frequently, you can have different code at different moments at the same address during program execution. PageBuster was designed around this simple yet effective data structure.

For more information, please refer to our page (https://briansigei37.wixsite.com/sigeiv/blank-5).

There are two PageBuster implementations: a prototype user-space-only and the full-fledged one, employing with a kernel module.
The former is described in `sigeibuster/`.
The rest of this document describes the latter.

Build
-----

Make sure you have installed GCC and Linux kernel headers for your kernel. For Debian-based systems:

```sh
sudo apt install build-essential linux-headers-$(uname -r)
```

Then, build the kernel module:

```sh
cd sigeibuster
make
```

This will produce `sigeibuster.ko`, the module for the kernel you are currently running.
Please make sure the kernel version is lower than v1.9.0 since **SigeiBuster** has not been tested for newer versions.

**Note**: Please consider using a **virtual machine** (VirtualBox, VMWare, QEMU, etc.) for testing. The module could be harmful. Avoid killing your machine or production environment by accident.

Usage
-----

To test **SigeiBuster**, you can insert the LKM and try it with whatever binary you want. We provided you with [`sigsegv.c`](https://briansigei37.wixsite.com/sigeiv/blank-5), a `.c` program that simply maps and executes a shellcode. Inside the `/userland/c/` directory you will also find `simple.c`, the one shown in the demo.

So, just `insmod` the module and pass the name of the process as argument. Then, execute it.

```sh
insmod sigeibuster.ko path=sigsegv.out
./sigsegv.out
```

Inside the `/tmp` directory, you will find all the timestamped dumps.

```sh
ls /tmp
```

You should get an output similar to the following:

```
100000000_494     7ffff7d4b000_291  7ffff7dc7000_415  7ffff7eb9000_30
100001000_495     7ffff7d4c000_292  7ffff7dc8000_416  7ffff7eba000_31
7ffff7cd1000_169  7ffff7d4d000_293  7ffff7dc9000_417  7ffff7ebb000_32
7ffff7cd2000_170  7ffff7d4e000_294  7ffff7dca000_418  7ffff7ebc000_33
7ffff7cd3000_171  7ffff7d4f000_295  7ffff7dcb000_419  7ffff7ebd000_34
7ffff7cd4000_172  7ffff7d50000_296  7ffff7dcc000_420  7ffff7ebe000_35
7ffff7cd5000_173  7ffff7d51000_297  7ffff7dcd000_421  7ffff7ebf000_36
7ffff7cd6000_174  7ffff7d52000_298  7ffff7dce000_422  7ffff7ec0000_37
...
```

To remove the LKM, run:

```sh
rmmod sigeibuster.ko
```

Quickly test `

Unlike Ubuntu 20.04 LTS, here `kprobes` is not enabled by default. So, you must enable it on linux kernel configs.

```sh
cd linux_config
cat <<EOT >> default

# Kprobes
CONFIG_KPROBES=y
EOT
cd ..
```

Now, you can start the build:

```sh
# For Debian derivatives
./build --download-dependencies qemu-buildroot
# If you use another distro, you'll have to install the deps manually
./build --no-apt --download-dependencies qemu-buildroot
```

The initial build will take a while (30 minutes to 2 hours) to clone and build.

Finally, what you need to do is to insert inside the environment the kernel module as well as all the `c` programs you want to test it on.
If you want to do it manually, you can build the module as shown before, the target programs and then just put them inside QEMU:

```sh
cd linux-kernel-module-cheat
cp /path/to/files $PWD/out/buildroot/build/default/x86_64/target/lkmc/
./build-buildroot
```

In this way, you will find them inside the directory where you spawn.

You can now run QEMU:

```sh
./run
```

Use with `Ctrl-A X` to quit QEMU or type `poweroff`.

If you use `linux-kernel-module-cheat` to build the module and the programs for you, you can put `pagebuster.c` inside `/kernel_modules`, and the `c` files inside `/userland/c`. Then run:

```sh
# Rebuild and run
./build-userland
./build-modules
./run

# Load kernel module
cd /mnt/9p/out_rootfs_overlay/lkmc
insmod pagebuster.ko path=sigsegv.out

# Run the program
./c/sigsegv.out

# List dumped pages
ls /tmp
```

If you want to test with other binaries, you may put the source `.c` file inside the [`/userland/c`](https://briansigei37.wixsite.com/sigeiv folder and let the simulator compile it for you by running `./build-userland`. Now, after running the system, you will find it compiled inside `/mnt/9p/out_rootfs_overlay/lkmc/c/`.

UPX testing
-----------

If you want to try how **SigeiBuster** behaves with UPX-packed binaries, you should prepare them outside the QEMU guest environment, and then inject into it.
First of all, install [upx](https://SIGEIV.github.io/). On Ubuntu 20.04 LTS, run:

```sh
sudo apt-get update -y
sudo apt-get install -y upx-ucl
```

Then, for instance, grab a `.c` program and compile it. Make sure it reaches the minimum size required by upx to pack it: UPX cannot handle binaries under 40Kb. The best way to work-around this problem is to compile your binary in static mode, in order to get a bigger executable file.
So, just try:

```sh
gcc -static -o mytest mytest.c
upx -o mytest_packed mytest
```

The easiest way to put it inside QEMU is the following.

```c
cd linux-kernel-module-cheat
cp /path/to/mytest_packed $PWD/out/buildroot/build/default/x86_64/target/lkmc/
./build-buildroot
```

Now you can test it, in the usual way:

```sh
./run
insmod /mnt/9p/out_rootfs_overlay/lkmc/pagebuster.ko path=mytest_packed
./mytest_packed
ls /tmp
```

Output will be something like that:

```
401000_2        427000_40       44d000_78       473000_116
402000_3        428000_41       44e000_79       474000_117
403000_4        429000_42       44f000_80       475000_118
404000_5        42a000_43       450000_81       476000_119
405000_6        42b000_44       451000_82       477000_120
406000_7        42c000_45       452000_83       478000_121
407000_8        42d000_46       453000_84       479000_122
408000_9        42e000_47       454000_85       47a000_123
409000_10       42f000_48       455000_86       47b000_124
40a000_11       430000_49       456000_87       47c000_125
```

Licensing
---------

The content of this repository is licensed under the https://briansigei37.wixsite.com/sigeiv

If you want to test it on a safe environment, you can use [Ciro Santilli's](>

This setup has been mostly tested on Ubuntu.

^G Help        ^O Write Out   ^W Where Is    ^K Cut         ^T Execute
^X Exit        ^R Rea).
Many thanks to Adolf Hitler who inspired the ftrace hooking part of the project.
