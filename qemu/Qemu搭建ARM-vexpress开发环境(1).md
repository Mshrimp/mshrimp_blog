# Qemu搭建ARM vexpress开发环境(1)



## 目录

[TOC]

嵌入式软件开发依赖于嵌入式硬件设备，比如：开发板、外部模块设备等，但是如果只是想学习、研究Linux内核，想学习Linux内核的架构，工作模式，需要修改一些代码，重新编译并烧写到开发板中进行验证，这样未必有些复杂，并且为此专门购买各种开发版，浪费资金，开会演示效果还需要携带一大串的板子和电线，不胜其烦。然而Qemu的使用可以避免频繁在开发板上烧写版本，如果进行的调试工作与外设无关，仅仅是内核方面的调试，Qemu模拟ARM开发环境完全可以完美地胜任。

<!--more-->



下面简单介绍下我的Qemu开发环境搭建过程



## 1. 环境

由于在开发过程中也需要Windows系统下的一些工具，双系统环境切换操作系统时必须重启，于是放弃了以前搭建的双系统环境，而采用在PC的Windows10系统下通过VirtualBox虚拟机安装Xubuntu系统进行开发，避免了双系统开发中需要不断重启切换PC系统的问题。Xubuntu系统和Ubuntu系统大同小异，只是桌面封装更加简洁。



### 1.1 所使用环境

> Ubuntu-18.04.1

或：

> PC系统：Windows10
> 虚拟机：VirtualBox-5.18
> 虚拟机系统：Xubuntu
> 模拟的开发板：vexpress



### 1.2 搭建环境时使用的工具

qemu-4.2.0
linux-4.14.172(Linux Kernel)
u-boot-2017.05
busybox-1.31.1
arm-linux-gnueabi-gcc (Linaro 7.5.0-3)



为了将Qemu搭建开发环境资料统一放到一起，创建～/qemu目录，所有相关文件全部放置在一起；



## 2. 安装交叉编译工具
```shell
# sudo apt install gcc-arm-linux-gnueabi
```

查看安装是否成功：

```shell
# arm-linux-gnueabi-gcc -v
Using built-in specs.
COLLECT_GCC=arm-linux-gnueabi-gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc-cross/arm-linux-gnueabi/7/lto-wrapper
Target: arm-linux-gnueabi
Configured with: ../src/configure -v --with-pkgversion='Ubuntu/Linaro 7.5.0-3ubuntu1~18.04' --with-bugurl=file:///usr/share/doc/gcc-7/README.Bugs --enable-languages=c,ada,c++,go,d,fortran,objc,obj-c++ --prefix=/usr --with-gcc-major-version-only --program-suffix=-7 --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --with-sysroot=/ --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-libitm --disable-libquadmath --disable-libquadmath-support --enable-plugin --with-system-zlib --with-target-system-zlib --enable-multiarch --enable-multilib --disable-sjlj-exceptions --with-arch=armv5t --with-float=soft --disable-werror --enable-multilib --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=arm-linux-gnueabi --program-prefix=arm-linux-gnueabi- --includedir=/usr/arm-linux-gnueabi/include
Thread model: posix
gcc version 7.5.0 (Ubuntu/Linaro 7.5.0-3ubuntu1~18.04) 
```



## 3. 安装Qemu工具

有两种方法可以在Linux环境下安装Qemu工具，第一种直接使用XUbuntu系统的apt工具安装，但是这种方法安装的Qemu系统版本不是最新的，如果需要安装最新版本的Qemu工具，就需要第二种方法，通过Git工具下载源码，切换到最新分支再去编译安装了；但是一般情况下通过git下载代码速度极慢，我们可以使用第三种方法，找到要下载的qemu版本，使用迅雷下载；具体操作如下所述：



### 3.1 快速安装Qemu
```shell
# sudo apt install qemu
```

这种情况下安装的qemu版本可能不是最新版本；如果想要安装最新版本的qemu，还得使用下边介绍的使用源代码编译安装的方法；



### 3.2 下载Qemu源码编译安装



#### 3.2.1 下载Qemu源码

1. 从Git服务器下载Qemu代码，记着在下载之前选择并切换需要的源码分支：

```shell
# git clone git://git.qemu-project.org/qemu.git
# cd qemu
# git checkout -b stable-*** remotes/origin/stable-***
```
2. 在windows系统下使用迅雷下载

   登陆download.qemu.org网站，选择需要的版本，点击下载，或者右键后选使用迅雷下载，速度会更快：

   在这里选择qemu-4.2.0.tar.xz使用；



#### 3.2.2 安装

在配置qemu之前，需要安装一些依赖的库或者软件包：

```shell
# sudo apt-get install zlib1g-dev 
# sudo apt-get install libglib2.0-0
# sudo apt-get install libglib2.0-dev
# sudo apt-get install libtool
# sudo apt-get install libsdl1.2-dev
# sudo apt-get install autoconf
```

解压源代码：

```shell
# tar -xvf qemu-4.2.0.tar.xz
```

为了防止编译后文件比较乱，选择创建build目录作为编译中间目标路径：

```shell
# cd qemu-4.2.0/
# mkdir build
# cd build/
```



配置、编译并安装Qemu：

```shell
# ../configure --target-list=arm-softmmu --audio-drv-list=
# make
# make install
```



#### 3.2.3 在编译过程中可能出现的问题

```shell
# ../configure --target-list=arm-softmmu --audio-drv-list=
ERROR: pkg-config binary 'pkg-config' not found
```

缺少库文件，按照上一步中的步骤安装库文件；



```shell
# ../configure --target-list=arm-softmmu --audio-drv-list=
ERROR: pixman >= 0.21.8 not present.
       Please install the pixman devel package.
```

可以通过apt-cache查询缺少的依赖库：

```shell
# apt-cache search pixman
libpixman-1-0 - pixel-manipulation library for X and cairo
libpixman-1-dev - pixel-manipulation library for X and cairo (development files)
```

安装缺少的依赖库：

```shell
# sudo apt-get install libpixman-1-0
# sudo apt-get install libpixman-1-dev
```



### 3.3 查看Qemu版本

```shell
# qemu-system-arm --version
QEMU emulator version 4.2.0
Copyright (c) 2003-2019 Fabrice Bellard and the QEMU Project developers
```



### 3.4 查看Qemu支持的开发板

Qemu工具支持大量开发板的虚拟，现存的大部分常用开发板都能很好地支持。通过下面的命令操作可以看到当前版本的Qemu工具支持的开发板列表：
```shell
# qemu-system-arm -M help
......
vexpress-a15         ARM Versatile Express for Cortex-A15
vexpress-a9          ARM Versatile Express for Cortex-A9
......
```



### 3.5 运行Qemu

该操作目前还不能运行，因为还没有编译内核，如果手边有编译好的别的版本的zImage文件，可以通过下面命令尝试运行看下效果。
```shell
# qemu-system-arm -M vexpress-a9 -m 512M -kernel ./zImage -dtb ./vexpress-v2p-ca9.dtb -nographic -append "console=ttyAMA0"
    -M          指定开发板
    -m          指定内存大小
    -kernel     指定内核文件
    -dtb        指定dtb文件
    -nographic  指定不需要图形界面
    -append     指定扩展显示界面，串口或者LCD
```

实例参考：

```shell
# qemu-system-arm -M vexpress-a9 -m 512M -kernel ~/qemu/zImage -dtb ~/qemu/vexpress-v2p-ca9.dtb -nographic -append "console=ttyAMA0"
-M vexpress-a9	模拟vexpress-a9板，可以使用-M ?参数来查询qemu支持的所有单板
-m 512M	单板物理内存512M
-kernel	~/qemu/zImage	指定内和镜像及路径
-dtb ~/qemu/vexpress-v2p-ca9.dtb	指定单板的设备树文件
-nographic	不使用图形界面，使用串口
-append "console=ttyAMA0"	指定内核启动参数，串口设备使用ttyAMA0
```



## 4. 配置并编译Linux内核



### 4.1 下载Linux内核

通过众所周知的内核下载网站www.kernel.org下载需要的内核版本，这里我下载的是相对来说最新的长期支持的内核版本linux-4.4.157。



### 4.2 解压Linux内核

```shell
# tar -xvf linux-4.4.157.tar.xz
```



### 4.3 编译Linux内核



#### 4.3.1 配置

```shell
# make vexpress_defconfig ARCH=arm O=./object
make[1]: Entering directory '/home/xiami/tool/linux-4.14.172/object'
  HOSTCC  scripts/basic/fixdep
  GEN     ./Makefile
  HOSTCC  scripts/kconfig/conf.o
  SHIPPED scripts/kconfig/zconf.tab.c
  SHIPPED scripts/kconfig/zconf.lex.c
  HOSTCC  scripts/kconfig/zconf.tab.o
  HOSTLD  scripts/kconfig/conf
#
# configuration written to .config
#
make[1]: Leaving directory '/home/xiami/tool/linux-4.14.172/object'
```



```shell
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig -j4 O=./object
```



#### 4.3.2 编译

```shell
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j4 O=./object
```

或者在Makefile中配置默认值，指定ARCH和CROSS_COMPILE，免得每次编译都需要带参数；

```shell
# make vexpress_defconfig
# make zImage -j4
# make modules -j4    // 编译驱动模块
# make dtbs		// 编译设备树
```
得到编译文件：
```shell
arch/arm/boot/zImage
arch/arm/boot/dts/vexpress-v2p-ca9.dtb
```

分别将编译生成的zImage和vexpress-v2p-ca9.dtb文件放到~/qemu目录；

```shell
# cp arch/arm/boot/zImage ~/qemu
# cp arch/arm/boot/dts/vexpress-v2p-ca9.dtb ~/qemu
```



### 4.4 Qemu启动命令

```shell
# qemu-system-arm -M vexpress-a9 -m 512M -kernel kernel/linux-4.4.157/arch/arm/boot/zImage -dtb kernel/linux-4.4.157/arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "console=ttyAMA0"
```
Qemu的启动命令需要带好几个参数，完成启动命令比较长，每次都输入很可能会出现错误，为了使用方便，可以将该命令放到shell脚本中执行：
```shell
# cat boot.sh
#! /bin/sh
qemu-system-arm \
        -M vexpress-a9  \
        -m 512M \
        -kernel kernel/linux-4.4.157/arch/arm/boot/zImage   \   
        -dtb kernel/linux-4.4.157/arch/arm/boot/dts/vexpress-v2p-ca9.dtb    \   
        -nographic  \
        -append "console=ttyAMA0"
```

启动日志
内核成功启动，内核的启动打印信息非常多。启动最后出错是因为没有挂载根文件系统。

```
$ qemu-system-arm -M vexpress-a9 -m 512M -kernel ~/qemu/zImage -dtb ~/qemu/vexpress-v2p-ca9.dtb -nographic -append "console=ttyAMA0"
Booting Linux on physical CPU 0x0
Linux version 4.14.172 (xiami@xiami) (gcc version 7.5.0 (Ubuntu/Linaro 7.5.0-3ubuntu1~18.04)) #1 SMP Sun Mar 15 12:27:54 CST 2020
CPU: ARMv7 Processor [410fc090] revision 0 (ARMv7), cr=10c5387d
CPU: PIPT / VIPT nonaliasing data cache, VIPT nonaliasing instruction cache
OF: fdt: Machine model: V2P-CA9
Memory policy: Data cache writeback
CPU: All CPU(s) started in SVC mode.
percpu: Embedded 15 pages/cpu s32396 r8192 d20852 u61440
Built 1 zonelists, mobility grouping on.  Total pages: 130048
Kernel command line: console=ttyAMA0
log_buf_len individual max cpu contribution: 4096 bytes
log_buf_len total cpu_extra contributions: 12288 bytes
log_buf_len min size: 16384 bytes
log_buf_len: 32768 bytes
early log buf free: 15044(91%)
PID hash table entries: 2048 (order: 1, 8192 bytes)
Dentry cache hash table entries: 65536 (order: 6, 262144 bytes)
Inode-cache hash table entries: 32768 (order: 5, 131072 bytes)
Memory: 509580K/524288K available (6144K kernel code, 403K rwdata, 1372K rodata, 1024K init, 161K bss, 14708K reserved, 0K cma-reserved)
Virtual kernel memory layout:
    vector  : 0xffff0000 - 0xffff1000   (   4 kB)
    fixmap  : 0xffc00000 - 0xfff00000   (3072 kB)
    vmalloc : 0xa0800000 - 0xff800000   (1520 MB)
    lowmem  : 0x80000000 - 0xa0000000   ( 512 MB)
    modules : 0x7f000000 - 0x80000000   (  16 MB)
      .text : 0x80008000 - 0x80700000   (7136 kB)
      .init : 0x80900000 - 0x80a00000   (1024 kB)
      .data : 0x80a00000 - 0x80a64f90   ( 404 kB)
       .bss : 0x80a6bd00 - 0x80a94468   ( 162 kB)
SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=4, Nodes=1
Hierarchical RCU implementation.
	RCU event tracing is enabled.
	RCU restricting CPUs from NR_CPUS=8 to nr_cpu_ids=4.
RCU: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=4
......
input: ImExPS/2 Generic Explorer Mouse as /devices/platform/smb@4000000/smb@4000000:motherboard/smb@4000000:motherboard:iofpga@7,00000000/10007000.kmi/serio1/input/input2
VFS: Cannot open root device "(null)" or unknown-block(0,0): error -6
Please append a correct "root=" boot option; here are the available partitions:
1f00          131072 mtdblock0 
 (driver?)
1f01           32768 mtdblock1 
 (driver?)
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
CPU: 0 PID: 1 Comm: swapper/0 Not tainted 4.14.172 #1
Hardware name: ARM-Versatile Express
[<8010f180>] (unwind_backtrace) from [<8010b444>] (show_stack+0x10/0x14)
[<8010b444>] (show_stack) from [<806616cc>] (dump_stack+0x94/0xa8)
[<806616cc>] (dump_stack) from [<8011d67c>] (panic+0xdc/0x248)
[<8011d67c>] (panic) from [<8090129c>] (mount_block_root+0x1d4/0x2a8)
[<8090129c>] (mount_block_root) from [<80901490>] (mount_root+0x120/0x128)
[<80901490>] (mount_root) from [<809015e8>] (prepare_namespace+0x150/0x198)
[<809015e8>] (prepare_namespace) from [<80900ea0>] (kernel_init_freeable+0x244/0x254)
[<80900ea0>] (kernel_init_freeable) from [<80674538>] (kernel_init+0x8/0x114)
[<80674538>] (kernel_init) from [<801076a8>] (ret_from_fork+0x14/0x2c)
---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

最后的Kernel panic是因为没有挂载根文件系统，下一布制作一个简易文件系统。



## 5. 制作简易根文件系统

使用busybox制作简易的根文件系统



### 5.1 下载busybox工具

从https://busybox.net/downloads/下载最新的busybox。

同样，建议在windows系统下使用迅雷下载，先找到合适的busybox版本，再用右键，选择用迅雷下载，这样速度会很快；



### 5.2 解压busybox

```shell
# tar -xvf busybox-1.31.1.tar.bz2
```



### 5.3 配置并编译busybox



#### 5.3.1 配置

```shell
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig
```



编译选择使用glibc动态库，因为静态库可能会出现一些未知的问题

```shell
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig
Settings  --->
    Build Options  --->
        [ ] Build static binary (no shared libs)
```



默认的安装目录是./_install，如果需要指定安装目录，可以在下边修改：

```shell
Settings  --->
Installation Options ("make install" behavior)
(./_install) Destination path for 'make install'
```



#### 5.3.2 编译

```shell
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
```



#### 5.3.3 安装

```shell
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install
```

提示下边信息，表示安装成功：

```shell
--------------------------------------------------
You will probably need to make your busybox binary
setuid root to ensure all configured applets will
work properly.
--------------------------------------------------
```

安装完成之后，生成的目标文件默认在./_install目录，这个目标文件目录就是下边要制作根文件系统需要用到的工具：

```shell
$ ls _install/
bin  linuxrc  sbin  usr
```



或者直接使用CONFIG_PREFIX指定安装目录：

```shell
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- CONFIG_PREFIX=/.../rootfs/ install
```



#### 5.3.4 提前设置好编译默认值

编译安装过程中，一直输入ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-比较麻烦，可以在Makefile中设置好默认值；

修改Makefile:

```shell
# vim Makefile
ARCH = arm
CROSS_COMPILE = arm-linux-gnueabi-
```
之后的编译、安装命令就简单了：
```shell
# make
# make install
```



### 5.4 生成简易根文件系统

制作一个简易的根文件系统，该文件系统包含的功能极其简陋，仅为了验证Qemu启动Linux内核后挂载跟文件系统的过程。以后会根据具体需要进一步完善该文件系统。



#### 1) 编译并安装busybox

将busybox编译生成的_install目录下的文件全部拷贝到根文件系统目标rootfs/目录：
```shell
# mkdir rootfs
# cp /.../busybox-1.29.3/_install/* rootfs/ -rfd
```
也可以在指定busybox的安装目录直接安装：
```shell
# make CONFIG_PREFIX=/.../rootfs/ install
```



#### 2) 添加glibc库

在根文件系统中添加加载器和动态库：
```shell
# mkdir rootfs/lib
# cp /usr/arm-linux-gnueabi/lib/* rootfs/lib/ -rfp
```



#### 3) 静态创建设备文件

```shell
# mkdir rootfs/dev
# cd rootfs/dev
# mknod -m 666 tty1 c 4 1
# mknod -m 666 tty2 c 4 2
# mknod -m 666 tty3 c 4 3
# mknod -m 666 tty4 c 4 4
# mknod -m 666 console c 5 1
# mknod -m 666 null c 1 3
```
至此，简易版根文件系统就制作完成，该根文件系统只含有最基本的功能，一些其他功能在以后的操作中会进行添加，如有兴趣可以继续参考下一篇文章《》《》。



### 5.5 制作SD卡文件系统镜像



#### 1) 生成一个空的SD卡镜像：

```shell
# dd if=/dev/zero of=rootfs.ext3 bs=1M count=32
32+0 records in
32+0 records out
33554432 bytes (34 MB, 32 MiB) copied, 0.0236764 s, 1.4 GB/s
```



#### 2) 将SD卡格式化为exts文件系统：

```shell
# mkfs.ext3 rootfs.ext3
mke2fs 1.42.13 (17-May-2015)
Discarding device blocks: done                            
Creating filesystem with 32768 1k blocks and 8192 inodes
Filesystem UUID: 51ab1063-a137-48e5-a6f4-4552dad3b898
Superblock backups stored on blocks:
    8193, 24577

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```



#### 3) 将rootfs烧写到SD卡：

```shell
# sudo mount -t ext3 rootfs.ext3 /mnt -o loop
# sudo cp -rf rootfs/* /mnt/
# sudo umount /mnt
```



在开发过程中，如果需要修改SD卡中的内容，可以将SD卡的镜像rootfs.ext3挂载到/mnt目录下，直接操作/mnt来修改；

```shell
# sudo mount -t ext3 rootfs.ext3 /mnt -o loop
```



```shell
# cp rootfs.ext3 ~/qemu
```



## 6. 验证



### 1) Qemu启动命令：

```shell
# qemu-system-arm -M vexpress-a9 -m 512M -kernel ~/qemu/zImage -dtb ~/qemu/vexpress-v2p-ca9.dtb -nographic -append "root=/dev/mmcblk0 rw console=ttyAMA0" -sd rootfs.ext3
```



### 2) 启动脚本：

```shell
# boot.sh
#! /bin/sh
qemu-system-arm \
        -M vexpress-a9  \
        -m 512M \
        -kernel ~/qemu/zImage   \   
        -dtb ~/qemu/vexpress-v2p-ca9.dtb    \   
        -nographic  \
        -append "root=/dev/mmcblk0 rw console=ttyAMA0"    \
        -sd rootfs.ext3
```

以上为在串口终端启动系统，按照以下的启动命令可以使用LCD屏作为输出启动系统。



### 3) 图形化启动内核：

```shell
qemu-system-arm -M vexpress-a9 -m 512M -kernel ~/qemu/zImage -dtb ~/qemu/vexpress-v2p-ca9.dtb -append "root=/dev/mmcblk0 rw console=tty0" -sd rootfs.ext3
```



### 4) 启动验证：

```
request_module: kmod_concurrent_max (0) close to 0 (max_modprobes: 50), for module binfmt-464c, throttling...
request_module: modprobe binfmt-464c cannot be processed, kmod busy with 50 threads for more than 5 seconds now
Starting init: /sbin/init exists but couldn't execute it (error -8)
request_module: kmod_concurrent_max (0) close to 0 (max_modprobes: 50), for module binfmt-464c, throttling...
request_module: modprobe binfmt-464c cannot be processed, kmod busy with 50 threads for more than 5 seconds now
Starting init: /bin/sh exists but couldn't execute it (error -8)
Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.
CPU: 0 PID: 1 Comm: swapper/0 Not tainted 4.14.172 #1
Hardware name: ARM-Versatile Express
[<8010f180>] (unwind_backtrace) from [<8010b444>] (show_stack+0x10/0x14)
[<8010b444>] (show_stack) from [<806616cc>] (dump_stack+0x94/0xa8)
[<806616cc>] (dump_stack) from [<8011d67c>] (panic+0xdc/0x248)
[<8011d67c>] (panic) from [<80674634>] (kernel_init+0x104/0x114)
[<80674634>] (kernel_init) from [<801076a8>] (ret_from_fork+0x14/0x2c)
---[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.
QEMU: Terminated
```

如上问题是由于编译生成的busybox工具，是x86环境下使用的：

```shell
# file bin/busybox 
bin/busybox: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=7fe433943e201f5337be6116a883d54fc1a4a349, stripped
```

是因为在安装busybox的时候，使用了make install，应该使用

```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install
```

编译工具生成arm平台使用的busybox工具：

```shell
# file rootfs/bin/busybox
rootfs/bin/busybox: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-, for GNU/Linux 3.2.0, BuildID[sha1]=cbcd33b8d6c946cb19408a5e8e714de554c87f52, stripped
```

再次验证：

```
rtc-pl031 10017000.rtc: setting system clock to 2018-09-24 13:22:14 UTC (1537795334)
ALSA device list:
  #0: ARM AC'97 Interface PL041 rev0 at 0x10004000, irq 33
input: ImExPS/2 Generic Explorer Mouse as /devices/platform/smb/smb:motherboard/smb:motherboard:iofpga@7,00000000/10007000.kmi/serio1/input/input2
EXT4-fs (mmcblk0): mounting ext3 file system using the ext4 subsystem
EXT4-fs (mmcblk0): recovery complete
EXT4-fs (mmcblk0): mounted filesystem with ordered data mode. Opts: (null)
VFS: Mounted root (ext3 filesystem) on device 179:0.
Freeing unused kernel memory: 284K
random: nonblocking pool is initialized
can't run '/etc/init.d/rcS': No such file or directory

Please press Enter to activate this console.
/ #
/ #
/ # uname -a
Linux (none) 4.4.157 #1 SMP Sun Sep 23 21:11:22 CST 2018 armv7l GNU/Linux
```

至此，Qemu启动Linux内核并挂载跟文件系统已经启动成功，通过串口终端可以正常和系统进行简单功能的交互。
打印中提示的不能运行/etc/init.d/rcS问题，只需要添加/etc/init.d/rcS文件即可，文件内容可以是提示语句。

```shell
# cat /etc/init.d/rcS
Hello Qemu Linux!
```

注意，要在创建/etc/init.d/rcS文件时，记着修改该文件的可执行权限，否则启动过程中会报错：

```
Freeing unused kernel memory: 1024K
random: crng init done
can't run '/etc/init.d/rcS': Permission denied

Please press Enter to activate this console. 
```



## 7. 退出Qemu环境

Qemu环境搭建好之后，在出错时需要关闭并重新启动Qemu，不用的时候需要关闭Qemu。



### 1）手动退出Qemu

```
Ctrl + A; X
```

操作之后，终端上会打印：

```
QEMU: Terminated
```



### 2）强制退出Qemu

有时候会发现无法通过shutdown等工具关闭，因为Qemu也是一个进程，可以通过杀掉Qemu进程的方法关闭Qemu模拟环境。

```shell
# ps -a
# kill xxx
```

如下可以采用脚本运行：

```shell
# cat kill_qemu.sh 
#! /bin/sh
ps -a | grep qemu-system-arm | awk '{print $1}' | xargs sudo kill -9
```

本文讲述了Qemu环境启动Linux内核，并挂载SD卡中的根文件系统的一些操作步骤。如果需要在Qemu环境下以ARM开发板的正常启动流程来加载Linux内核并挂载根文件系统，可以参考下一篇文章《[Qemu搭建ARM vexpress开发环境(2)----通过u-boot启动Linux内核](https://www.jianshu.com/p/8619a6739040
)》。