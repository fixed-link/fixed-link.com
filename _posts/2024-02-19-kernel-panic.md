---
layout: post
title: kernel panic 的一些调试相关内容
description: kernel panic 的一些调试相关内容
summary: kernel panic
tags: [doc]
---


关键信息
```bash
ethtool_check_ops+0x18/0x38
函数名称：ethtool_check_ops
函数长度：0x38
偏移位置：0x18
函数基址：通过system.map或/proc/kallsyms查询
```

寻找出错位置
```bash
# 前置条件：CONFIG_DEBUG_INFO=y
# 0xFFFF800011AD4328 由上文的函数基址+偏移位置得出，也即是PC寄存器当前值
aarch64-linux-gnu-addr2line -C -f -e vmlinux 0xFFFF800011AD4328

```

调试手段
```C
#include <linux/bug.h>

dump_stack(); // 打印栈信息

// 产生panic，并调用dump_stack打印栈信息
BUG();
BUG_ON(condition);

// 调用dump_stack但不产生panic
WARN(condition,format...)
WARN_ON(condition)
WARN_ONCE(condition, format...)
WARN_ON_ONCE(condition)

```

一些相关寄存器

> R15：又叫程序计数器（Program Counter）PC，PC主要用于存放CPU取指的地址。

> R14：又叫链接寄存器（Link register）LR，LR主要用于存放函数的返回地址，即当函数返回时，知道自己该回到哪儿去继续运行。

> R13：又叫堆栈指针寄存器（Stack pointer）SP，SP通常用于保存堆栈地址，在使用入栈和出栈指令时，SP中的堆栈地址会自动的更新。

> R12：又叫内部过程调用暂存寄存器（Intra-Procedure-call scratch register）IP，主要用于暂存SP。

> R11：又叫帧指针寄存器（Frame pointer）FP，通常指向一个函数的栈帧底部，表示一个函数栈的开始位置。
