---
title: ssh之config使用
tags: Linux
date: 2020-05-09 17:51:59
---

### ssh 使用

我们使用简单的 ssh 命令即可登录远程服务器，如`ssh root@192.168.10.101`（默认使用端口号 22），如果服务器使用的是非标准的端口，则需要指定 ssh 连接端口，如`ssh root@192.168.10.101 -p 10022`

ssh 命令行选项

```bash
usage: ssh [-1246AaCfGgKkMNnqsTtVvXxYy] [-b bind_address] [-c cipher_spec]
        [-D [bind_address:]port] [-E log_file] [-e escape_char]
        [-F configfile] [-I pkcs11] [-i identity_file] [-L address]
        [-l login_name] [-m mac_spec] [-O ctl_cmd] [-o option] [-p port]
        [-Q query_option] [-R address] [-S ctl_path] [-W host:port]
        [-w local_tun[:remote_tun]] [user@]hostname [command]
```

<!--more-->

当远程服务器数量增加到很难记住所有服务器的用户名、远程 ip、命令行选项时，我们该怎么办呢？

一般有两种方式处理：创建别名指令和使用 config 配置

### ssh 别名

在`~/.bashrc`或者`~/.zshrc`（使用 zsh）文件中添加别名配置，使用如下命令：

```bash
echo "alias dev=ssh root@192.168.10.101 -p 10022" >> ~/.bashrc
```

## ssh config

```bash
# ~/.ssh/config 文件内容
Host 101
    HostName 192.168.10.101
    User root
    Port 1022
```

添加上述配置之后，意味着我可以使用`ssh 101`命令登录 192.168.10.101 服务器。

也可以配置密钥登录，使用如下：

```bash
# ~/.ssh/config 文件内容
Host 101
    HostName 192.168.10.101
    User root
    Port 1022

Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github.key
```

添加上述配置，意味着当我使用 git clone 远程仓库时（比如 `git clone git@github.com:amaodou/amaodou.github.io.git`），使用密钥 github.key 进行认证。

注意：如果`~/.ssh/config`文件不存在时，需要手动创建同时设置文件不能被其他用户访问

```bash
# 创建.ssh目录
mkdir -p ~/.ssh && chmod 700 ~/.ssh

# 新建config文件，设置访问权限
touch ~/.ssh/config
chmod 600 ~/.ssh/config
```
