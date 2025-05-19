---
layout: default
title:  "【玩转树莓派】制作一个口袋服务器 2"
tags: raspberrypi zh-cn
published: true
---

书接上回，笔者用树莓派 Zero 2 W 制作了一个口袋服务器。用起来总觉得不得劲。

口袋做到了，服务器还是差很远。

树莓派的系统用来玩玩还行，真正可靠性还是差点意思，有些功能还是跟服务器系统差一些。

所以我想到了 [Ubuntu Server](https://cdimage.ubuntu.com/releases/20.04/release/)。

选择[Preinstalled server image](https://cdimage.ubuntu.com/releases/20.04/release/ubuntu-20.04.5-preinstalled-server-armhf+raspi.img.xz)。看在口袋服务器的份上，还是选择 Hard-Float 版本的。

按照常规方式烧录到TF卡上，插入树莓派，接上电源，连接串口工具，打开 PuTTY，设置波特率为 `115200`，数据位 `8`，停止位 `1`，无奇偶校验。然后插入TF卡，接上电源，就可以看到树莓派的启动信息了。

登录信息和树莓派OS不太一样，参考[此处](https://wiki.ubuntu.com/ARM/RaspberryPi#First_boot_.28Username.2FPassword.29)。

> The login username is "`ubuntu`", password is "`ubuntu`". **You will be asked to change the password on first login.**

## 连接 Wi-Fi

依旧是轻车熟路，先连个Wi-Fi。

Ubuntu Server 20.04 之后的版本连接 Wi-Fi 的方式略显诡异。

打开：

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

在文件中添加以下内容：

```yaml
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
        usb0:
            dhcp4: no
            addresses:
                - 192.168.100.2/24
    version: 2
    wifis:
        wlan0:
            optional: true
            dhcp4: true
            access-points:
                "你家Wi-Fi名称":
                    hidden: true
                    password: "你家Wi-Fi密码"
```

保存后，执行以下命令：

```bash
sudo netplan apply
```

## 换源

连上Wi-Fi 之后，换源。参考[此处](https://mirrors.ustc.edu.cn/help/ubuntu-ports.html)。

```bash
sudo sed -i -e 's@//ports.ubuntu.com/\? @//ports.ubuntu.com/ubuntu-ports @g' \
            -e 's@//ports.ubuntu.com@//mirrors.ustc.edu.cn@g' \
            /etc/apt/sources.list
```

## 更新软件包列表并安装 `ifupdown` 和 `dnsmasq`

```bash
sudo apt update
sudo apt install -y ifupdown dnsmasq
```

## 配置 USB 网络

首先得让 `dnsmasq` 监听 USB 网络接口，把原来的服务关掉。

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

然后设置接口：

```bash
sudo nano /etc/network/interfaces
```

然后添加以下内容：

```bash
auto lo
iface lo inet loopback

auto usb0
allow-hotplug usb0
iface usb0 inet static
        address 192.168.100.2
        netmask 255.255.255.0

allow-hotplug wlan0
iface wlan0 inet dhcp
```

接下来配置 `/etc/dnsmasq.conf` 文件，添加以下内容：

```bash
interface=usb0
dhcp-range=192.168.100.10,192.168.100.100,24h
dhcp-option=3
dhcp-option=6
```

新建 systemd 服务文件等待 usb0 后再启动 dnsmasq

```bash
sudo nano /etc/systemd/system/dnsmasq-usb0.service
```

添加以下内容：

```bash
[Unit]
Description=Delayed start of dnsmasq after usb0 is up
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStartPre=/bin/bash -c 'while ! ip link show usb0 | grep -q "state UP"; do sleep 1; done'
ExecStart=/usr/sbin/dnsmasq -k --conf-file=/etc/dnsmasq.conf
Restart=always

[Install]
WantedBy=multi-user.target
```

保存后，执行以下命令：

```bash
sudo systemctl daemon-reload
sudo systemctl disable dnsmasq
sudo systemctl enable --now dnsmasq-usb0.service
```

当然，也可以重启树莓派。

基本上每次等能连接差不多要1~2分钟，正好在猩八课插上电脑再去拿个咖啡回来就能干活了。
