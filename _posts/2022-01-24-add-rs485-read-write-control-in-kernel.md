---
layout: post
title: 在kernel中增加rs485的读写控制
description: 在kernel中增加rs485的读写控制
summary: 在kernel中增加rs485的读写控制
tags: [doc]
---


一些rs485的IC通过gpio来控制读写方向，本文基于8250的驱动，讲讲在kernel中如何通过控制gpio来控制rs485的读写方向。

首先是增加定义：

```C
#include <linux/gpio.h>
#include <linux/timer.h>

#define RS485_NR        CONFIG_SERIAL_8250_NR_UARTS
#define RS485_GPIO_DEF  GPIO0_A3

typedef struct {
	ktime_t ktime;
	struct hrtimer timer;
	struct uart_port *port;
} rs485_ctl_t;

rs485_ctl_t rs485_ctl[RS485_NR];
```

然后在serial8250_init_port中初始化：

```C
int i = port->line;

// 这里用到了struct uart_port中的空闲变量unused1来保存gpio
port->unused1 = RS485_GPIO_DEF;
port->rs485_config = serial8250_rs485_config;

hrtimer_init(&rs485_ctl[i].timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);

rs485_ctl[i].timer.function = serial8250_rs485_stop_tx_timeout;
rs485_ctl[i].port = port;
```

接着实现在init中所涉及的函数：

```C
// 这里通过struct serial_rs485的padding[0]在应用层传递gpio，如果没有，就用默认的gpio
int serial8250_rs485_config(struct uart_port *port, struct serial_rs485 *rs485)
{
	unsigned gpio;

	memcpy(&port->rs485, rs485, sizeof(struct serial_rs485));

	gpio = (unsigned)port->rs485.padding[0];

	if (gpio > 0)
		port->unused1 = gpio;
	else
		gpio = (unsigned)port->unused1;

	if (port->rs485.flags & SER_RS485_ENABLED)
		return serial8250_rs485_gpio_init(gpio);

	return -EINVAL;
}

static int serial8250_rs485_gpio_init(unsigned gpio)
{
	int ret;

	if (gpio_is_valid(gpio))
	{
		ret = gpio_request(gpio, "RS485 R/W Switch GPIO");
		if (ret)
			return ret;
		else
		{
			gpio_direction_output(gpio, 0);
			return 0;
		}
	}

	return -EBUSY;
}

static enum hrtimer_restart serial8250_rs485_stop_tx_timeout(struct hrtimer *t)
{
	struct rs485_ctl_t *rs485_ctl =
		container_of(t, struct rs485_ctl_t, timer);
	struct uart_port *port = rs485_ctl->port;
	unsigned gpio = (unsigned)port->unused1;

	if (serial8250_tx_empty(port))
	{
		gpio_set_value(gpio, 0);
		return HRTIMER_NORESTART;
	}
	else
	{
		hrtimer_forward_now(t, rs485_ctl->ktime);
		return HRTIMER_RESTART;
	}
}
```

跟着在serial8250_start_tx中控制gpio来控制读写方向

```C
// 发送数据前拉高gpio
if (port->rs485.flags & SER_RS485_ENABLED)
	serial8250_rs485_start_tx((unsigned)port->unused1);

// 发送完成后拉低gpio
if (port->rs485.flags & SER_RS485_ENABLED)
{
	int i = port->line;

	rs485_ctl[i].port = port;
	rs485_ctl[i].ktime = ktime_set(0, HZ * 1000);

	hrtimer_restart(&rs485_ctl[i].timer,
	                rs485_ctl[i].ktime, HRTIMER_MODE_REL);
}
```

在关闭应用程序时，释放gpio

```C
if (port->rs485.flags & SER_RS485_ENABLED)
	serial8250_rs485_gpio_free((unsigned)port->unused1);
```

实现相关的函数

```C
static inline void serial8250_rs485_start_tx(unsigned gpio)
{
	gpio_set_value(gpio, 1);
}

static inline void serial8250_rs485_stop_tx(unsigned gpio)
{
	gpio_set_value(gpio, 0);
}

static inline void serial8250_rs485_gpio_free(unsigned gpio)
{
	if (gpio_is_valid(gpio))
	{
		gpio_set_value(gpio, 0);
		gpio_free(gpio);
	}

}
```

最后在serial8250_register_8250_port注释默认的rs485_config

```C
// uart->port.rs485_config	= up->port.rs485_config;
```
