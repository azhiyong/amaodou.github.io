---
title: Git使用
tags: Git
date: 2020-05-09 18:53:12
---

本文简单介绍一下 Git 配置、使用、以及多个远程仓库同步

### Git 配置

**添加`--global`参数进行全局配置**

- 配置用户信息

  Git 每次提交都会使用这些信息，并且记录到提交中

  ```bash
  git config --global user.name "amaodou"
  git config --global user.email "amaodou@gmail.com"
  ```

- Git 换行符转换

  不同平台的换行符不一样，为了解决这个问题，Git 可以配置换行自动转换

  _Windows 平台的换行符是 CRLF，Linux 平台的换行符是 LF_

  ```bash
  # 提交时换行符转换成 LF，检出时转换成 CRLF （一般 Windowns 平台配置）
  git config --global core.autocrlf true

  # 提交时换行符转换成 LF，检出时不处理
  git config --global core.autocrlf input

  # 提交和检出都不处理
  git config --global core.autocrlf false
  ```

- Git 别名

  添加如下配置后，可以使用`git co`代表`git checkout`

  ```bash
  git config --global alias.co checkout
  git config --global alias.br branch
  git config --global alias.ci commit
  git config --global alias.st status
  ```

  也可以通过设置别名定义 log 输出格式

  ```bash
  git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cd) %C(bold blue)<%an>%Creset' --abbrev-commit --date=format:'%Y-%m-%d %H:%M:%S'"
  ```

### Git 操作

- 初始化 Git 仓库

  ```bash
  git init
  ```

- 将变更添加到缓冲区

  ```bash
  git add .
  ```

- 提交缓冲区的变更

  ```bash
  git commit -m "说明本次提交内容"
  ```

- 拉取远程仓库

  ```bash
  git pull
  ```

- 推送到远程仓库

  ```bash
  git push
  ```

  如果本地仓库还没有跟远程仓库关联，需要先做关联再推送

  ```bash
  git remote add origin git@github.com:amaodou/test.git  # 关联远程仓库
  git push -u origin master  # 推送到远程 master 分支
  ```

  如果要推送到远程仓库的其他分支，提交到 draft 分支

  ```bash
  git remote add origin git@github.com:amaodou/test.git
  git checkout draft  # 本地仓库分支 draft
  git push -u origin draft  # 推送到远程 draft 分支
  ```

### Git 多个远程仓库同步

比如本地仓库想要同时推送到 github 和 gitee 仓库

```bash
git remote add origin git@github.com:amaodou/test.git  # github 仓库地址
git remote set-url --add origin git@gitee.com:amaodou/test.git  # gitee 仓库地址
git push -u origin master
```
