---
title: Dynomite快速入门
date: 2020-05-09 11:02:40
tags:
---
### Dynomite 是什么

Dynomite 是受[Amazon Dynamo][1]白皮书启发，适用于不同存储引擎和协议的分布式 Dynamo 层的实现，目前已经支持`Redis`和`Memcached`。Dynomite 支持多数据中心复制，为高可用而设计的。

Dynomite 的最终目标是能够在本身不提供高可用和多数据中心复制功能的存储引擎上实现这些功能，并且做到高效、简单、高性能。简单来说就是，Dynomite 可以使非分布式的`Redis`和`Memcached`变成分布式的。

### Dynomite 怎么用

既然 Dynomite 可以简单理解为对 Redis 和 Memcached 的包装，那我们在代码中可以像使用 Redis 或 Memcached 一样使用 Dynomite

#### 环境搭建

> 以`Redis`作为存储引擎，创建一个单 DC，2 个 Rack，每个 Rack 2 个节点的存储环境，如下：

![Dynomite拓补图](/images/dynomite-topology.png)

#### 安装配置 Redis

1. 安装 Redis

   ```bash
   cd ~
   wget http://download.redis.io/releases/redis-5.0.5.tar.gz
   tar -xvf redis-5.0.5.tar.gz
   cd redis-5.0.5
   make install
   ```

2. 配置启动 4 个 Redis 节点

   ```bash
   export REDIS_APP=~/apps/redis
   mkdir -p $REDIS_APP/{conf,logs}
   cat > $REDIS_APP/conf/redis-{6396,6397,6398,6399}.conf < ~/redis-5.0.5/redis.conf

   for conf in $REDIS_APP/conf/redis-*.conf
   do
       port=$(echo $(basename $conf) | sed 's/[^0-9]//g')
       sed -i "s/6379/$port/g" $conf
       sed -i "s:^logfile .*:logfile $REDIS_APP/logs/$port\.log:g" $conf

       redis-server $conf &
   done
   unset REDIS_APP
   ```

#### 安装配置 Dynomite

1. 源码安装 Dynomite

   ```bash
   apt-get install git autoconf automake libtool openssl-devel -y
   git clone git@github.com:Netflix/dynomite.git
   cd dynomite
   autoreconf -fvi
   ./configure --enable-debug=yes  //启用debug日志
   make
   make install
   ```

2. 创建配置文件存放的目录

   ```bash
   export DYNOMITE_APP=~/apps/dynomite
   mkdir -p $DYNOMITE_APP/{conf,logs}
   ```

   配置 Rack1 的第一个节点 8102，配置如下：

   ```bash
   cat > $DYNOMITE_APP/conf/node-8102.yml << EOF
   dyn_o_mite:
       datacenter: dc1
       rack: rack1
       listen: 0.0.0.0:8102
       dyn_listen: 0.0.0.0:8101
       dyn_seeds:
           - 127.0.0.1:8201:rack1:dc1:2147483647
           - 127.0.0.1:8301:rack2:dc1:0
           - 127.0.0.1:8401:rack2:dc1:2147483647
       dyn_seed_provider: simple_provider
       tokens: '0'
       servers:
           - 127.0.0.1:6396:1
       data_store: 0
       stats_listen: 127.0.0.1:33331
       preconnect: true
   EOF
   ```

   配置 Rack1 的第二个节点 8202，配置如下：

   ```bash
   cat > $DYNOMITE_APP/conf/node-8202.yml << EOF
   dyn_o_mite:
       datacenter: dc1
       rack: rack1
       listen: 0.0.0.0:8202
       dyn_listen: 0.0.0.0:8201
       dyn_seeds:
           - 127.0.0.1:8101:rack1:dc1:0
           - 127.0.0.1:8301:rack2:dc1:0
           - 127.0.0.1:8401:rack2:dc1:2147483647
       dyn_seed_provider: simple_provider
       tokens: '2147483647'
       servers:
           - 127.0.0.1:6397:1
       data_store: 0
       stats_listen: 127.0.0.1:33332
       preconnect: true
   EOF
   ```

   配置 Rack2 的第一个节点 8302，配置如下：

   ```bash
   cat > $DYNOMITE_APP/conf/node-8302.yml << EOF
   dyn_o_mite:
       datacenter: dc1
       rack: rack2
       listen: 0.0.0.0:8302
       dyn_listen: 0.0.0.0:8301
       dyn_seeds:
           - 127.0.0.1:8201:rack1:dc1:2147483647
           - 127.0.0.1:8101:rack1:dc1:0
           - 127.0.0.1:8401:rack2:dc1:2147483647
       dyn_seed_provider: simple_provider
       tokens: '0'
       servers:
           - 127.0.0.1:6398:1
       data_store: 0
       stats_listen: 127.0.0.1:33333
       preconnect: true
   EOF
   ```

   配置 Rack2 的第二个节点 8402，配置如下：

   ```bash
   cat > $DYNOMITE_APP/conf/node-8402.yml << EOF
   dyn_o_mite:
       datacenter: dc1
       rack: rack2
       listen: 0.0.0.0:8402
       dyn_listen: 0.0.0.0:8401
       dyn_seeds:
           - 127.0.0.1:8201:rack1:dc1:2147483647
           - 127.0.0.1:8101:rack1:dc1:0
           - 127.0.0.1:8301:rack2:dc1:0
       dyn_seed_provider: simple_provider
       tokens: '2147483647'
       servers:
           - 127.0.0.1:6399:1
       data_store: 0
       stats_listen: 127.0.0.1:33334
       preconnect: true
   EOF
   ```

   启动 Dynomite 节点

   ```bash
   for conf in $DYNOMITE_APP/conf/node-*.yml
   do
       port=$(echo $(basename $conf) | sed 's/[^0-9]//g')
       dynomite -c $conf -o $DYNOMITE_APP/logs/$port.log -p $DYNOMITE_APP/logs/$port.pid -d
   done
   unset DYNOMITE_APP
   ```

#### 配置描述

- datacenter 数据中心名称
- rack 机架名称，每个机架都包含一份完整的集群数据
- listen 外部访问节点的端口
- dyn_listen 集群内部通信使用的端口
- dyn_seeds 集群中的其他节点，格式 address:port:rack:dc:tokens
- dyn_seed_provider 提供 seed 节点列表实现
- tokens 用于计算每个节点管理的数据范围，每个节点只有一个 token
- servers 存储引擎服务（如 Redis/Memcached）地址，格式 ip:port:weight，每个节点只有一个
- data_store 指定使用的存储引擎 redis(0)或者 memcached(1)
- stats_listen 节点数据统计
- preconnect 是否在启动前连接所有 servers

### Dynomite 测试

```bash
$ redis-cli -h 127.0.0.1 -p 8102
127.0.0.1:8102> set test1 test1
OK
127.0.0.1:8102> set test1111 test1111
OK
127.0.0.1:8102> set test2 test2
OK
127.0.0.1:8102> set test2222 test2222
OK
127.0.0.1:8102> keys *
1) "test2"
2) "test2222"
127.0.0.1:8102> get test1
"test1"
127.0.0.1:8102> get test2
"test2"

$ redis-cli -h 127.0.0.1 -p 6396
127.0.0.1:6396> keys *
1) "test2"
2) "test2222"

$ redis-cli -h 127.0.0.1 -p 6398
127.0.0.1:6398> keys *
1) "test2"
2) "test2222"
```

### 注意事项

- Dynomite 集群由一个或多个 datacenter 组成，每个 datacenter 包含一个或多个 rack，每个 rack 包含一个或多个节点。每个 rack 上的节点数量可以不相同，同理，每个 datacenter 上的 rack 数量也可以不相同；
- 在 Dynomite 集群中，每个 rack 都存储了一份完整的集群数据；
- Rack 中每个节点存储的范围是从当前节点的 token 到下一个节点的 token - 1，通过一致性 Hash 算法计算 Key 的 token，将数据存储到正确的节点上；
- 假如 Redis 节点在加入 Dynomite 集群之前已经存了一部分数据，加入集群后通过 Dynomite 可能读取不到其中一些数据，因为这些数据计算出来的 token 不在节点的管理范围内；

[1]: http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
