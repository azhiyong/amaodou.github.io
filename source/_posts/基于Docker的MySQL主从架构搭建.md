---
title: 基于Docker的MySQL主从架构搭建
tags: MySQL
date: 2021-01-02 12:28:19
---


最近要在本地搭建一套 MySQL 主从做分库分表和读写分离，由于本地资源有限不能做到每个 MySQL 实例独占一台虚拟机，所以考虑使用 Docker 在单台虚拟机上部署多个 MySQL 实例。

整体部署结构 2 台虚拟机，每台虚拟机上部署 1 主 2 从，如下

| ip            | 说明                                         |
| ------------- | -------------------------------------------- |
| 192.168.56.10 | 主节点 3306 端口，两个从节点 3307、3308 端口 |
| 192.168.56.11 | 主节点 3306 端口，两个从节点 3307、3308 端口 |

![MySQL主从部署](/images/mysql/MySQL分库主从部署.png)

<!--more-->

### 安装 Docker

> Docker 官方安装文档：https://docs.docker.com/engine/install/centos/

#### 卸载旧版本的 Docker

```bash
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

#### 配置 Docker 仓库

yum-utils 提供了 yum-config-manager 工具，这里我们使用 aliyun 的仓库

官方源 https://download.docker.com/linux/centos/docker-ce.repo

```bash
$ sudo yum install -y yum-utils

$ sudo yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

#### 安装 Docker

- 安装最新版本的 Docker（也可以安装指定版本）

  ```bash
  $ sudo yum install docker-ce docker-ce-cli containerd.io
  ```

- 安装指定版本的 Docker

  查看 Docker 版本列表

  ```bash
  $ yum list docker-ce --showduplicates | sort -r

  docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable
  docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable
  docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable
  docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable
  ```

  指定版本号，比如 3:18.09.1-3.el7 对应版本号 18.09.1

  ```bash
  $ sudo yum install docker-ce-18.09.1 docker-ce-cli-18.09.1 containerd.io
  ```

#### 启动 Docker

```bash
$ sudo systemctl start docker
```

#### 验证是否安装成功

```bash
$ sudo docker run hello-world
```

### Docker 安装 MySQL 实例

#### 拉取 MySQL 镜像

```bash
# 查找 MySQL 镜像
$ docker search mysql

# 拉取指定版本的 MySQL 镜像，不加版本号则表示拉取最新的镜像
# docker pull mysql
$ docker pull mysql:5.7
```

#### 启动 MySQL 容器

```bash
$ docker run -it --name mysql_3306 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root mysql:5.7
```

run 表示启动一个新的容器
-it 分配一个虚拟的终端
--name 表示新容器的名称，后面可以使用这个名称来操作容器
-p 3306:3306 表示将本地 3306 端口映射到容器的 3306 端口
-e MYSQL_ROOT_PASSWORD=root 设置环境变量，此处是设置 root 密码
mysql:5.7 指定使用的镜像和版本（不指定默认使用最新的版本）

#### 配置容器内的 MySQL 实例

```bash
# 进入容器
$ docker exec -it mysql_3306 /bin/bash

# docker中还没有安装vim工具，需要安装
$ apt-get update
$ apt-get install vim -y
```

修改 MySQL 配置文件

```bash
$ cat /etc/mysql/my.cnf

# 此处省略注释
# ...

!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/

[mysqld]
# 指定binlog文件名称
log-bin=mysql-bin
# 指定实例id，不能重复
server-id=1
sync_binlog=1
binlog-ignore-db=information_schema
binlog-ignore-db=performance_schema
binlog-ignore-db=mysql
binlog-ignore-db=sys

relay-log-purge=1
relay_log=mysql-relay-bin
```

配置完成后重启容器

```bash
$ docker restart mysql_3306

# 查看容器进程
$ docker ps

# 查看容器信息，包括容器的ip和文件目录
$ docker inspect docker_3306
```

同上安装 mysql_3307、mysql_3308 从节点，只需要修改端口号和配置中的 server-id。

### 配置主从

主节点 mysql_3306 配置主从复制账号密码，此处直接使用 root

```bash
mysql> grant all privileges on *.* to root@'%' identified by 'root';
mysql> flush privileges;

# 查看master节点状态
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000006
         Position: 154
     Binlog_Do_DB:
 Binlog_Ignore_DB: information_schema,performance_schema,mysql,sys
Executed_Gtid_Set:
1 row in set (0.00 sec)
```

从节点 mysql_3307、mysql_3308 切换主节点配置

```bash
mysql> CHANGE MASTER TO
  MASTER_HOST='192.168.56.10',
  MASTER_USER='root',
  MASTER_PASSWORD='root',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='mysql-bin.000006',
  MASTER_LOG_POS=154,
  MASTER_CONNECT_RETRY=10;

# 开始复制
mysql> start slave;
```

完。
