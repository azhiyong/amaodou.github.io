---
title: Tomcat笔记
date: 2020-05-08 23:28:27
tags: Tomcat
---

## Tomcat 整体结构

![Tomcat](/images/tomcat/tomcat体系结构.png)

### Server

一个 Server 可以包含多个 Service

<!--more-->

#### Service

一个 Service 由多个 Connector 和一个 Container 组成，可以包含共享的线程池

##### Connector

Tomcat 的连接器，负责接收请求，并将请求交给 Container 处理

- Endpoint

  负责监听 Socket 连接，建立 TCP 3 次握手，并将 Socket 连接交给 Processor 处理

- Processor

  接收 Socket 读取输入流，封装 Tomcat 自身的 Request 和 Response，将 Request 和 Response 交给 CoyoteAdapter 处理

- ProtocolHandler

  处理协议和 IO，Tomcat 提供 6 个实现类：AjpNioProtocol，AjpAprProtocol，AjpNio2Protocol，Http11NioProtocol，Http11Nio2Protocol，Http11AprProtocol，一般使用 Http11NioProtocol

- Adapter

  实现类 CoyoteAdapter，将不同协议的请求内容适配成标准的 HttpServletRequest 和 HttpServletResponse，再交给 Container 处理

##### Container

Servlet 容器，负责加载和管理 Servlet，将请求交给具体的 Servlet 处理

- Engine

  Servlet 容器引擎，一个 Engine 管理多个 Host，负责将请求发送到对应的 Host 处理

- Host

  表示虚拟主机，一个 Host 可以包含多个 Context，负责将请求发送到对应的 Context 处理

- Context

  表示一个 web 应用，一个 Context 可以包含多个 Wrapper，负责将请求发送到对应的 Wrapper 处理

- Wrapper

  表示一个 Servlet，最终调用 Servlet.service 方法

### Tomcat 启动流程

![Tomcat启动流程](/images/tomcat/tomcat启动流程.png)

### Tomcat 请求处理流程

![Tomcat请求处理流程](/images/tomcat/tomcat请求处理流程.png)
