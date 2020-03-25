---
title: Qemu搭建ARM vexpress开发环境(三)----NFS网络根文件系统
date: 2019-09-30 21:00:00
tags: Qemu 

---

标签： Qemu ARM Linux u-boot

---

经过上一篇《[Qemu搭建ARM vexpress开发环境(二)----通过u-boot启动Linux内核](https://www.jianshu.com/p/8619a6739040)》，已经实现了通过u-boot加载Kernel启动开发板，并且挂载根文件系统，本文讲述通过NFS网络挂载根文件系统。

通过NFS网络根文件系统，可以实现开发板在通过u-boot启动内核后，通过NFS网络在别的PC主机上挂载根文件系统。对于开发调试阶段的工作学习提供了很大的便利，可以直接在Linux主机上开发、编译驱动或者APP，并将目标文件拷贝到NFS服务目录中进行使用（此时文件相当于被拷贝到了开发板的根文件系统中）。也可以在主机端直接修改rootfs文件系统中别的文件，等效于在开发板上直接修改。

<!--more-->



## 目录

[TOC]



本文来介绍NFS挂载网络根文件系统的操作步骤，本方法不仅仅适用于Qemu搭建的ARM vexpress开发板环境，也适用于所有其他的开发板实体。

由于各个开发板的NFS网络文件系统制作方法是相同的，也可以参考Exynos4412和NanopiNEO开发板环境搭建中的NFS网络文件系统制作方法部分内容。



## 1. 环境配置

Linux主机支持NFS服务
修改bootargs启动参数
    设置NFS为根文件系统
    设置主机NFS文件系统地址
内核支持NFS挂载文件系统



## 2. 安装并配置NFS服务



### 2.1 Linux主机开启NFS服务



#### 1) 安装NFS

```
# sudo apt install nfs-kernel-server
```



#### 2) 配置NFS

```
# vim /etc/exports
// 添加NFS共享目录
/home/mcy/qemu/rootfs    *(rw, sync, no_root_squash, no_subtree_check)
    rw    可读可写操作
    sync    内存和磁盘上的内容保持同步
    no_root_squash    Linux主机不再将开发板设置为匿名用户，可以操作文件读写
    no_subtree_check    不检查根文件系统子目录文件
```


#### 3) 重启NFS服务

```
sudo /etc/init.d/rpcbind restart
sudo /etc/init.d/nfs-kernel-server restart
```
或者：
```
# systemctl restart nfs-kernel-server
```



#### 4) 检查NFS共享目录是否创建

```
# sudo showmount -e
Export list for mcy-VirtualBox:
/home/mcy/qemu/rootfs *
```
注：
使用NFS网络文件系统时，需要Linux主机关闭系统防火墙，否则，系统在运行时会出现异常。



### 2.2 开发板配置支持NFS网络

修改u-boot中的启动参数：
```
# vim include/configs/
CONFIG_BOOTCOMMAND
    setenv bootargs 'root=/dev/nfs rw    \
    nfsroot=192.168.0.105:/home/mcy/qemu/rootfs init=/linuxrc    \
    ip=192.168.0.110 console=ttyAMA0';    \
```

配置内核支持NFS挂载文件系统



完善NFS文件系统
重启reboot命令



## 3. 制作根文件系统

编译busybox
```
nfs
Linux System Utilities  --->
    [*] mount (30 kb)
        [*]   Support mounting NFS file systems on Linux < 2.6.23
```

创建rootfs目录，并在rootfs目录下创建文件：
```
# mkdir etc
# cd etc
# vim inittab
::sysinit:/etc/init.d/rcS        // 执行rcS脚本
#::respawn:-/bin/sh
#tty2::askfirst:-/bin/sh
#::ctrlaltdel:/bin/umount -a -r

console::askfirst:-/bin/sh
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
```
```
# vim init.d/rcS
#! /bin/sh
PATH=/sbin:/bin:/user/sbin:/usr/bin
LD_LIBRARY_PATH=/lib
export PATH LD_LIBRARY_PATH

mount -a        // 挂载根文件系统 fstab
mkdir -p /dev/pts
mount -t devpts devpts dev/pts
mdev -s
mkdir -p /var/lock

echo "......"
```

```
# vim fstab
proc    /proc    proc    defaults    0    0
tmpfs    /tmp    tmpfs    default    0    0
sysfs    /sys    sysfs    default    0    0
tmpfs    /dev    tmpfs    default    0    0
var    /dev    tmpfs    default    0    0
ramfs    /dev    ramfs    default    0    0
```
```
# vim profile
PS1='xiami@vexpress:\w #'
export PS1
```
也可以在~/.bashrc中修改或设置PS1



启动流程：
Linux内核启动之后，挂载根文件系统
开启init进程，bootargs init=/linuxrc，启动第一个用户进程
在用户进程中读取inittab脚本，


构建其他目录
其他的目录可以是空目录
```
# cd rootfs
# mkdir proc mnt tmp sys root
```