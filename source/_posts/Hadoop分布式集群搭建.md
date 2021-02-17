---
title: Hadoop分布式集群搭建
tags: 
- Linux
- Hadoop
date: 2021-02-17 16:55:03
---

## Hadoop 是什么

Hadoop 是一个适合大数据的分布式存储和计算平台，用来解决大海量数据的储存和计算问题。

狭义上的 Hadoop 指的是一个框架，包含分布式文件系统 HDFS、分布式计算引擎 MapReducer、资源调度框架 Yarn;

广义上的 Hadoop 指的是一个生态圈，除了包含 Hadoop 框架本身，还包含一些辅助框架（日志数据采集工具 Flume、关系型数据库数据采集工具 sqoop、海量列式非关系型数据库 HBase、数据仓库工具 Hive）;

<!-- more -->

## Hadoop 分布式集群搭建

### 集群规划

| 虚拟机 ip     | hostname | HDFS 角色                   | Yarn 角色                    |
| ------------- | -------- | --------------------------- | ---------------------------- |
| 192.168.56.10 | mdou10   | NameNode, DataNode          | NodeManager                  |
| 192.168.56.11 | mdou11   | SecondaryNameNode, DataNode | ResourceManager, NodeManager |
| 192.168.56.20 | mdou20   | DataNode                    | NodeManager                  |

### 环境准备

准备 3 台虚拟机（关闭防火墙、配置 hostname、配置 ssh 免密登录）

1. 关闭防火墙

   ```bash
   [root@mdou10 ~] systemctl stop firewalld
   [root@mdou10 ~] systemctl disable firewalld
   ```

2. 配置 hostname

   ```bash
   # 设置hostname
   [root@mdou10 ~] hostnamectl set-hostname mdou10

   # 配置host
   [root@mdou10 ~] cat > /etc/hosts << EOF
   192.168.56.10 mdou10
   192.168.56.11 mdou11
   192.168.56.20 mdou20
   EOF
   ```

3. 配置 ssh 免密登录

   ```bash
   # 生成ssh秘钥对
   [root@mdou10 ~] ssh-keygen

   # 将ssh公钥拷贝到authorized_keys文件
   [root@mdou10 ~] cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

   # 配置ssh免密登录
   [root@mdou10 ~] ssh 192.168.56.11 'cat >> ~/.ssh/authorized_keys' < ~/.ssh/id_rsa.pub
   ```

### 配置 JAVA_HOME

```bash
# /etc/profile 添加配置
export JAVA_HOME=/usr/local/jdk1.8.0_151
export PATH=$JAVA_HOME/bin:$PATH
```

### 配置 Hadoop

1. core-site.xml

   ```xml
   <configuration>
       <!-- 指定hdfs的NameNode -->
       <property>
           <name>fs.defaultFS</name>
           <value>hdfs://mdou10:9000</value>
       </property>
       <!-- 指定Hadoop运行时产生文件的存储目录 -->
       <property>
           <name>hadoop.tmp.dir</name>
           <value>/usr/local/hadoop-2.9.2/data/tmp</value>
       </property>
   </configuration>
   ```

2. hdfs-site.xml

   ```xml
   <configuration>
       <!-- 指定hdfs的secondaryNameNode -->
       <property>
           <name>dfs.namenode.secondary.http-address</name>
           <value>mdou11:50090</value>
       </property>
       <!--副本数量 -->
       <property>
           <name>dfs.replication</name>
           <value>3</value>
       </property>
   </configuration>
   ```

3. slaves

   这里使用 hostname

   ```bash
   [root@mdou10 hadoop] cat slaves
   mdou10
   mdou11
   mdou20
   ```

4. mapred-site.xml

   ```xml
   <configuration>
       <!-- 指定MR运行在Yarn上 -->
       <property>
           <name>mapreduce.framework.name</name>
           <value>yarn</value>
       </property>
       <!-- 历史任务服务器端地址 -->
       <property>
           <name>mapreduce.jobhistory.address</name>
           <value>mdou11:10020</value>
       </property>
       <!-- 历史任务服务器web端地址 -->
       <property>
           <name>mapreduce.jobhistory.webapp.address</name>
           <value>mdou11:19888</value>
       </property>
   </configuration>
   ```

5. yarn-site.xml

   ```xml
   <configuration>
       <!-- 配置ResourceManager节点 -->
       <property>
           <name>yarn.resourcemanager.hostname</name>
           <value>mdou11</value>
       </property>
       <!-- Reducer获取数据的方式 -->
       <property>
           <name>yarn.nodemanager.aux-services</name>
           <value>mapreduce_shuffle</value>
       </property>
       <!-- 日志聚集功能使能 -->
       <property>
           <name>yarn.log-aggregation-enable</name>
           <value>true</value>
       </property>
       <!-- 日志保留时间设置7天 -->
       <property>
           <name>yarn.log-aggregation.retain-seconds</name>
           <value>604800</value>
       </property>
       <!-- 是否检查磁盘容量 -->
       <property>
           <name>yarn.nodemanager.pmem-check-enabled</name>
           <value>false</value>
       </property>
       <!-- 是否检查内存容量 -->
       <property>
           <name>yarn.nodemanager.vmem-check-enabled</name>
           <value>false</value>
       </property>
   </configuration>
   ```

### Hadoop 配置同步到其他虚拟机

```bash
[root@mdou10 hadoop-2.9.2] rsync -rvl /usr/local/hadoop-2.9.2 root@192.168.56.11:/usr/local
[root@mdou10 hadoop-2.9.2] rsync -rvl /usr/local/hadoop-2.9.2 root@192.168.56.20:/usr/local
```

### 启动集群

```bash
# 启动HDFS，在NameNode节点执行；使用jps查看是否启动成功
[root@mdou10 hadoop-2.9.2] start-dfs.sh

# 启动Yarn，在ResourceManager节点执行
[root@mdou11 ~] start-yarn.sh

# 启动jobhistoryserver，在ResourceManager节点执行
[root@mdou11 ~] mr-jobhistory-daemon.sh start historyserver
```

Hdfs web 页面 http://mdou10:50070/
Yarn web 页面 http://mdou11:8088/

### Hadoop 测试

```bash
# 创建test目录
[root@mdou10 hadoop-2.9.2] hdfs dfs -mkdir -p /test

# 本地abc.txt上传文件到hdfs
[root@mdou10 hadoop-2.9.2] hdfs dfs -put abc.txt /test/abc.txt

# 查看hdfs上的文件
[root@mdou10 hadoop-2.9.2] hdfs dfs -cat /test/abc.txt

# 执行hadoop统计单词数量的示例（wordcount是执行的任务，/wcoutput目录会自动生成）
[root@mdou10 hadoop-2.9.2] hadoop jar /usr/local/hadoop-2.9.2/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.9.2.jar wordcount /test/abc.txt /wcoutput
```
