---
layout: post
title: 对不同的主机使用不同的公钥连接
description: 对不同的主机使用不同的公钥连接
summary: 对不同的主机使用不同的公钥连接
tags: [doc]
---


1. 首先生成公钥
```
ssh-keygen -t rsa -b 4096 -C "<user@server>" -f ~/.ssh/<filename>
将user@server和filename有意义的字符串。此时密钥对就会在指定路径生成。
```
2. 将\<filename\>.pub 拷贝到远程主机上
```
ssh-copy-id -i ~/.ssh/<filename>.pub <user@server>
远程主机会要求你登录。
```
3. 拷贝完成后，在~/.ssh/config作出如下配置：
```
host <alias name>
    user <your name>
    hostname <server name>
    port <ssh port>
    identityfile ~/.ssh/<filename>

例如：

host me
    user test
    hostname chowe.one
    port 22
    identityfile ~/.ssh/test

保存后执行

ssh me

这样就会以你设置的密钥登陆到远程主机上。

同样，也可以用于scp风格的指令中：
scp me:path/to/filename ~/path/to/filename
```