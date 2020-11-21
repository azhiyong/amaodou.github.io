---
title: PXE实现自动装机
date: 2020-05-09 11:01:46
tags: Linux
---

什么是 PXE？

- PXE，全名 Pre-boot Execution Environment，预启动执行环境；

- 通过网络接口启动计算机，不依赖本地存储设备（如硬盘）或本地已安装的操作系统；

- 由 Intel 和 Systemsoft 公司于 1999 年 9 月 20 日公布的技术；

PXE 的工作过程

![PXE工作流程](/images/linux/pxe工作流程.png)

## 准备环境

1. VirtualBox 新建虚拟机安装 CentOS 7 系统，用于搭建 PXE 服务器（这里我的 PXE 服务器 ip 是 192.168.56.101），其中：

   网卡 1 选择`网络地址转换（NAT）`连接方式：虚拟机能访问主机和公网，主机需要通过端口转发的方式才能访问虚拟机；

   网卡 2 选择`仅主机（Host-Only）网络`连接方式：主机和虚拟机能互相访问；

   VirtualBox 网络模式如下：

   | Mode      | VM→Host | VM←Host      | VM1↔VM2 | VM→Net/LAN | VM←Net/LAN   |
   | --------- | ------- | ------------ | ------- | ---------- | ------------ |
   | Host-only | +       | +            | +       | -          | -            |
   | Internal  | -       | -            | +       | -          | -            |
   | Bridged   | +       | +            | +       | +          | +            |
   | NAT       | +       | Port forward | -       | +          | Port forward |

    <!--more-->

2. 准备客户端装机镜像，CentOS/Ubuntu 都可以，比如 CentOS-7-x86_64-Minimal-1804.iso

## 设置PXE引导服务器

### 配置 tftp 服务

1. 安装 tftp

    ```bash
    yum install tftp-server -y
    ```

2. 配置 tftp，默认情况下 tftp 服务是禁用的，修改 disable 为 no

    ```bash
    [root@localhost ~]# cat /etc/xinetd.d/tftp
    # default: off
    # description: The tftp server serves files using the trivial file transfer \
    #       protocol.  The tftp protocol is often used to boot diskless \
    #       workstations, download configuration files to network-aware printers, \
    #       and to start the installation process for some operating systems.
    service tftp
    {
            socket_type             = dgram
            protocol                = udp
            wait                    = yes
            user                    = root
            server                  = /usr/sbin/in.tftpd
            server_args             = -s /var/lib/tftpboot  #指定tftp服务的根目录
            disable                 = no #是否禁用
            per_source              = 11
            cps                     = 100 2
            flags                   = IPv4
    }
    ```

3. 安装 syslinux-tftpboot，安装完成后会在 tftp 目录`/var/lib/tftpboot`下生成引导加载程序文件（安装 syslinux 也可以）

    ```bash
    yum install syslinux-tftpboot -y
    ```

    syslinux 是一个功能强大的引导加载程序，而且兼容各种介质，syslinux 是一个小型的 Linux 操作系统，它的目的是简化首次安装 linux 的时间，并建立修护或其它特殊用途的启动盘

    安装 syslinux 的话，在安装完成之后，需要将`/usr/share/syslinux`目录下的文件`pxelinux.0,chain.c32,mboot.c32,menu.c32,memdisk`拷贝到 tftp 服务目录

4. 启动 tftp

    ```bash
    systemctl start tftp
    systemctl enable tftp #设置开机启动
    ```

5. 验证 tftp 服务，需要关闭防火墙`systemctl stop firewalld`

    ```bash
    [root@localhost ~]# tftp 127.0.0.1
    tftp> get pxelinux.0
    tftp> quit
    [root@localhost ~]# ll
    -rw-r--r--. 1 root root 26759 Jul 21 10:16 pxelinux.0
    ```

### 复制镜像安装文件到 PXE 服务器

1. 挂载需要在客户端安装的系统镜像，比如 CentOS-7-x86_64-Minimal-1804.iso

    ```bash
    # 挂载centos镜像
    mount CentOS-7-x86_64-Minimal-1804.iso /mnt/centos7

    # 挂载ubuntu镜像
    mount ubuntu-16.04.6-server-amd64.iso /mnt/ubuntu16.04
    ```

2. 从挂载镜像目录中将 vmlinuz 和 initrd.img 文件复制到 tftp 服务目录

   为了支持自动安装多种系统，可以在 tftp 服务器目录下创建各个系统目录用于存放各自的内核文件和临时根文件系统

   - CentOS 7

     ```bash
     mkdir /var/lib/tftpboot/centos7
     cp /mnt/centos7/images/pxeboot/{initrd.img,vmlinuz} /var/lib/tftpboot/centos7
     ```

   - Ubuntu 16.04

     ```bash
     mkdir /var/lib/tftpboot/ubuntu16.04
     cp /mnt/ubuntu16.04/install/netboot/ubuntu-installer/amd64/{initrd.gz,linux} /var/lib/tftpboot/ubuntu16.04
     ```

3. 安装 httpd 服务，复制挂载镜像目录下的所有文件到 http 服务目录（默认是/var/www/html）

    ```bash
    yum install httpd -y

    ## centos镜像文件
    mkdir /var/www/html/centos7
    cp -R /mnt/centos7/* /var/www/html/centos7/

    # ubuntu镜像文件
    mkdir /var/www/html/ubuntu16.04
    cp -R /mnt/ubuntu16.04/* /var/www/html/ubuntu16.04/
    ```

4. 创建 kiskstart 配置

    可以通过 system-config-kickstart 在桌面环境生成 ks.cfg，该文件包含了安装过程需要配置的参数信息

    ```bash
    [root@localhost ~]# cat /var/www/html/centos7/ks.cfg
    #安装系统
    install

    #Use Web installation
    url --url="http://192.168.56.101/centos7"

    #以文本模式执行安装
    text

    lang en_US.UTF-8
    keyboard us

    #Root password
    rootpw --iscrypted $1$AdmTOr23$6kQZu85whleHaSJ.9xXlx.

    #Network information
    network --bootproto=dhcp --device=eth0

    #安装完成后关机
    poweroff

    #System authorization infomation
    auth --enableshadow --passalgo=sha512

    #Firewall configuration
    firewall --disabled

    timezone Asia/Shanghai
    logging --level=debug

    #Clear the Master Boot Record
    zerombr yes

    #System bootloader configuration，location指定引导记录写入位置（mbr）
    bootloader --location=mbr

    #Partition clearing information
    clearpart --all --initlabel

    #boot分区200M
    part /boot --fstype ext2 --size 400
    #swap分区1024M
    part swap --size 1024
    #根分区，grow使用剩余可用空间
    part / --fstype ext4 --size 1 --grow

    %packages
    @^minimal
    @core
    %end
    ```

    系统密码可以通过 `openssl passwd -1 "PASSWORD"` 生成，替换 `rootpw --iscrypted $1$AdmTOr23$6kQZu85whleHaSJ.9xXlx.`

5. 创建 /var/lib/tftpboot/pxelinux.cfg/default

    ```bash
    [root@localhost ~]# cat /var/lib/tftpboot/pxelinux.cfg/default
    default menu.c32
        timeout 30
        menu title PXE install CentOS or Ubuntu

    label centos7
        menu label ^Install CentOS 7
        kernel centos7/vmlinuz
        append initrd=centos7/initrd.img ks=http://192.168.56.101/centos7/ks.cfg
    label ubuntu16
        menu label ^Install Ubuntu 16.04
        kernel ubuntu16.04/linux
        append initrd=ubuntu16.04/initrd.gz ks=http://192.168.56.101/ubuntu16.04/ks.cfg
    ```

6. 启动 http 服务

    ```bash
    systemctl start httpd
    systemctl enable httpd #开机启动http服务
    ```

### 安装 dhcp 服务

1. 安装 dhcp

    ```bash
    yum install dhcp -y
    ```

2. 配置 dhcp

    ```bash
    [root@localhost ~]# cat /etc/dhcp/dhcpd.conf

    subnet 192.168.56.0 netmask 255.255.255.0 {
        range 192.168.56.2 192.168.56.100;
        #option domain-name "pxe-server.com";
        #option domain-name-servers 192.168.56.101;
        option routers 192.168.56.1;
        option broadcast-address 192.168.56.255;
        default-lease-time 600;
        max-lease-time 7200;
        next-server 192.168.56.101;
        filename "pxelinux.0";
    }
    ```

3. 启动 dhcp

    ```bash
    systemctl start dhcpd
    systemctl enable dhcpd #开机启动
    ```

## pxe 客户端安装测试

1. 在 virtualBox 网络管理器中禁用 DHCP 服务，这样客户端虚拟机才会请求我们搭建好的 dhcp 服务器

   ![禁用dhcp](/images/linux/禁用vbox的dhcp服务.png)

2. 新建客户端虚拟机，内存调整为 2048MB（内存太小会安装失败）

   调整启动顺序（网络启动，不用挂载镜像）

   ![配置启动顺序](/images/linux/pxe%20client配置启动顺序.png)

   启用网卡 1 选择`仅Host-Only网络`连接方式

   ![配置网络](/images/linux/pxe%20client配置网络.png)

   客户端向 tftp 服务器请求 pxelinux.0 文件

   ![请求pxelinux.0](/images/linux/pxe%20client请求pxelinux0.png)

   客户端出现引导菜单

   ![引导菜单](/images/linux/pxe%20client引导菜单.png)

## 参考资料

- No space left on device: https://access.redhat.com/solutions/3293511
