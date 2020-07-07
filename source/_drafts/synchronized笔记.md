---
title: synchronized笔记
tags:
---
synchronized 5层怎么实现? 字节码添加MONITORENTER、MONITOREXIT

synchronized 锁升级 : new -> 偏向锁（对象头的markword中存放线程id） -> 轻量级锁（markword指向竞争线程在线程栈中生成LR） -> 重量级锁（向操作系统内核申请）

![锁升级](/image/java/synchronized锁升级.png)

cpu多级缓存

cpu指令重排序
