---
title: initrd概述
date: 2020-07-10 15:03:30
tags: Linux
---

## initrd 是什么

initrd(initial ram disk)是在实际根文件系统可用之前挂载到系统中的一个初始根文件系统，它包含了必要的可执行程序和系统文件，例如将内核模块加载到内存中使用的 insmod 工具。initrd 与内核绑定在一起，作为内核引导过程的一部分进行加载，通过 initrd 加载一些必要的模块，这样才能稍后挂载真正的根文件系统。

> linux 内核在初始化完成后会执行 init 进程，而 init 进程会挂载我们的根文件系统，但由于 init 程序也是在根文件系统上的，所以就有了悖论。为了解决这个问题，linux 2.6 之前除了内核 vmlinuz 之外还有一个独立的 initrd.img（其实就是一个文件系统映像），linux 内核初始化后会挂载 initrd.img 作为一个临时的根文件系统，而 init 程序就在 initrd.img 中，然后 init 进程会挂载真正的根文件系统。linux 2.6 采用 initramfs(initial ram filesystem)，它是一个 cpio 格式的内存文件系统。

linux 内核启动 init 程序有两种方案：

1. ramdisk，把一块内存（ram）当做磁盘（disk）去挂载，然后找到 init 程序并执行（initrd 就是 ramdisk 的实现）
2. ramfs，直接在内存（ram）上挂载文件系统（fs），然后执行文件系统中的 init 程序（initramfs 就是 ramfs 的实现）

<!--more-->

## 构建 initrd

构建 initrd 映像文件需要使用到 busybox 工具，它是一个集成了三百多个最常用 linux 命令和工具的软件。busybox 包含了一些简单的工具，例如 ls、cat 和 echo 等等，还包含了一些更大、更复杂的工具，例 grep、find、mount 以及 telnet。busybox 的优点是它将很多工具打包成一个文件，同时还可以共享它们通用的元素，这样可以极大的减少映像文件的大小。

编译安装 busybox

1. [busybox 官网](https://www.busybox.net/)下载源码解压

   ```bash
   wget https://busybox.net/downloads/busybox-1.30.1.tar.bz2
   tar -xvf busybox-1.30.1.tar.bz2
   ```

2. 配置安装 busybox

   执行成功后会在当前目录生成 `_install` 目录 （如果 make 报错的话，执行 `yum install epel-release` 解决）

   ```bash
   # 安装必要的类库
   yum install -y glibc-static libmcrypt-devel

   cd busybox-1.30.1
   make menuconfig
   make
   make install
   ```

   `make menuconfig`选择`Build static binary(no shared libs)`

   ![busybox-menuconfig](/images/linux/busybox-menuconfig.png)

3. 打包生成 initrd.img

   根文件系统必要的目录

   ```bash
   cd _install
   mkdir proc sys etc dev
   ```

   dev 目录创建 console 和 null 设备文件

   ```bash
   cd dev
   mknod console c 5 1
   mknod null c 1 3
   ```

   etc 目录下的相关文件
   init 进程是内核创建的第一个进程，它会解释执行/etc/inittab 里的命令，然后执行/etc/init.d/rcS 脚本，在这个脚本中我们需要挂载文件系统，至于挂载哪一个文件系统由/etc/fstab 文件决定

   创建 etc/fstab 文件

   ```bash
   cd etc
   vim fstab
   ```

   fstab 文件内容如下：

   ```bash
   # cat etc/fstab
   proc /proc proc defaults 0 0
   sysfs /sys sysfs defaults 0 0
   ```

   创建 etc/init.d/rcS

   ```bash
   cat > etc/init.d/rcS << EOF
   #!/bin/sh
   mount -a
   EOF

   chmod +x etc/init.d/rcS
   ```

   创建 etc/inittab

   ```bash
   cat > etc/inittab << EOF
   ::sysinit:/etc/init.d/rcS
   console::respawn:-/bin/sh
   ::ctrlaltdel:/sbin/reboot
   ::shutdown:/bin/umount -a -r
   EOF
   ```

   然后切换到`_install`目录

   ```bash
   rm -rf linuxrc
   ln -sv bin/busybox init
   find . | cpio --quiet -H newc -o | gzip -9 -n > ../initrd.gz
   ```

   至此，initrd 映像文件已经构建完成，最后把生成的 initrd.gz 文件复制到 `/boot`目录

   ```bash
   cp ../initrd.gz /boot
   ```

## 使用 initrd 引导系统

重启系统，在出现 grub 引导菜单项的时候输入"c"进入 grub 命令模式，

```bash
# 指定内核映像文件路径（如果提示找不到文件，试试 linux /boot/vmlinuz）
linux vmlinuz

# 指定临时文件系统（即我们构建的放到boot目录的initrd文件，如果提示找不到文件，试试 linux /boot/initrd.gz）
initrd initrd.gz

# 开始引导
boot
```
