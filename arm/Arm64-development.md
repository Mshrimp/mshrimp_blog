



搭建环境



kernel.org

下载：linux-4.14.172.tar.gz



### 下载

#### 方法一

linaro.org



https://www.linaro.org/downloads/

![Screenshot from 2020-03-11 21-24-02](Arm64-development/Screenshot from 2020-03-11 21-24-02.png)



https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/



![Screenshot from 2020-03-11 21-25-34](Arm64-development/Screenshot from 2020-03-11 21-25-34.png)





#### 方法二



```
$ wget https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
--2020-03-12 21:25:35--  https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
Resolving releases.linaro.org (releases.linaro.org)... 13.228.101.204
Connecting to releases.linaro.org (releases.linaro.org)|13.228.101.204|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://publishing-ap-linaro-org.s3.amazonaws.com/releases/components/toolchain/binaries/latest-7/aarch64-linux-gnu/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz?Signature=KxyNlsZpN2fR%2F2CYjq%2BtAhB9Aco%3D&Expires=1584019631&AWSAccessKeyId=AKIAIELXV2RYNAHFUP7A [following]
--2020-03-12 21:25:41--  https://publishing-ap-linaro-org.s3.amazonaws.com/releases/components/toolchain/binaries/latest-7/aarch64-linux-gnu/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz?Signature=KxyNlsZpN2fR%2F2CYjq%2BtAhB9Aco%3D&Expires=1584019631&AWSAccessKeyId=AKIAIELXV2RYNAHFUP7A
Resolving publishing-ap-linaro-org.s3.amazonaws.com (publishing-ap-linaro-org.s3.amazonaws.com)... 52.219.128.96
Connecting to publishing-ap-linaro-org.s3.amazonaws.com (publishing-ap-linaro-org.s3.amazonaws.com)|52.219.128.96|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 117896452 (112M) [application/x-tar]
Saving to: ‘gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz’

1.03M  7.69KB/s    eta 3h 41m
```





#### 方法三





### 安装



```
$ /usr/local/arm-gcc/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-gcc -v
Using built-in specs.
COLLECT_GCC=/usr/local/arm-gcc/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-gcc
COLLECT_LTO_WRAPPER=/usr/local/arm-gcc/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin/../libexec/gcc/aarch64-linux-gnu/7.5.0/lto-wrapper
Target: aarch64-linux-gnu
Configured with: '/home/tcwg-buildslave/workspace/tcwg-make-release_0/snapshots/gcc.git~linaro-7.5-2019.12/configure' SHELL=/bin/bash --with-mpc=/home/tcwg-buildslave/workspace/tcwg-make-release_0/_build/builds/destdir/x86_64-unknown-linux-gnu --with-mpfr=/home/tcwg-buildslave/workspace/tcwg-make-release_0/_build/builds/destdir/x86_64-unknown-linux-gnu --with-gmp=/home/tcwg-buildslave/workspace/tcwg-make-release_0/_build/builds/destdir/x86_64-unknown-linux-gnu --with-gnu-as --with-gnu-ld --disable-libmudflap --enable-lto --enable-shared --without-included-gettext --enable-nls --with-system-zlib --disable-sjlj-exceptions --enable-gnu-unique-object --enable-linker-build-id --disable-libstdcxx-pch --enable-c99 --enable-clocale=gnu --enable-libstdcxx-debug --enable-long-long --with-cloog=no --with-ppl=no --with-isl=no --disable-multilib --enable-fix-cortex-a53-835769 --enable-fix-cortex-a53-843419 --with-arch=armv8-a --enable-threads=posix --enable-multiarch --enable-libstdcxx-time=yes --enable-gnu-indirect-function --with-build-sysroot=/home/tcwg-buildslave/workspace/tcwg-make-release_0/_build/sysroots/aarch64-linux-gnu --with-sysroot=/home/tcwg-buildslave/workspace/tcwg-make-release_0/_build/builds/destdir/x86_64-unknown-linux-gnu/aarch64-linux-gnu/libc --enable-checking=release --disable-bootstrap --enable-languages=c,c++,fortran,lto --build=x86_64-unknown-linux-gnu --host=x86_64-unknown-linux-gnu --target=aarch64-linux-gnu --prefix=/home/tcwg-buildslave/workspace/tcwg-make-release_0/_build/builds/destdir/x86_64-unknown-linux-gnu
Thread model: posix
gcc version 7.5.0 (Linaro GCC 7.5-2019.12)
```







-----------





```
$ make arch=ARM64 CROSS_COMPILE=/usr/local/arm-gcc/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu- menuconfig

Command 'make' not found, but can be installed with:

sudo apt install make      
sudo apt install make-guile
```



```
$ sudo apt install make
[sudo] password for xiami: 
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Suggested packages:
  make-doc
The following NEW packages will be installed:
  make
0 upgraded, 1 newly installed, 0 to remove and 37 not upgraded.
Need to get 154 kB of archives.
After this operation, 381 kB of additional disk space will be used.
Get:1 http://cn.archive.ubuntu.com/ubuntu bionic/main amd64 make amd64 4.1-9.1ubuntu1 [154 kB]
Fetched 154 kB in 5s (31.2 kB/s)
Selecting previously unselected package make.
(Reading database ... 168814 files and directories currently installed.)
Preparing to unpack .../make_4.1-9.1ubuntu1_amd64.deb ...
Unpacking make (4.1-9.1ubuntu1) ...
Setting up make (4.1-9.1ubuntu1) ...
Processing triggers for man-db (2.8.3-2ubuntu0.1) ...

```



```
$ sudo apt install make-guile
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Suggested packages:
  make-doc
The following packages will be REMOVED:
  make
The following NEW packages will be installed:
  make-guile
0 upgraded, 1 newly installed, 1 to remove and 37 not upgraded.
Need to get 155 kB of archives.
After this operation, 4,096 B of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://cn.archive.ubuntu.com/ubuntu bionic/universe amd64 make-guile amd64 4.1-9.1ubuntu1 [155 kB]
Fetched 155 kB in 4s (41.8 kB/s)     
(Reading database ... 168829 files and directories currently installed.)
Removing make (4.1-9.1ubuntu1) ...
Selecting previously unselected package make-guile.
(Reading database ... 168814 files and directories currently installed.)
Preparing to unpack .../make-guile_4.1-9.1ubuntu1_amd64.deb ...
Unpacking make-guile (4.1-9.1ubuntu1) ...
Setting up make-guile (4.1-9.1ubuntu1) ...
Processing triggers for man-db (2.8.3-2ubuntu0.1) ...

```





```
$ make arch=ARM64 CROSS_COMPILE=/usr/local/arm-gcc/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu- defconfig
  HOSTCC  scripts/basic/fixdep
/bin/sh: 1: gcc: not found
scripts/Makefile.host:102: recipe for target 'scripts/basic/fixdep' failed
make[1]: *** [scripts/basic/fixdep] Error 127
Makefile:464: recipe for target 'scripts_basic' failed
make: *** [scripts_basic] Error 2

```



```
#apt  install  libncurses5-dev
# apt-get  install  libx11-dev:i386 libreadline6-dev:i386
#apt-get install build-essential
# apt-get  install  lib32stdc++6
# apt-get  install  lib32z1
```















32位



sudo apt-get install gcc-arm-linux-gnueabi



$ arm-linux-gnueabi-gcc -v
Using built-in specs.
COLLECT_GCC=arm-linux-gnueabi-gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc-cross/arm-linux-gnueabi/7/lto-wrapper
Target: arm-linux-gnueabi
Configured with: ../src/configure -v --with-pkgversion='Ubuntu/Linaro 7.5.0-3ubuntu1~18.04' --with-bugurl=file:///usr/share/doc/gcc-7/README.Bugs --enable-languages=c,ada,c++,go,d,fortran,objc,obj-c++ --prefix=/usr --with-gcc-major-version-only --program-suffix=-7 --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --with-sysroot=/ --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-libitm --disable-libquadmath --disable-libquadmath-support --enable-plugin --with-system-zlib --with-target-system-zlib --enable-multiarch --enable-multilib --disable-sjlj-exceptions --with-arch=armv5t --with-float=soft --disable-werror --enable-multilib --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=arm-linux-gnueabi --program-prefix=arm-linux-gnueabi- --includedir=/usr/arm-linux-gnueabi/include
Thread model: posix
gcc version 7.5.0 (Ubuntu/Linaro 7.5.0-3ubuntu1~18.04) 





$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig -j4 O=./object

$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j4 O=./object





```
sudo apt install exuberant-ctags
```




