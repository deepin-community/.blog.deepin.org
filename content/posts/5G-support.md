---
title: deepin 5g wwan 支持
date: 2023-11-8
draft: false
authors: ["longlong","nn3300"]
tags: ["Tools"]
---

## 第一步：安装modemmanager

modemmanager是一个由freedesktop托管的项目，旨在在linux设备上运行调制解调器以让linux设备获得蜂窝无线网络连接的能力，所以我们第一步骤就是安装此软件。目前deepin已经支持此软件最新版本：

```bash
sudo apt install modemmanager
```

同时，你需要确保你的内核模块已经正确加载了你的WWAN驱动，你可以通过使用`lspci -vvv`查看详细信息

在安装了modemmanager后，你可以通过`mmcli` 命令使用命令行与其交互，使用`mmcli —help-all`来获取全部的帮助选项。

## 第二步：使用mmcli连接5G网络

使用`mmcli -L` 获取你的wwan卡信息，返回的信息有三个内容，分别为：DBus地址、设备类型 、设备ID，其中DBus地址的最后一位为设备编号，你可以使用`mmcli --modem=<设备编号>`的方式查看设备详细支持信息。

如果你的SIM卡设置了PIN锁，则需要在连接网络之前使用`mmcli --modem=0 --sim=0 --pin=****` 的方式连接，而后我们就可以启动相关设备了：

```bash
mmcli --modem=<设备编号> --enable
```

然后你需要使用simple connect连接网络

```bash
mmcli -m <设备编号> --simple-connect='apn=<apn名>,ip-type=ipv4v6'
```

比如我的连接方式为

```bash
mmcli -m 4 --simple-connect='apn=ctnet,ip-type=ipv4v6'
```

此时你再使用`mmcli -m <设备编号>` 查看信息的时候 可以查看Bearer的信息，Bearer的DBus的最后一位为其编号。使用命令查看Bearer的信息：

```bash
mmcli -m <设备编号> -b <Bearer编号>
```

如果你看到Bearer相关信息，就几乎接近成功了：

```text
  ------------------------------------
  General            |           path: /org/freedesktop/ModemManager1/Bearer/0
                     |           type: default
  ------------------------------------
  Status             |      connected: yes
                     |      suspended: no
                     |    multiplexed: no
                     |      interface: wwan0
                     |     ip timeout: 20
  ------------------------------------
  Properties         |            apn: ctnet
                     |        roaming: allowed
                     |        ip type: ipv4
                     |   allowed-auth: none, pap, chap, mschap, mschapv2, eap
                     |           user: ctnet@mycdma.cn
                     |       password: vnet.mobi
  ------------------------------------
  IPv4 configuration |         method: static
                     |        address: 10.122.58.19
                     |         prefix: 8
                     |        gateway: 10.122.58.17
                     |            dns: 202.103.24.68, 202.103.44.150
                     |            mtu: 1420
  ------------------------------------
  Statistics         |     start date: 2023-11-07T05:40:08Z
                     |       duration: 1260
                     |   uplink-speed: 1250000000
                     | downlink-speed: 4670000000
                     |       attempts: 1
                     | total-duration: 1260

```

## 第三步：使用nmcli打开连接

networkmanager对mm是有做支持，在你完成上述步骤之后，可以通过`ip a`命令查看，可以看到一个被down掉的接口，我们使用nmcli查看其详细信息：

```bash
nmcli device show
```

你就可以 看到一个以wwan开头的设备：

```text
GENERAL.DEVICE:                         wwan0mbim0
GENERAL.TYPE:                           gsm
GENERAL.HWADDR:                         （未知）
GENERAL.MTU:                            1420
GENERAL.STATE:                          100（已连接）
GENERAL.CONNECTION:                     wwan0mbim0
GENERAL.CON-PATH:                       /org/freedesktop/NetworkManager/ActiveConnection/3
IP4.ADDRESS[1]:                         10.122.58.19/8
IP4.GATEWAY:                            10.122.58.17
IP4.ROUTE[1]:                           dst = 10.0.0.0/8, nh = 0.0.0.0, mt = 700
IP4.ROUTE[2]:                           dst = 0.0.0.0/0, nh = 10.122.58.17, mt = 700
IP4.DNS[1]:                             202.103.24.68
IP4.DNS[2]:                             202.103.44.150
IP6.GATEWAY:                            --

```

使用下述命令打开此设备

```bash
nmcli d connect <设备名>
```

上述设备名就是以wwan开头的设备

然后你就可以使用5G网络连接了
