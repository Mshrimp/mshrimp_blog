---
title: Qemu搭建ARM vexpress开发环境(一)
date: 2019-07-30 21:37:50
tags:
---

标签： Qemu ARM Linux

---

嵌入式软件开发依赖于嵌入式硬件设备，比如：开发板、外部模块设备等，但是如果只是想学习、研究Linux内核，想学习Linux内核的架构，工作模式，需要修改一些代码，重新编译并烧写到开发板中进行验证，这样未必有些复杂，并且为此专门购买各种开发版，浪费资金，开会演示效果还需要携带一大串的板子和电线，不胜其烦。然而Qemu的使用可以避免频繁在开发板上烧写版本，如果进行的调试工作与外设无关，仅仅是内核方面的调试，Qemu模拟ARM开发环境完全可以完美地胜任。

<!--more-->

下面简单介绍下我的Qemu开发环境搭建过程

## 1. 环境
由于在开发过程中也需要Windows系统下的一些工具，双系统环境切换操作系统时必须重启，于是放弃了以前搭建的双系统环境，而采用在PC的Windows10系统下通过VirtualBox虚拟机安装Xubuntu系统进行开发，避免了双系统开发中需要不断重启切换PC系统的问题。Xubuntu系统和Ubuntu系统大同小异，只是桌面封装更加简洁。

### 1.1 所使用环境
PC系统：Windows10
虚拟机：VirtualBox-5.18
虚拟机系统：Xubuntu
模拟的开发板：vexpress

### 1.2 搭建环境时使用的工具
Qemu-2.7
linux-4.4.157(Linux Kernel)
u-boot-2017.05
busybox-1.29.3
arm-linux-gnueabi-

## 2. 安装交叉编译工具
```
# sudo apt install gcc-arm-linux-gnueabi
```

## 3. 安装Qemu工具
有两种方法可以在Linux环境下安装Qemu工具，第一种直接使用XUbuntu系统的apt工具安装，但是这种方法安装的Qemu系统版本不是最新的，如果需要安装最新版本的Qemu工具，就需要通过Git工具下载源码，切换到最新分支再去编译安装了。具体操作如下所述

### 3.1 快速安装Qemu：
```
# sudo apt install qemu
```

### 3.2 下载Qemu源码编译安装

从Git服务器下载Qemu代码，记着在下载之前选择并切换需要的源码分支：
```
# git clone git://git.qemu-project.org/qemu.git
```
编译并安装Qemu：
```
# ./configure --target-list=arm-softmmu --audio-drv-list=
# make
# make install
```

### 3.3 查看Qemu版本
```
# qemu-system-arm --version
QEMU emulator version 2.7.1 (v2.7.1-dirty), Copyright (c) 2003-2016 Fabrice Bellard and the QEMU Project developers
```
### 3.4 查看Qemu支持的开发板
Qemu工具支持大量开发板的虚拟，现存的大部分常用开发板都能很好地支持。通过下面的命令操作可以看到当前版本的Qemu工具支持的开发板列表：
```
# qemu-system-arm -M help
```

### 3.5 运行Qemu
该操作目前还不能运行，因为还没有编译内核，如果手边有编译好的别的版本的zImage文件，可以通过下面命令尝试运行看下效果。
```
# qemu-system-arm -M vexpress-a9 -m 512M -kernel ./zImage -dtb ./vexpress-v2p-ca9.dtb -nographic -append "console=ttyAMA0"
    -M          指定开发板
    -m          指定内存大小
    -kernel     指定内核文件
    -dtb        指定dtb文件
    -nographic  指定不需要图形界面
    -append     指定扩展显示界面，串口或者LCD
```

## 4. 配置并编译Linux内核

### 4.1 下载Linux内核

通过众所周知的内核下载网站www.kernel.org下载需要的内核版本，这里我下载的是相对来说最新的长期支持的内核版本linux-4.4.157。

### 4.2 解压Linux内核
```
# tar -xvf linux-4.4.157.tar.xz
```

### 4.3 编译Linux内核
```
# make vexpress_defconfig
# make zImage -j4
# make modules -j4    // 编译驱动模块
# make dtbs
```
得到编译文件：
```
arch/arm/boot/zImage
arch/arm/boot/dts/*.dtb
```

### 4.4 Qemu启动命令
```
# qemu-system-arm -M vexpress-a9 -m 512M -kernel kernel/linux-4.4.157/arch/arm/boot/zImage -dtb kernel/linux-4.4.157/arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "console=ttyAMA0"
```
Qemu的启动命令需要带好几个参数，完成启动命令比较长，每次都输入很可能会出现错误，为了使用方便，可以将该命令放到shell脚本中执行：
```
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
内核成功启动，内核的启动打印信息非常多，为避免累赘，前半部分的启动日志省略。启动最后出错是因为没有挂载根文件系统。
```
input: ImExPS/2 Generic Explorer Mouse as /devices/platform/smb/smb:motherboard/smb:motherboard:iofpga@7,00000000/10007000.kmi/serio1/input/input2
VFS: Cannot open root device "(null)" or unknown-block(0,0): error -6
Please append a correct "root=" boot option; here are the available partitions:
1f00          131072 mtdblock0  (driver?)
1f01           32768 mtdblock1  (driver?)
b300           32768 mmcblk0  driver: mmcblk
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
CPU: 0 PID: 1 Comm: swapper/0 Not tainted 4.4.157 #1
Hardware name: ARM-Versatile Express
[<80016420>] (unwind_backtrace) from [<80012e80>] (show_stack+0x10/0x14)
[<80012e80>] (show_stack) from [<802478f8>] (dump_stack+0x84/0x98)
[<802478f8>] (dump_stack) from [<800a7d7c>] (panic+0x9c/0x1f4)
[<800a7d7c>] (panic) from [<806302d4>] (mount_block_root+0x1c8/0x268)
[<806302d4>] (mount_block_root) from [<80630498>] (mount_root+0x124/0x12c)
[<80630498>] (mount_root) from [<806305f0>] (prepare_namespace+0x150/0x198)
[<806305f0>] (prepare_namespace) from [<8062fedc>] (kernel_init_freeable+0x250/0x260)
[<8062fedc>] (kernel_init_freeable) from [<804a99f0>] (kernel_init+0x8/0xe8)
[<804a99f0>] (kernel_init) from [<8000f490>] (ret_from_fork+0x14/0x24)
---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```



## 5. 制作根文件系统

使用busybox制作简易的根文件系统

### 5.1 下载busybox工具
从https://busybox.net/downloads/下载最新的busybox。

### 5.2 解压busybox
```
# tar -xvf busybox-1.29.3.tar.bz2
```

### 5.3 配置并编译busybox

修改Makefile:
```
ARCH = arm
CROSS_COMPILE = arm-linux-gnueabi-
```
编译选择使用glibc动态库，因为静态库可能会出现一些未知的问题
```
# make menuconfig
Settings  --->
    Build Options  --->
        [ ] Build static binary (no shared libs)
```


编译并安装：
```
# make
# make install
```

### 5.4 制作简易根文件系统

制作一个简易的根文件系统，该文件系统包含的功能极其简陋，仅为了验证Qemu启动Linux内核后挂载跟文件系统的过程。以后会根据具体需要进一步完善该文件系统。

#### 1) 编译并安装busybox
将busybox编译生成的_install目录下的文件全部拷贝到rootfs/目录：
```
# mkdir rootfs
# cp /.../busybox-1.29.3/_install/* rootfs/ -rfd
```
也可以在指定busybox的安装目录直接安装：
```
# make CONFIG_PREFIX=/.../rootfs/ install
```

#### 2) 安装glibc库
在根文件系统中添加加载器和动态库：
```
# mkdir rootfs/lib
# cp /usr/arm-linux-gnueabi/lib/* rootfs/lib/ -rfp
```

#### 3) 静态创建设备文件
```
# mkdir rootfs/dev
# cd rootfs/dev
# mknod -m 666 tty1 c 4 1
# mknod -m 666 tty2 c 4 2
# mknod -m 666 tty3 c 4 3
# mknod -m 666 tty4 c 4 4
# mknod -m 666 console c 5 1
# mknod -m 666 null c 1 3
```
至此，简易版根文件系统就制作完成，该根文件系统只含有最基本的功能，一些其他功能在以后的操作中会进行添加，如有兴趣可以继续参考下一篇文章《》《》

### 5.5 制作SD卡文件系统镜像

#### 1) 生成一个空的SD卡镜像：
```
# dd if=/dev/zero of=rootfs.ext3 bs=1M count=32
32+0 records in
32+0 records out
33554432 bytes (34 MB, 32 MiB) copied, 0.0236764 s, 1.4 GB/s
```

#### 2) 将SD卡格式化为exts文件系统：
```
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
```
# sudo mount -t ext3 rootfs.ext3 /mnt -o loop
# sudo cp -rf rootfs/* /mnt/
# sudo umount /mnt
```

### 5.6 验证

#### 1) Qemu启动命令：
```
qemu-system-arm -M vexpress-a9 -m 512M -kernel kernel/linux-4.4.157/arch/arm/boot/zImage -dtb kernel/linux-4.4.157/arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "root=/dev/mmcblk0 rw console=ttyAMA0" -sd rootfs.ext3
```

#### 2) 启动脚本：
```
# boot.sh
#! /bin/sh
qemu-system-arm \
        -M vexpress-a9  \
        -m 512M \
        -kernel kernel/linux-4.4.157/arch/arm/boot/zImage   \   
        -dtb kernel/linux-4.4.157/arch/arm/boot/dts/vexpress-v2p-ca9.dtb    \   
        -nographic  \
        -append "root=/dev/mmcblk0 rw console=ttyAMA0"    \
        -sd rootfs.ext3
```

以上为在串口终端启动系统，按照以下的启动命令可以使用LCD屏作为输出启动系统。

#### 3) 图形化启动内核：
```
qemu-system-arm -M vexpress-a9 -m 512M -kernel kernel/linux-4.4.157/arch/arm/boot/zImage -dtb kernel/linux-4.4.157/arch/arm/boot/dts/vexpress-v2p-ca9.dtb -append "root=/dev/mmcblk0 rw console=tty0" -sd rootfs.ext3
```

#### 4) 启动日志：
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

```
# cat /etc/init.d/rcS
Hello Qemu Linux!
```

### 5.7 退出Qemu环境

Qemu环境搭建好之后，在出错时需要关闭并重新启动Qemu，不用的时候需要关闭Qemu。

##### 1）手动退出Qemu
```
Ctrl + A; X
```

##### 2）强制退出Qemu

有时候会发现无法通过shutdown等工具关闭，因为Qemu也是一个进程，可以通过杀掉Qemu进程的方法关闭Qemu模拟环境。

```
# ps -a
# kill xxx
```

如下可以采用脚本运行：

```
# cat kill_qemu.sh 
#! /bin/sh
ps -a | grep qemu-system-arm | awk '{print $1}' | xargs sudo kill -9
```

本文讲述了Qemu环境启动Linux内核，并挂载SD卡中的根文件系统的一些操作步骤。如果需要在Qemu环境下以ARM开发板的正常启动流程来加载Linux内核并挂载根文件系统，可以参考下一篇文章《[Qemu搭建ARM vexpress开发环境(二)----通过u-boot启动Linux内核](https://www.jianshu.com/p/8619a6739040
)》。