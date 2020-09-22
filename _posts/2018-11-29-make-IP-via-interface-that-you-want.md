---
layout: post
title: 对不同的IP指定访问路由/网卡
description: 对不同的IP指定访问路由/网卡
summary: 对不同的IP指定访问路由/网卡
tags: [doc]
---


1. 设定IP地址和网关
```
ip addr 192.168.1.11/24 dev ens33
将192.168.1.11替换成你的IP地址，将ens33替换成你的网卡名字
如果是DHCP动态获取IP，跳过这一步
```
2. 设定默认网关
```
sudo ip route add default via 192.168.1.1 dev ens33
```
3. 对特殊IP指定路由/网关
```
sudo ip route add 192.168.0.11 via 192.168.1.254 dev ens33
```
4. 结论

在例子中，如果你添加了0网段的IP但192.168.0.11不在局域网中这样是无法访问的，但是指定192.168.0.11通过默认网关192.168.1.254则可以正常访问。