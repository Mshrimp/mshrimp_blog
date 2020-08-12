# Qemu搭建ARM vexpress开发环境(3)----NFS网络根文件系统



## 目录

[TOC]

经过上一篇《[Qemu搭建ARM vexpress开发环境(2)----通过u-boot启动Linux内核](https://www.jianshu.com/p/8619a6739040)》，已经实现了通过u-boot加载Kernel启动开发板，并且挂载根文件系统，本文讲述通过NFS网络挂载根文件系统。

通过NFS网络根文件系统，可以实现开发板在通过u-boot启动内核后，通过NFS网络在别的PC主机上挂载根文件系统。对于开发调试阶段的工作学习提供了很大的便利，可以直接在Linux主机上开发、编译驱动或者APP，并将目标文件拷贝到NFS服务目录中进行使用（此时文件相当于被拷贝到了开发板的根文件系统中）。也可以在主机端直接修改rootfs文件系统中别的文件，等效于在开发板上直接修改。

<!--more-->

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
	*	允许所有网段访问，也可以使用具体的IP地址
    rw    可读可写操作
    sync    内存和磁盘上的内容保持同步
    no_root_squash    Linux主机不再将开发板设置为匿名用户，可以操作文件读写
    no_subtree_check    不检查根文件系统子目录文件
```

#### 3) 重启NFS服务

因为NFS是一个RPC程序，使用它之前，需要映射好端口，通过rpcbind来完成；所以需要重启rpcbin服务和nfs服务；



```
$ sudo /etc/init.d/rpcbind restart
[ ok ] Restarting rpcbind (via systemctl): rpcbind.service.
```



```
$ sudo /etc/init.d/rpcbind status
● rpcbind.service - RPC bind portmap service
   Loaded: loaded (/lib/systemd/system/rpcbind.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-03-26 21:22:13 CST; 2min 19s ago
     Docs: man:rpcbind(8)
 Main PID: 6779 (rpcbind)
    Tasks: 1 (limit: 4448)
   CGroup: /system.slice/rpcbind.service
           └─6779 /sbin/rpcbind -f -w

Mar 26 21:22:13 xiami systemd[1]: Starting RPC bind portmap service...
Mar 26 21:22:13 xiami systemd[1]: Started RPC bind portmap service.
```





```
$ sudo /etc/init.d/nfs-kernel-server restart
[ ok ] Restarting nfs-kernel-server (via systemctl): nfs-kernel-server.service.
```



```
$ sudo /etc/init.d/nfs-kernel-server status
● nfs-server.service - NFS server and services
   Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor preset: enabled)
   Active: active (exited) since Thu 2020-03-26 21:32:46 CST; 19s ago
  Process: 8259 ExecStopPost=/usr/sbin/exportfs -f (code=exited, status=0/SUCCESS)
  Process: 8258 ExecStopPost=/usr/sbin/exportfs -au (code=exited, status=0/SUCCESS)
  Process: 8257 ExecStop=/usr/sbin/rpc.nfsd 0 (code=exited, status=0/SUCCESS)
  Process: 8267 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 8266 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 8267 (code=exited, status=0/SUCCESS)

Mar 26 21:32:45 xiami systemd[1]: Starting NFS server and services...
Mar 26 21:32:46 xiami systemd[1]: Started NFS server and services.
```





```
$ sudo /etc/init.d/nfs-kernel-server restart
[....] Restarting nfs-kernel-server (via systemctl): nfs-kernel-server.serviceJob for nfs-server.service canceled.
 failed!
```

如果出现以上报错，可能是/etc/exports文件设置的NFS目录错误；比如我的设置错误是因为括号中的各个选项之间多了空格：

```
# vim /etc/exports
/home/xiami/qemu/nfs_root	*(rw, sync, no_root_squash, no_subtree_check)
修改为：
/home/xiami/qemu/nfs_root	*(rw,sync,no_root_squash,no_subtree_check)
```



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
Export list for xiami:
/home/xiami/qemu/rootfs *
```
注：
使用NFS网络文件系统时，需要Linux主机关闭系统防火墙，否则，系统在运行时会出现异常。



Ubuntu下关闭防火墙操作：

```
# sudo ufw disable
```

查看防火墙状态：

```
# sudo ufw status
Status: inactive
```

开启防火墙：

```
# sudo ufw enable
```





### 2.2 开发板配置支持NFS网络

修改u-boot中的启动参数：
```
# vim include/configs/vexpress_common.h
#define CONFIG_BOOTCOMMAND  \
    "tftp 0x60003000 uImage; tftp 0x60500000 vexpress-v2p-ca9.dtb;  \
    setenv bootargs 'root=/dev/nfs rw   \
    nfsroot=192.168.1.102:/home/xiami/qemu/nfs_root init=/linuxrc   \
    ip=192.168.1.110 console=ttyAMA0';  \
    bootm 0x60003000 - 0x60500000;"
```

配置内核支持NFS挂载文件系统



完善NFS文件系统
重启reboot命令



## 3. 制作根文件系统

编译busybox
```
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig

Linux System Utilities  --->
    [*] mount (30 kb)
        [*]   Support mounting NFS file systems on Linux < 2.6.23
```



```
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig
scripts/kconfig/mconf Config.in
#
# using defaults found in .config
#


*** End of configuration.
*** Execute 'make' to build the project or try 'make help'.
```

编译：

```
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
```



安装：

```
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install
```

提示下边信息，表示安装成功：

```
--------------------------------------------------
You will probably need to make your busybox binary
setuid root to ensure all configured applets will
work properly.
--------------------------------------------------
```





创建rootfs目录，并在rootfs目录下创建文件：



```
# mkdir dev
# cd dev
# sudo mknod -m 666 tty1 c 4 1
# sudo mknod -m 666 tty2 c 4 2
# sudo mknod -m 666 tty3 c 4 3
# sudo mknod -m 666 tty4 c 4 4
# sudo mknod -m 666 console c 5 1
# sudo mknod -m 666 null c 1 3
# ls -l
crw-rw-rw- 1 root root 5, 1 Mar 26 22:16 console
crw-rw-rw- 1 root root 1, 3 Mar 26 22:16 null
crw-rw-rw- 1 root root 4, 1 Mar 26 22:18 tty1
crw-rw-rw- 1 root root 4, 2 Mar 26 22:18 tty2
crw-rw-rw- 1 root root 4, 3 Mar 26 22:19 tty3
crw-rw-rw- 1 root root 4, 4 Mar 26 22:19 tty4
```





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







```
VFS: Unable to mount root fs via NFS, trying floppy.
VFS: Cannot open root device "nfs" or unknown-block(2,0): error -6
Please append a correct "root=" boot option; here are the available partitions:
1f00          131072 mtdblock0 
 (driver?)
1f01           32768 mtdblock1 
 (driver?)
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(2,0)
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
---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(2,0)
```











```
  │                                                                         │  
  │ Symbol: ROOT_NFS [=y]                                                   │  
  │ Type  : boolean                                                         │  
  │ Prompt: Root file system on NFS                                         │  
  │   Location:                                                             │  
  │     -> File systems                                                     │  
  │ (3)   -> Network File Systems (NETWORK_FILESYSTEMS [=y])                │  
  │   Defined at fs/nfs/Kconfig:160                                         │  
  │   Depends on: NETWORK_FILESYSTEMS [=y] && NFS_FS [=y]=y && IP_PNP [=y]  │  

```



```
sudo mount -o nolock -t nfs 192.168.1.104:/home/xiami/qemu/nfs_root /mnt
```





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