---
layout: post
title: 在Linux中将phy连接到fixed phy bus
description: 在Linux中将phy连接到fixed phy bus
summary: 在Linux中将phy连接到fixed phy bus
tags: [doc]
---


开发板上放了个交换机的phy，而且走的不是MDIO，而是SMI，驱动无法识别成有效phy，导致网络不通。以下是修改驱动把phy连接到fixed bus，从而建立kernel与phy之间的通信。本次修改基于Linux 3.x，没有启用dts。

1. 首先把phy添加到fixed驱动里面

```C
// phy的属性
struct fixed_phy_status status = {
	.link = 1, // 是否启用
	.speed = 1000, // 速度
	.duplex = 1, // 全双工
};

// 因为识别不到phy，phy_addr给一个系统还没有注册的值就好，这里是0
fixed_phy_add(PHY_POLL, phy_addr, status);
```

2. 在fixed bus里面找到刚刚添加的phy

```C
// 这里的bus是fixed bus，在fixed bus驱动里面，定义为
// struct fixed_mdio_bus *fmb；
// bus = fmb->mii_bus；
phy = get_phy_device(bus, phy_addr, false);
```

3. 把找到的phy注册设备

```C
phy_device_register(phy);
```

4. 注册成功后，在后续驱动的phy_connect()就可以正常连接到phy了，要注意的是传递给phy_connect的bus_id是fixed bus的id，而不是实际的bus id。

ps：高版本的Linux通常都启用dts，通过dts配置phy连接到fixed bus网上已有相关blog，这里不再叙述。
