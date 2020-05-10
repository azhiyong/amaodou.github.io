---
title: PXE环境搭建
date: 2020-05-09 11:01:46
tags: Linux
---

## 准备环境

1. VirtualBox 新建虚拟机安装 CentOS 7 系统，启用网卡 1 选择`网络地址转换(NAT)`连接方式、启用网卡 2 选择`仅Host-Only网络`连接方式（ip 192.168.56.105），VirtualBox 网络模式如下：

   | Mode      | VM→Host | VM←Host      | VM1↔VM2 | VM→Net/LAN | VM←Net/LAN   |
   | --------- | ------- | ------------ | ------- | ---------- | ------------ |
   | Host-only | +       | +            | +       | -          | -            |
   | Internal  | -       | -            | +       | -          | -            |
   | Bridged   | +       | +            | +       | +          | +            |
   | NAT       | +       | Port forward | -       | +          | Port forward |

    <!--more-->

2. 下载 CentOS 7 镜像，如 CentOS-7-x86_64-Minimal-1804.iso

3. 关闭防火墙 `systemctl stop firewalld`

## 搭建 PXE 装机环境

### 挂载 CentOS 7 镜像

```bash
mount CentOS-7-x86_64-Minimal-1804.iso /mnt/centos7
```

### 安装配置 httpd 服务

```bash
yum install httpd -y
```

复制 centos7 镜像的所有文件到 httpd 服务目录

```bash
mkdir /var/www/html/centos7
cp -R /mnt/centos7/* /var/www/html/centos7/
```

创建 kiskstart 配置文件

```bash
mkdir /var/www/html/ks

cat > /var/www/html/ks/ks.cfg << EOF
# 安装系统
install

lang en_US.UTF-8
keyboard us

# 系统密码
rootpw --iscrypted $1$AdmTOr23$6kQZu85whleHaSJ.9xXlx.

# 设置网卡
network  --bootproto=dhcp --device=enp0s3 --ipv6=auto --activate
network  --bootproto=dhcp --device=enp0s8 --ipv6=auto --activate
network  --hostname=localhost.localdomain

repo --name=epel --baseurl=http://dl.fedoraproject.org/pub/epel/7/x86_64

# 安装完成后重启
reboot

auth --enableshadow --passalgo=sha512

# 禁用服务
firewall --disabled
selinux --disabled
firstboot --disabled

text
cmdline

timezone Asia/Shanghai
logging --level=debug

# 系统分区
zerombr
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
clearpart --all --initlabel
part / --fstype xfs --asprimary --size 4096 --mkfsoptions='-n ftype=1'
part swap --asprimary --size 1024

%packages
@^minimal
@core

%end
EOF
```

系统密码可以通过 `openssl passwd -1 "PASSWORD"` 生成，替换 `rootpw --iscrypted $1$AdmTOr23$6kQZu85whleHaSJ.9xXlx.`

启动 httpd 服务并设置开机启动

```bash
systemctl start httpd
systemctl enable httpd
```

### 安装配置 tftp 服务

```bash
yum install tftp tftp-server syslinux -y
```

复制引导文件、引导菜单和内核文件到 tftp 服务目录

```bash
cp /usr/share/syslinux/{pxelinux.0,chain.c32,mboot.c32,menu.c32,memdisk} /var/lib/tftpboot
cp /mnt/centos/images/pxeboot/{initrd.img,vmlinuz} /var/lib/tftpboot
```

创建默认 pxe 引导配置

```bash
cat > /var/lib/tftpboot/pxelinux.cfg/default << EOF
default menu.c32
    prompt 5
    timeout 30
    MENU TITLE CentOS 7 PXE Menu

    LABEL linux
    MENU LABEL Install CentOS 7 x86_64
    kernel vmlinuz
    append initrd=initrd.img inst.repo=http://192.168.56.105/centos7 ks=http://192.168.56.105/ks/ks.cfg
EOF
```

启动 tftp 服务并设置开机启动

```bash
systemctl start tftp
systemctl enable tftp
```

### 安装配置 dhcp

```bash
yum install dhcp -y
```

配置 dhcp 服务端如下

```bash
cat > /etc/dhcp/dhcpd.conf << EOF

# tftp服务地址和pxe引导文件名称
next-server 192.168.56.105;
filename "pxelinux.0";

default-lease-time 600;
max-lease-time 1200;
option domain-name "192.168.56.105";
option domain-name-servers 192.168.56.105;

subnet 192.168.56.0 netmask 255.255.255.0 {
    range 192.168.56.120 192.168.56.254;
}

EOF
```

启动 dhcp 服务并设置开机启动

```bash
systemctl start dhcpd
systemctl enable dhcpd
```

## pxe 安装测试

1. 配置主机网络管理器，修改 Host-Only 网络适配器，停用 DHCP 服务；

   ![停用DHCP服务](/images/vbox-dhcp.png)

2. 新建虚拟电脑，内存调整为 2048MB（内存太小会安装失败），调整启动顺序（优先网络启动，不用挂载镜像），启用网卡 1 选择`仅Host-Only网络`连接方式

   ![调整启动顺序](/images/boot-order.png)
   ![设置网卡](/images/vbox-network.png)
   ![加载pxelinux文件](/images/load-pxelinux.png)
   ![菜单项](/images/install-menu.png)
