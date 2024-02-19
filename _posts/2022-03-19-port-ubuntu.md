---
layout: post
title: 移植Ubuntu到嵌入式Linux平台
description: 移植Ubuntu到嵌入式Linux平台，ARM64 ROCKCHIP
summary: 移植Ubuntu到嵌入式Linux平台
tags: [doc]
---


在宿主机上下载基本包，然后在开发板进行安装
```bash
# 下载
sudo debootstrap --arch=arm64 --foreign --components=main,restricted,universe,multiverse \
    --exclude=apt-transport-https,gcc --include=openssh-server,vim bionic workspace \
    https://mirrors.cloud.tencent.com/ubuntu-ports
# 下载版本为bionic的Ubuntu到workspace这个目录，下载地址可以根据实际情况替换
# Ubuntu的版本代号可前往https://wiki.ubuntu.com/Releases查看

# 将workspace复制到开发板上，然后安装
LANG=C.UTF-8 LANGUAGE==C.UTF-8 LC_ALL=C.UTF-8 \
    chroot workspace /debootstrap/debootstrap --second-stage --no-check-gpg
```

或者，直接在开发板上下载安装
```bash
debootstrap --arch=arm64 --components=main,restricted,universe,multiverse \
    --exclude=apt-transport-https,gcc --include=openssh-server,vim bionic workspace \
    https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports
```

安装完成后，在开发板进入Ubuntu，开始进行配置
```bash
chroot workspace
```

配置网络 [参考](https://cloud.tencent.com/developer/article/1699857)
```bash
cat /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    eth0:
      dhcp4: true
  version: 2
```

如果需要桌面接管网络，则如下配置
```
cat /etc/netplan/00-installer-config.yaml
network:
  version: 2
  renderer: NetworkManager
  
diff /etc/NetworkManager/NetworkManager.conf
-managed=false
+managed=true
```

添加默认dns
```
diff /etc/systemd/resolved.conf
[Resolve]
-#DNS=
+DNS=8.8.8.8 114.114.114.114 119.29.29.29 223.6.6.6 180.76.76.76 1.1.1.1 1.2.4.8

# 使用systemd-resolve --status 查看dns状态
```

开机打印IP [参考](https://www.jianshu.com/p/7fd8b6ea336e)
```bash
cat /lib/systemd/system/getip.service
[Unit]
Description=get ip
After=systemd-networkd.service

[Service]
Type=idle
ExecStart=/etc/getip.sh

[Install]
WantedBy=multi-user.target

#===========================#

cat /etc/getip.sh
#!/bin/sh

ip=`hostname -I`

while [ -z $ip ]
do
        ip=`hostname -I`
        sleep 1
done

echo ============== $ip============== > /dev/console

#===========================#
# 使能
systemctl enable /lib/systemd/system/getip.service
```

修改串口自动登录root [参考](https://blog.csdn.net/qq_40177571/article/details/113498224)
```bash
cp /lib/systemd/system/serial-getty\@.service \
/lib/systemd/system/serial-getty\@ttyS0.service
diff /lib/systemd/system/serial-getty\@ttyS0.service
-ExecStart=-/sbin/agetty -o '-p -- \\u' --keep-baud 115200,38400,9600 %I $TERM
+ExecStart=-/sbin/agetty -a root --keep-baud 1500000,115200,38400 %I $TERM
# 使能
systemctl enable /lib/systemd/system/serial-getty\@ttyFIQ0.service
```

修改sshd允许root登录 [参考](https://blog.csdn.net/Magic_Ninja/article/details/102308367)
```bash
diff /etc/ssh/sshd_config
-#PermitRootLogin prohibit-password
+PermitRootLogin yes
```

扩展rootfs
```bash
cat /lib/systemd/system/resize-helper.service
[Unit]
Description=Resize root filesystem to fit available disk space
After=systemd-remount-fs.service

[Service]
Type=oneshot
ExecStart=/sbin/resize2fs /dev/mmcblk0p6
ExecStartPost=/bin/systemctl disable resize-helper.service

[Install]
WantedBy=local-fs.target

#===========================#
# 使能
systemctl enable /lib/systemd/system/resize-helper.service
```

禁止休眠（可选）
```bash
systemctl disable sleep.target suspend.target hibernate.target hybrid-sleep.target
```

挂载user分区（如果是桌面环境可以跳过）
```bash
cat /etc/fstab

/dev/mmcblk0p8 /mnt ext4 defaults 0 0
```

配置系统
```bash
# 配置locales
dpkg-reconfigure locales
# 配置时区
dpkg-reconfigure tzdata
# 更换APT源，选择适合的源，为避免错误，先用HTTP协议的
vi /etc/apt/sources.list
# 然后更新证书
apt update && apt install apt-transport-https ca-certificates
# 修改/etc/apt/sources.list，启用HTTPS的源，更新整个系统
apt update && apt upgrade -y
```

退出，然后打包
```bash
dd of=bionic.ext4 bs=1M seek=6144 count=0 # seek是rootfs分区的大小
mke2fs -t ext4 -d workspace bionic.ext4
tune2fs -c 0 -i 0 bionic.ext4
resize2fs -M bionic.ext4
```

把workspace备份到宿主机以便后续开发
```bash
tar cf bionic.tar workspace
scp bionic.tar user@IP:.
```

把bionic.ext4复制到宿主机然后进行烧录到开发板
```
scp bionic.ext4 user@IP:.
```

搞定
