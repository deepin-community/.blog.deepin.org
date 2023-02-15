---
title: REUSE使用指南
date: 2022-08-10 13:00:00
draft: false
authors: ["longlong"]
tags: ["Tools"]
---

# 什么是REUSE

REUSE是一个工具，准确的来说是一个帮助我们下载开源许可证书和检查开源声明是否合规的脚本：
<!--more-->

# 使用REUSE配置项目

## 安装

REUSE依赖于本地的python环境，所以需要首先安装python和pip

```bash
sudo apt install python3
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
sudo python3 get-pip.py
```

然后安装REUSE

```bash
pip3 install --user reuse

```

在此之后，确保它`~/.local/bin`在您的`$PATH`中。在 Windows 上，您的环境所需的路径可能类似于 `%USERPROFILE%\AppData\Roaming\Python\Python39\Scripts`，具体取决于您安装的 Python 版本。

## 使用

### 初始化项目

进入一个项目，使用`reuse init`命令可以初始化项目，此时在你的命令行菜单上会以交互的方式询问你需要的许可证和版权所有人以及邮箱。注意：许可证的名称需要规范：[许可证名称](https://spdx.org/licenses/ "许可证名称")

如果你在这一步遗漏了也没事，可以使用`reuse download <许可证名称>`的方式下载所需的许可证，这些许可证会一并放入项目下的License文件夹中

### 为代码文件添加版权声明

#### 使用自动化脚本添加

含有代码逻辑文件添加版权声明方法为：

```bash
reuse addheader --copyright="UnionTech Software Technology Co., Ltd." --license=GPL-3.0-or-later -r <目录>
```

这只是一个简单的例子，本质上resuse还有更多的用法：

`--copyright-style`可以将默认值更改 SPDX-FileCopyrightText为以下样式之一：

```text
spdx:           SPDX-FileCopyrightText: <year> <statement>
spdx-symbol:    SPDX-FileCopyrightText: © <year> <statement>
string:         Copyright <year> <statement>
string-c:       Copyright (C) <year> <statement>
string-symbol:  Copyright © <year> <statement>
symbol:         © <year> <statement>
```

reuse不会限制你使用多少个copyright 也不会限制你使用的license的个数。

reuse 会猜测你使用的语言，并且添加对应的注释格式，如果不能正确添加或者需要更多的需求可以考虑下面的方法。

#### 手动进行添加

reuse允许你手动进行添加版权声明，只要你的版权声明符合它的规范即可。你所需要的是：充分的耐心，键盘上未曾损坏的`Ctrl` `C` `V` 三个按键即可完成这一切。我个人建议是用上面提供的自动化脚本添加一个版权声明作为模板，然后自己修改下，就可以展现你作为CV工程师的实力了。

#### 自动进行添加

理论上reuse不限制你添加版权声明的方式，你可以选择你喜欢的脚本语言来完成这一切。

### 为非代码文件添加版权声明

在项目中是存在非代码文件，比如图片，音频等等。部分也是受到版权保护的，但是我们也没法直接在上面操作（别和我说水印和音频水印，这俩玩意不能无损添加）所以我们要使用一个外置的dep5文件去声明这些非代码文件的版权，dep5文件在执行项目`init`之后就已经自动帮你生成好了，所以直接写就是的了。比如：

```text
# css
Files: *.css
Copyright: None
License: CC0-1.0
```

第一行就是注释 ，第二行是适合的文件，第三行是版权信息（如果使用cc0,则填写None） ，第四行是使用的licence。第一行的注释你可以选择不写，但是为了以后的可读性，强烈建议你写上，甚至可以作为分类的标签。

第二行文件，支持使用通配符进行模糊匹配，但是不支持正则表达式：仅支持，`*`匹配任意多个字符 `？`匹配一个字符。所以可以利用通配符来制定匹配某一个文件夹下面的所有文件

例子：

```text
# assets
Files: styleplugins/dstyleplugin/assets/*
Copyright: UnionTech Software Technology Co., Ltd.
License: GPL-3.0-or-later
```

这个例子展示了有版权信息的情况

例子：

```text
# png svg
Files: platformthemeplugin/icons/* styleplugins/chameleon/menu_shadow.svg styles/images/woodbackground.png
       styles/images/woodbutton.png tests/iconengines/builtinengine/icons/actions/icon_Layout_16px.svg
       tests/iconengines/svgiconengine/icon_window_16px.svg
Copyright: None
License: CC0-1.0
```

这个例子展示了为多个文件路径及文件配置版权信息

**强烈推荐你使用路径和具体文件名的方式来制定版权信息，而要避免大范围使用通配符的情况**

例子：

```text
Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
Upstream-Name: qt5integration
Upstream-Contact: UnionTech Software Technology Co., Ltd.  <>
Source: https://github.com/linuxdeepin/qt5integration

# README
Files: README.md CHANGELOG.md
Copyright: None
License: CC0-1.0

# assets
Files: styleplugins/dstyleplugin/assets/*
Copyright: UnionTech Software Technology Co., Ltd.
License: GPL-3.0-or-later

# Project file
Files: *.pro *.prf *.pri
Copyright: None
License: CC0-1.0

# css
Files: *.css
Copyright: None
License: CC0-1.0

# qrc
Files: *.qrc
Copyright: None
License: CC0-1.0

# png svg
Files: platformthemeplugin/icons/* styleplugins/chameleon/menu_shadow.svg styles/images/woodbackground.png
       styles/images/woodbutton.png tests/iconengines/builtinengine/icons/actions/icon_Layout_16px.svg
       tests/iconengines/svgiconengine/icon_window_16px.svg
Copyright: None
License: CC0-1.0

# sh
Files: tests/test-recoverage-qmake.sh
Copyright: None
License: CC0-1.0

# ignore git
Files: .git*
Copyright: None
License: CC0-1.0

# xml toml json conf yaml
Files: *.xml *.toml *.json *conf *.yaml
Copyright: None
License: CC0-1.0

# rpm
Files: rpm/*
Copyright: None
License: CC0-1.0

# debian
Files: debian/*
Copyright: None
License: CC0-1.0

# Arch
Files: archlinux/*
Copyright: None
License: CC0-1.0
```

### 无需版权声明文件的处理方案

有部分文件是无需版权声明的，比如说某些资源文件脚本文件或者序列化的文件，这种文件有两种处理方式：

- `.gitignore` 是用来忽略某些与项目无关的文件和编译过程中产生的文件，同样的如果reuse也会忽略在`.gitignore`中标记的文件
- `dep5` 在执行项目init之后，会在项目目录下产生`.reuse/dep5` 文件，打开后能看到官方给的范例，如果一个文件不需要特殊版权声明则使用`CC0-1.0`（会放弃对此文件的所有版权）

## 检查项目合规

```bash
reuse lint
```

这个命令会列出项目文件数，如果项目已经合规，将会以`：-）`提示，如果项目有些文件不符合开源许可证规范将会以`：-（`提示，此时你就需要按照上面的提示进行相应的修改即可。

一般情况下，每个文件都需要有与之相关的版权和许可信息。REUSE规范详细说明了几种方法。总的来说，有这些方法：

- 将标签放在文件的标题中。
- 将标签放置在与`.license`文件相邻的文件中。
- 将信息放入 `DEP5` 文件中。

如果发现一个文件没有与之关联的版权和/或许可信息，则该项目不合规。

## 个人建议

个人建议如果对现有项目使用reuse配置且需要保留版权的时间信息，不要使用一键配置版权头的方式，而是进行手动替换，这里推荐使用vscode的文件筛选功能，筛选出你需要添加的文件，然后cv下去。在配置dep5文件的时候，一定小心通配符的范围，不宜过大。并且对于第三方版权文件一定得小心规避，同样需要完整保留第三方版权信息。

## 统信软件常见Dep5书写参考

参考详细见：

```text
Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
Upstream-Name: dtkgui
Upstream-Contact: UnionTech Software Technology Co., Ltd. <>
Source: https://github.com/linuxdeepin/dtkgui

# ci
Files: .github/* .gitlab-ci.yml
Copyright: None
License: CC0-1.0

# gitignore #
Files: .gitignore
Copyright: None
License: CC0-1.0

# json conf yaml
Files: *.json *conf *.yaml
Copyright: None
License: CC0-1.0

#interface
Files: src/util/D* src/kernel/D* src/filedrag/D*
Copyright: None
License: CC0-1.0

# rpm
Files: rpm/*
Copyright: None
License: CC0-1.0

# debian
Files: debian/*
Copyright: None
License: LGPL-3.0-or-later

# Arch
Files: archlinux/*
Copyright: None
License: CC0-1.0

# README&doc
Files: README.md doc/src/*.qdoc
Copyright: None
License:  CC-BY-4.0

# DBus
Files: src/dbus/*.xml
Copyright: None
License: CC0-1.0

# Project file
Files: *.pro *.prf *.pri *.qrc *CMakeLists.txt
Copyright: None
License: CC0-1.0

# svg
Files: src/util/icons/actions/*  src/util/icons/icons/* src/util/icons/texts/*
  tests/images/logo_icon.svg
Copyright: UnionTech Software Technology Co., Ltd.
License: LGPL-3.0-or-later
```

## 常见开源协议选择

一般情况下对于文档信息使用`CC-BY-4.0` 对于我们的一般开源项目使用`LGPL-3.0-or-later`许可证，如果遇到第三方文件，则是需要保留原有许可证，保留原版版权声明。
