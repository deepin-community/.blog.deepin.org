---
title: Starting sessions with systemd part 2
date: 2021-12-29 21:09:07
tags: ['systemd']
draft: false
authors: ["justforlxz"]
---

接着上一篇文章，这篇文章主要讲解一下具体的实现。

<!--more-->

[https://github.com/linuxdeepin/dde-session](https://github.com/linuxdeepin/dde-session) 已经包含了所有的文件和提交。

上面文章说过，为了让 dde 使用 systemd --user 来关系服务，我提供了一组服务：

```
dde-session-initialized.target
dde-session-manager.service
dde-session-manager.target
dde-session-pre.target
dde-session-restart-dbus.service
dde-session-shutdown.service
dde-session-shutdown.target
dde-session.target
dde-session-x11-services-ready.target
dde-session-x11-services.target
dde-session-x11.target
org.deepin.Session.service
```

lightdm 在认证通过以后，会启动 dde-session 进程，dde-session 会通过 systemd 的 dbus 启动 `org.deepin.Session.service`。

在 `org.deepin.Session.service` 中会执行 `dde-session-ctl --systemd-service`，启动 `dde-session-x11.target`.

`dde-session-x11.target` 关联了所有初始化的服务，systemd 会帮助自动运行这些服务，并且最终会运行到 `dde-session-manager.service` 上。

> dde-session-manager.service

```ini
[Service]
Type=simple
ExecStart=/usr/bin/startdde
ExecStopPost=/usr/lib/libexec/dde-session-ctl --logout
```

在 `org.deepin.Session.service` 中会启动一个特殊的服务，这个服务会监听 dde-session 的退出，因为总要有一个服务去监听 session 的状态，要确保 session 在服务在，session 退服务退，服务退 session 退，共存亡，这也是所有操作中最麻烦的一步。

> org.deepin.Session.service

```ini
[Service]
Type=dbus
BusName=org.deepin.Session
ExecStart=/usr/bin/dde-session --systemd-service
ExecStop=/usr/lib/libexec/dde-session-ctl --shutdown
```

在 dde-session-ctl 中是这么实现 shutdown 的:

```c++
if (isShutdown) {
    // kill startdde-session or call login1
    QDBusInterface systemd("org.freedesktop.systemd1", "/org/freedesktop/systemd1", "org.freedesktop.systemd1.Manager");
    qInfo() << systemd.call("StartUnit", "dde-session-shutdown.target", "replace-irreversibly");
    return 0;
}
```

在 dde-session-ctl 中是这么实现 logout 的:

```cpp
if (parser.isSet(logout)) {
    org::deepin::Session session("org.deepin.Session", "/org/deepin/Session", QDBusConnection::sessionBus());
    session.Logout();

    return 0;
}
```

> dde-session main.cpp

```c++
// 添加启动参数，表示这是通过 systemd 启动的。
QCommandLineOption systemd(QStringList{"d", "systemd-service", "wait for systemd services"});
parser.addOption(systemd);
parser.process(app);

if (parser.isSet(systemd)) {
    return -1;
}

QDBusServiceWatcher *watcher = new QDBusServiceWatcher("org.deepin.Session", QDBusConnection::sessionBus(), QDBusServiceWatcher::WatchForUnregistration);
    watcher->connect(watcher, &QDBusServiceWatcher::serviceUnregistered, [=] {
        QDBusInterface systemdDBus("org.freedesktop.systemd1", "/org/freedesktop/systemd1", "org.freedesktop.systemd1.Manager");
        qInfo() << systemdDBus.call("StartUnit", "dde-session-shutdown.service", "replace");
        qApp->quit();
    });
    qInfo() << systemdDBus.call("StartUnit", "dde-session-x11.target", "replace");
}
```

`dde-session-shutdown.service` 只负责执行 `dde-session-ctl --shutdown`，用于启动 `dde-session-shutdown.target`。

> dde-session-shutdown.service

```ini
[Service]
Type=oneshot
ExecStart=/usr/lib/libexec/dde-session-ctl --logout
```

通过 systemd 的接口启动了 `dde-session-shutdown.target`，从而启动了清理流程。

`dde-session-shutdown.target` 没有执行任何内容，仅仅是关联了一组服务，但是是冲突这些服务，因为这是启动的时候

在 `dde-session-shutdown.target` 中还关联了最重要的一个服务: `dde-session-restart-dbus.service`，这个服务是用来关闭 `dbus.service` 服务的，这样就可以保证 session 结束的时候，不会有 dbus 服务逃逸出去。

> dde-session-restart-dbus.service

```ini
[Service]
Type=notify
ExecStart=/usr/lib/libexec/dde-session-ctl --restart-dbus
```

在 dde-session-ctl 中是这么实现 restart-dbus 的:

```cpp
if (isRestartDBus) {
    QDBusInterface systemd("org.freedesktop.systemd1", "/org/freedesktop/systemd1", "org.freedesktop.systemd1.Manager");
    qInfo() << systemd.call("StopUnit", "dbus.service", "replace-irreversibly");
    return 0;
}
```

复盘一下整个流程，在登录界面输入密码登录以后，lightdm 会启动 `dde-session` 作为 session 的入口，`dde-session` 会注册一个 `org.deepin.Session` 的 dbus 服务，并且会启动关联的 systemd 服务，启动一个 `dde-session --systemd-service`，在这个进程里，会启动 `dde-session-x11.target`，从而最终启动了 `dde-session-manager.service`，将原本的会话入口 startdde 启动了。

在 `dde-session --systemd-service` 会监听 `org.deepin.Session` 服务的存活，当服务不存在时，就会开始执行退出流程，从而启动了 `dde-session-ctl --shutdown`，在 dde-session-ctl 的帮助下，启动了 `dde-session-shutdown.target`，将所有启动的 dde 核心服务都冲突掉，从而完成了服务关闭，在最后阶段再将 `dbus.service` 服务停止，完成最终的清理。

还有一种退出模式，手动启动了 `dde-session-shutdown.service` 服务，也会触发完成的退出流程，`dde-session-shutdown.service` 会利用 `dde-session-ctl --logout` 将会话注销，从而触发上面的执行流程，完成会话与服务的关闭。

目前还有一些问题没有解决，没有创建独立的 systemd.slice 和 systemd.scope，这可以帮助 dde 细致的划分和管理自己的子 services，退出 dbus.service 的手段非常黑，而且 dbus 服务的程序由于图形服务也已经结束了，导致成片的 X11 Error。这些都是要解决的。

下一步准备写几个 service，用来代替 startdde 的组件启动，比如 kwin、dde-desktop、dde-dock 和 xdg-autostart 等。

就目前的情况来看，任重而道远。

原文链接：[https://blog.justforlxz.com/2021/12/29/Starting-sessions-with-systemd-part-2/](https://blog.justforlxz.com/2021/12/29/Starting-sessions-with-systemd-part-2/)
