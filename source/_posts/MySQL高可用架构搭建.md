---
layout: layout
title: MySQL高可用架构搭建
date: 2020-11-21 11:05:31
tags: MySQL
---

## 准备工作

3 台 mysql 服务器 和 1 台 mha 服务器

| 角色        | ip           | 说明         |
| ----------- | ------------ | ------------ |
| mha manager | 192.168.56.3 | mha 管理节点 |
| master      | 192.168.56.5 | mysql 主库   |
| slave       | 192.168.56.6 | mysql 从库   |
| slave       | 192.168.56.7 | mysql 从库   |

mysql 服务器和 mha 服务器全部关闭防火墙

```bash
# 关闭防火墙
systemctl stop firewalld

# 取消开机启动
systemctl disable firewalld
```

<!--more-->

## mysql 主从搭建

3 台 mysql 服务器安装 mysql

### 下载

```bash
## 下载
wegt https://cdn.mysql.com/archives/mysql-5.7/mysql-5.7.28-1.el7.x86_64.rpm- bundle.tar

## 解压
tar -xvf mysql-5.7.28-1.el7.x86_64.rpm-bundle.tar

# mysql-community-embedded-5.7.28-1.el7.x86_64.rpm
# mysql-community-libs-compat-5.7.28-1.el7.x86_64.rpm
# mysql-community-devel-5.7.28-1.el7.x86_64.rpm
# mysql-community-embedded-compat-5.7.28-1.el7.x86_64.rpm
# mysql-community-libs-5.7.28-1.el7.x86_64.rpm
# mysql-community-test-5.7.28-1.el7.x86_64.rpm
# mysql-community-common-5.7.28-1.el7.x86_64.rpm
# mysql-community-embedded-devel-5.7.28-1.el7.x86_64.rpm
# mysql-community-client-5.7.28-1.el7.x86_64.rpm
# mysql-community-server-5.7.28-1.el7.x86_64.rpm
```

### 移除 mariadb

```bash
rpm -qa | grep mariadb

# mariadb-libs-5.5.56-2.el7.x86_64

# 移除（--nodeps 不验证软件包依赖）
rpm -e mariadb-libs-5.5.56-2.el7.x86_64 --nodeps
```

### 安装 mysql

按需按顺序安装

```bash
rpm -ivh mysql-community-common-5.7.28-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.28-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-compat-5.7.28-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.28-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.28-1.el7.x86_64.rpm
```

### 初始化数据库

```bash
# 查看mysqld命令选项
mysqld --verbose --help

# --initialize        Create the default database and exit. Create a super user
#                     with a random expired password and store it into the log.

# 初始化数据库
mysqld --initialize --user=mysql

# 从日志中获取mysql密码
less /var/log/mysqld.log

# 2020-12-06T01:49:03.177037Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
# 2020-12-06T01:49:03.749858Z 0 [Warning] InnoDB: New log files created, LSN=45790
# 2020-12-06T01:49:03.978416Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
# 2020-12-06T01:49:04.156867Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: 3859d6df-3765-11eb-906f-0800279d8ede.
# 2020-12-06T01:49:04.218652Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
# 2020-12-06T01:49:04.711667Z 0 [Warning] CA certificate ca.pem is self signed.
# 2020-12-06T01:49:04.921715Z 1 [Note] A temporary password is generated for root@localhost: xNB)/RbxI3LT

# 启动mysql服务
systemctl start mysqld

# 将mysql服务设置为开机启动
systemctl enable mysqld

# 登录
mysql -h127.0.0.1 -uroot -p'xNB)/RbxI3LT'

# 重置mysql密码（help set password 命令说明）
mysql> set password=password('root');
```

### 主从同步配置

#### master 配置

mysql 配置文件 `/etc/my.cnf`

```bash
# 服务器id，不能重复
server-id=1

# bin_log配置
log_bin=mysql-bin
sync-binlog=1
binlog-ignore-db=information_schema
binlog-ignore-db=mysql
binlog-ignore-db=performance_schema
binlog-ignore-db=sys

# relay_log配置
relay_log=mysql-relay-bin
log_slave_updates=1
relay_log_purge=0
```

修改配置后重启 mysql 服务

```bash
systemctl restart mysqld
```

配置授权

```bash
mysql> grant replication slave on *.* to root@'%' identified by 'root';
mysql> grant all privileges on *.* to root@'%' identified by 'root';

# 重新加载授权
mysql> flush privileges;

mysql> show master status;
+------------------+----------+--------------+-------------------------------------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB                                | Executed_Gtid_Set |
+------------------+----------+--------------+-------------------------------------------------+-------------------+
| mysql-bin.000001 |      869 |              | information_schema,mysql,performance_schema,sys |                   |
+------------------+----------+--------------+-------------------------------------------------+-------------------+
```

#### slave 配置

mysql 配置文件 `/etc/my.cnf`

```bash
# 服务器id，不能重复
server-id=2

# bin_log配置
log_bin=mysql-bin
sync-binlog=1
binlog-ignore-db=information_schema
binlog-ignore-db=mysql
binlog-ignore-db=performance_schema
binlog-ignore-db=sys

# relay_log配置
relay_log=mysql-relay-bin
log_slave_updates=1
relay_log_purge=0

read_only=1
```

slave 开启同步

```bash
# 查看命令说明
mysql> help change master to;

mysql> CHANGE MASTER TO
  MASTER_HOST='192.168.56.5',
  MASTER_USER='root',
  MASTER_PASSWORD='root',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=869;

# 开启同步
mysql> start slave;
```

### 半同步复制

#### master 安装插件

```bash
mysql> install plugin rpl_semi_sync_master soname 'semisync_master.so';

mysql> show variables like '%semi%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | OFF        |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
+-------------------------------------------+------------+
```

修改配置文件`/etc/my.cnf`，添加配置

```bash
# 开启半同步复制
rpl_semi_sync_master_enabled=ON
rpl_semi_sync_master_timeout=1000
```

重启 mysql 服务

```bash
systemctl restart mysqld
```

#### slave 安装插件

```bash
mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
```

修改配置文件`/etc/my.cnf`，添加配置

```bash
# 自动开启半同步复制
rpl_semi_sync_slave_enabled=ON
```

重启 mysql 服务

```bash
systemctl restart mysqld
```

#### 检查半同步复制是否成功

```bash
less /var/log/mysqld.log

# 2020-12-06T02:49:04.711667Z 0 [Note] Slave I/O thread: Start semi-sync replication to master 'root@192.168.56.5:3306' in log 'mysql-bin.000001' at position 869
```

## MHA 高可用搭建

### ssh 互通

生成 ssh 秘钥对

```bash
# 生成秘钥对
ssh-keygen

# 查看公钥
cat ~/.ssh/id_rsa.pub
# ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDk6r4vFUA35M8FrxoCq455dTYRV7Lg98MklLJ3P71i3RS0i5vsZGuRB0VbQK6TcThq0gooVMuJy17P8lgXUlc30W+GGg8gqz7yQ1Y7+aktg2iuY45eETFjB+qhzWUtEQsWAJaFEPGrYCSsbE3XCj2JA1w11zIc+PxpKFUzX+JK7MSXIerSv94Q4PGTEeudNc/fHHSothFCAvxF/BdCiZvttPXUJkhuI1RbX5RF46SPfcTOvRn76rCHyntPiE0ZY9eLLtCkAk48XTRV37kxbnoQmvLg7UhNXm1a14q3ZDKCP49YQ52cTihTTBLyPwCwu+HUHQuRLUoXDWj1GBVB6t5Z root@localhost.localdomain
```

每个节点的公钥都要拷贝到另外 3 个节点（4 台服务器两两互通）

```bash
ssh root@192.168.56.5 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys' < ~/.ssh/id_rsa.pub
ssh root@192.168.56.6 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys' < ~/.ssh/id_rsa.pub
ssh root@192.168.56.7 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```

### MHA 安装

获取 mha 安装包，mha 分为管理节点和 mysql 节点（ 管理节点需要安装 mha4mysql-manager 和 mha4mysql-node，mysql 节点安装 mha4mysql-node）

```bash
wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
```

安装 mha4mysql-node

```bash
# 安装mha4mysql-node依赖
yum install perl-DBD-MySQL -y

rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
```

mha 管理节点安装 mha4mysql-manager 和 mha4mysql-node

```bash
# perl-Log-Dispatch perl-Parallel-ForkManager在yum仓库找不到
wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

# 安装依赖
yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel- ForkManager -y

rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
```

### MHA 配置

MHA Manager 服务器需要为每个监控的 Master/Slave 集群提供一个专用的配置文件，所有集群也可以共享全局配置。

初始化目录

```bash
mkdir -p /var/log/mha/app1
touch /var/log/mha/app1/manager.log
```

全局配置文件 `/etc/masterha_default.cnf`

```bash
[server default]
user=root
password=root
# ssh登录账号
ssh_user=root
# 主从复制账号
repl_user=root
# 主从复制密码
repl_password=root
# ping次数
ping_interval=1
# 二次检查的主机
secondary_check_script=masterha_secondary_check -s 192.168.56.5 -s 192.168.56.6 -s 192.168.56.7
```

mysql 集群监控配置`/etc/mha/app1.cnf`

```bash
[server default]
# MHA监控实例根目录
manager_workdir=/var/log/mha/app1
# MHA监控实例日志文件
manager_log=/var/log/mha/app1/manager.log

[server1]
hostname=192.168.56.5
candidate_master=1
master_binlog_dir="/var/lib/mysql"

[server2]
hostname=192.168.56.6
candidate_master=1
master_binlog_dir="/var/lib/mysql"

[server3]
hostname=192.168.56.7
candidate_master=1
master_binlog_dir="/var/lib/mysql"
```

检测配置文件

```bash
# ssh通信检测
masterha_check_ssh --conf=/etc/mha/app1.cnf

# 主从同步检测
masterha_check_repl --conf=/etc/mha/app1.cnf
```

### 启动 MHA manager

```bash
nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf -- ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &

# 查看mha manager状态
masterha_check_status --conf=/etc/mha/app1.cnf

# 查看日志

tail -f /var/log/mha/app1/manager.log
```

### mysql master 故障测试

master 节点执行

```bash
systemctl stop mysqld
```

mha manager 日志

```bash
# Tue Dec  1 11:58:33 2020 - [info] New master is 192.168.56.6(192.168.56.6:3306)
# Tue Dec  1 11:58:33 2020 - [info] Starting master failover..
# Tue Dec  1 11:58:33 2020 - [info]
# From:
# 192.168.56.7(192.168.56.7:3306) (current master)
#  +--192.168.56.6(192.168.56.6:3306)
#  +--192.168.56.5(192.168.56.5:3306)

# To:
# 192.168.56.6(192.168.56.6:3306) (new master)
#  +--192.168.56.5(192.168.56.5:3306)
```
