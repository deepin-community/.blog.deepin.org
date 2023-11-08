---
title: "xtrace"
date: 2023-11-04T10:19:31+08:00
draft: false
authors: ["zzxyb"]
tags: ["x11"]
---

## 介绍
xtrace是一个用于跟踪分析X11图形协议通信的工具,它可以监控和记录X11服务器上的各种场景,以帮助开发人员诊断和调试与图形界面相关的问题。作为一款强大的工具，xtrace可用于逆向工程、调试分析、性能分析等领域。在Linux X11系统中，xtrace能够记录一个程序在运行时所发起的X11协议请求和XServer发送给程序的事件，以及这些调用的参数。这对于在不阅读源码情况排查程序中的问题、理解程序行为、分析性能瓶颈以及进行协议审计都非常有用。笔者在工作过程中使用该工具深度剖析过腾讯会议、simplescreenrecorder等应用程序的实现，在没有阅读代码的前提下可以获得软件录屏的工作流程，配合阅读常规的X11录屏代码，可分析出其部分工功能异常原因，这些经验在xwayland适配X11应用程序截图录屏项目中通过实战解决了一系列问题。

本文主要介绍的是X11协议的监视工具，希望您在读完本篇文章后可以对如何监控X11协议有比较深刻的认知，在工作中经常会碰到一些X11应用程序运行时功能异常，在没有应用程序源码的情况下通过xtrace进行调试是一个不错的选择，希望在阅读完这篇文章后，能丰富您的调试技巧。祝您阅读愉快！
<!--more-->

xtrace的安装和使用非常简单，打开终端像如下输入命令即可启动工具：

```shell
// UOS上安装xtrace，deepin上没有可以在http://snapshot.debian.org/上下载
sudo apt install xtrace

// 运行之后协议电报内容会存储在/tmp/dde-calendar.xtrace中
xtrace -n -o /tmp/dde-calendar.xtrace dde-calendar	
```

| 命令行选项       | 含义或用途  |
| ---------- | ----- | 
| --display, -d | 用于ssh远程调试，例如:xtrace -d :0 dde-calendar |
| --outfile, -o | 用于将通信内容转存到磁盘，例如:xtrace -o /tmp/x.log dde-calendar |
| --stopwhendone, -s | 进程退出后停止xtrace，例如:xtrace -s dde-calendar |

## 工作原理
如图1所示，X11其主要有两种通信方式，在同一个计算机内主要使用unix domain socket进行通信；在不同计算机之间使用TCP/IP通信。只需要“截获”通信消息，将其转为易于读取的格式输出即可达到监控目的。这是xtrace工作的基本原理。
{{< figure src="x11-frame.jpg" title="图1. 不同客户端使用同一个XServer显示器框架图" >}}

如下图2是X11窗口创建的基本流程，因为篇幅限制，图中只绘制了基础的请求和事件，但X11协议远比图中绘制的复杂，笔者在此只介绍一下基础的协议交互模型，方便在读者在查看xtrace追踪日志时有基础的认知。
{{< figure src="client-server.jpg" title="图2. X11协议交互流程" >}}

总之xtrace是Xorg X Server自带的一个工具，通过分析xtrace的输出，你可以了解X Server是如何处理客户端应用程序的请求，以及可能的问题所在。xtrace 工具的作用：
 - 调试X问题：如果你遇到了与图形界面相关的问题，例如窗口无法正常显示、图形卡驱动问题等，你可以使用xtrace来捕获X Server的活动，帮助你找出问题所在。
 - 性能分析：xtrace可以记录X Server内部的操作，帮助你分析系统的图形性能，找出潜在的瓶颈。
 - 理解X协议交互：X Server与客户端应用程序之间的通信是通过X协议进行的。xtrace可以捕获这些通信，帮助你理解应用程序与X Server之间的交互方式。
 - 功能逆向：对于依赖X11协议的软件无法查看源码，可以通过观察通信协议，逆向其部分核心功能的实现
请注意，xtrace的输出可能会非常详细，因此在使用时需要注意过滤和分析输出，以便关注于你感兴趣的信息。

X11在文件系统中暴露了通信用的socket套接字文件，这使得第三方应用程序可以监控套接字文件从而“窥视"X11客户端和服务端之间的通信，这一技术点是xtrace工作的主要基础。

```shell
// X11 socket套接字文件在文件系统中的位置
ls /tmp/.X11-unix
X0  X9
```

如下日志所示xtrace工作的本质就是不断地获取wrote和received的数据然后将其解析为易于阅读的描述语言打印出来。
```shell
// 全量日志
001:>:received 32 bytes
001:>:09dc:32: Reply to InternAtom: atom=0x250("_NET_KDE_COMPOSITE_TOGGLING")
001:>:wrote 32 bytes
001:>:received 32 bytes
001:>:09dc: Event XKEYBOARD-XkbEvent(85) type=2 time=0x018d8071 device=0x03 not-yet-supported=0x10,0x00,0x00,0x10,0x01,0x00,0x00,0x00,0x00,0x01,0x90,0x10,0x10,0x10,0x90,0x00,0x01,0x90,0x11,0x00,0x00,0x87,0x05;
001:>:wrote 32 bytes
001:>:received 32 bytes
001:>:09dc: Event XKEYBOARD-XkbEvent(85) type=2 time=0x018d8072 device=0x03 not-yet-supported=0x10,0x00,0x00,0x10,0x00,0x00,0x00,0x00,0x00,0x00,0x10,0x10,0x10,0x10,0x10,0x00,0x01,0x90,0x11,0x00,0x00,0x87,0x05;
001:>:wrote 32 bytes
001:>:received 32 bytes
001:>:09dc: Event XKEYBOARD-XkbEvent(85) type=2 time=0x018d8073 device=0x03 not-yet-supported=0x10,0x00,0x00,0x10,0x01,0x00,0x00,0x00,0x00,0x01,0x90,0x10,0x10,0x10,0x90,0x00,0x01,0x90,0x11,0x00,0x00,0x87,0x05;
```

其工作流程大致如下：
 - 通过套接字地址族AF\_INET判断是tcp通信还是本地domain socket通信，然后通过generateSocketName或者calculateTCPport拿到addr相关的信息
 - 进入mainqueue死循环读取socket通信数据
 - 通过parse\_server(c)翻译Event事件通信，然后打印输出
 - parse\_client(c)翻译Requst请求通信，然后打印输出

相关的代码调用堆栈如下：
```c
Breakpoint 1, startline (c=0x4f8270, d=TO_SERVER, format=0x417a71 "%04x:%3u: Request(%hhu): %s ") at parse.c:52
52              if( (print_timestamps || print_reltimestamps)
(gdb) bt
#0  startline (c=0x4f8270, d=TO_SERVER, format=0x417a71 "%04x:%3u: Request(%hhu): %s ") at parse.c:52
#1  0x000000000040b5f5 in print_client_request (c=0x4f8270, bigrequest=false) at parse.c:1692
#2  0x000000000040cc2d in parse_client (c=0x4f8270) at parse.c:1996
#3  0x0000000000403b4a in mainqueue (listener=4) at main.c:406
#4  0x00000000004045d4 in main (argc=4, argv=0x7fffffffdef8) at main.c:706
(gdb) c
Continuing.

Breakpoint 1, startline (c=0x4f8270, d=TO_CLIENT, format=0x417b32 "%04x:%u: Reply to %s: ") at parse.c:52
52              if( (print_timestamps || print_reltimestamps)
(gdb) bt
#0  startline (c=0x4f8270, d=TO_CLIENT, format=0x417b32 "%04x:%u: Reply to %s: ") at parse.c:52
#1  0x000000000040c1f3 in print_server_reply (c=0x4f8270) at parse.c:1859
#2  0x000000000040d0e5 in parse_server (c=0x4f8270) at parse.c:2064
#3  0x00000000004035ed in mainqueue (listener=4) at main.c:336
#4  0x00000000004045d4 in main (argc=4, argv=0x7fffffffdef8) at main.c:706

// connect socket
Breakpoint 2, generateSocketName (addr=0x7fffffffdd30, display=9) at x11common.c:114
114             snprintf(addr->sun_path,sizeof(addr->sun_path),"/tmp/.X11-unix/X%d",display);
(gdb) p display 
$5 = 9
(gdb) c
Continuing.
[Detaching after fork from child process 7682]
Got connection from unknown(local)

Breakpoint 2, generateSocketName (addr=0x7fffffffd9d0, display=0) at x11common.c:114
114             snprintf(addr->sun_path,sizeof(addr->sun_path),"/tmp/.X11-unix/X%d",display);
(gdb) p display 
$6 = 0
(gdb) bt
#0  generateSocketName (addr=0x7fffffffd9d0, display=0) at x11common.c:114
#1  0x0000000000404c92 in connectToServer (displayname=0x7fffffffe363 ":0", family=1, hostname=0x0, display=0) at x11client.c:77
#2  0x0000000000402766 in acceptConnection (listener=3) at main.c:95
#3  0x0000000000403edd in mainqueue (listener=3) at main.c:452
#4  0x00000000004045d4 in main (argc=2, argv=0x7fffffffdf18) at main.c:706
(gdb) 
```


## 协议分析
xtrace通过拦截X11协议通信来进行分析。它捕获传输到X服务器的请求以及服务器对这些请求的响应，将他们解析化以日志输出的形式打印出来。所以本质来说协议分析指的是X11协议交互分析，需要对
X11相关的协议做到非常了解，即每个协议有什么功能，在xcb中是如何处理，在xserver中又是如何处理。受限于篇幅，笔者在此不会阐述所有的协议分析，而是拿我们平时常用的一些软件做一些分析和介绍。
| 日志流关键字       | 含义或用途  |
| ---------- | ----- | 
| Present-Request(148,1) | 客户端用于GLX等送显 |
| Request(1): CreateWindow | 请求创建X窗口，shm、glx都有 |
| GLX-Request(152,3): glXCreateContext | 请求创建GLX上下文，可以用来判断客户端是否GLX应用 |
| MIT-SHM-Request(130,3): PutImage | X11 shm客户端请求更新图像 |
| Event XKEYBOARD-XkbEvent(85) | xserver键盘事件传递给X客户端 |
| Event Generic(35) XInputExtension | 鼠标事件，后面带着ButtonPress、ButtonRelease、Motion等 |
| Request(36): GrabServer | grab请求 |
| Request(37): UngrabServer | 解除grab请求 |
| DeleteProperty | 请求删除X11窗口的一些属性 |
| ChangeProperty | 改变X11窗口的一些属性 |
| PropertyNotify | 窗口属性改变发送事件通知客户端 |
| MIT-SHM-Request(130,4): GetImage | shm方式获取屏幕图像 |
| Request(62): CopyArea | 复制屏幕一部分区域图像（离屏） |
| Request(53): CreatePixmap | 创建图像（离屏） |

上述表格中只是介绍了常见部分的协议，X协议非常的丰富，完整的模块如下所示：
 - XCB BigRequests API	发送和接受超过请求长度(65535字节)限制的数据
 - XCB Composite API	支持窗口合成，将多个窗口的内容合成最终的显示图像
 - XCB Damage API	用于跟踪窗口或者绘图上下文的可视区域的改变
 - XCB DPMS API	用于管理显示器的电源管理功能，控制显示器电源模式
 - XCB DRI2 API	支持直接渲染和硬件加速
 - XCB DRI3 API	支持直接渲染和硬件加速，对DRI2的扩展和改进
 - XCB Glx API	提供GLX接口创建OpenGL上下文，进行图形渲染和交互
 - XCB Present API	实现高性能的图像呈现在屏幕上
 - XCB RandR API	管理显示器的分辨率、屏幕方向和显示器布局等
 - XCB Record API	记录X服务器的事件流，以便进行调试、分析等
 - XCB Render API	xrender绘图
 - XCB ScreenSaver API	管理屏幕保护程序的行为和状态
 - XCB Shape API	用于创建和操作不规则窗口的形状
 - XCB Shm API	应用程序能够通过共享内存的方式高效的传输图像
 - XCB Sync API	实现同步操作和时间戳的管理，确保预期的时序
 - XCB XCMisc API	提供额外的杂项函数和功能，获取一些服务器信息
 - XCB Core API	核心部分，提供了基本通信功能和操作
 - XCB Xevie API	拦截和处理X服务器上的事件流
 - XCB XF86Dri API	直接渲染，直接访问图形硬件，提高图形性能和效率
 - XCB XFixes API	增强功能：光标、窗口形状、窗口属性、窗口位置
 - XCB Xinerama API	用于管理多个显示器的配置和操作
 - XCB Input API	处理输入事件（如键盘、鼠标、触摸屏等）交互
 - XCB xkb API	配置与操作键盘相关的设置，键盘布局和状态
 - XCB XPrint API	直接从应用程序打印文档、图像和其他内容
 - XCB API	xcb基础功能
 - XCB SELinux API	在应用程序中管理selinux安全策略和执行安全操作
 - XCB Test API	测试协议，常用于远程控制
 - XCB Xv API	xvideo相关，用于视频渲染、视频加速、获取视频信息
 - XCB XvMC API	在GPU上执行视频解码和运动补偿功能
笔者在工作中也有时对xtrace日志无法分析到有用信息,此时会查看[XCB](https://xcb.freedesktop.org/manual/modules.html)帮助文档，通过分析对比查找到相关的xcb函数，从而逐渐熟悉X11协议。

## 总结
总体来说xtrace是一个有用的工具，对于笔者来说经常会接触生态软件的图形显示问题，对于少部分软件开发商不愿意提供代码和问题复现最小demo，此时其软件对于笔者来说是一个黑盒，当问题边界靠近X相关的技术时，笔者会使用xtrace去详细分析该软件的详细功能，往往这可以在底层剖析软件的显示工作方式，配合系统上相关的组建库代码，可以方便地处理客户的紧急问题。
笔者编写这篇文档是希望可以鼓励更多的同事使用xtrace，这个工具可以帮助你熟悉X11的工具原理，同时也可以理解像qt、gtk等UI库的底层实现，在定位系统复制粘贴、图形显示、拖拽、窗口相关的问题时可以提供更加底层的日志，方便更加精准地定位问题根因！

## 参考资料
- [X11](https://en.wikipedia.org/wiki/X_Window_System)
- [XServer](https://en.wikipedia.org/wiki/X_server)
