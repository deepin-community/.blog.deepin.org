---
title: 'QDir 和 std::filesystem 的简单对比'
date: 2022-03-04 10:11:45
tags:
  - C++
  - Qt
draft: false
authors: ["justforlxz"]
---

作为一名使用 Qt 的开发人员，Qt 为我提供了大量好用的基础设施，例如广受好评的 QString、QNetwork之类的，这是 Qt 平台为我提供的帮助，我只需要在这个平台上开发就足够了。

同样作为一名 C++ 开发人员，C++ 标准库也是我需要用的基础设施，但是标准库提供的功能就不如 Qt 了，最令人诟病的就是 C++ 的 std::string，业内充斥着对 std::string 的不屑与谩骂。

但是这一情况将会在 C++ 20 标准后改善，这里不具体展开，将来我会准备写一份 C++ 20 的功能介绍。

<!--more-->

今天在开发项目的时候，为了节省资源，我仅使用标准库完成开发，缺少了 Qt 平台为我提供的有利帮助，项目开发的难度瞬间增加，在此期间，我发现了 C++ 的目录操作似乎能比的上 Qt 提供的 QDir封装，我打算在本篇文章中介绍一下这两者的不同。

## QDir

Qt 的基本思路是继承大于组合，所以 Qt 为我们提供的都是各种继承的类，在 Qt 中，我们使用 QDir 类进行目录操作。

QDir 类使用相对或绝对文件路径来指向一个文件或目录。如果总是使用 “/” 作为目录分隔符，Qt 将会把你的路径转化为符合底层的操作系统的。

QDir 类由于不涉及 IO 的具体操作，所以没有继承自 QObject 或者其他 QIO 的类。

QDir 提供了非常多的方法，可以方便的获取目录的名称、绝对路径、相对路径、设置目录过滤器等。

一个基本的用法如下:

```Qt
QDir dir("/tmp");
if (!dir.exists()) {
    return
}

for (const QFileInfo& item : dir.entryInfoList()) {
    if (item.isDir()) {
        // ...
      }
}

dir.setFilter(QDir::Files | QDir::Hidden | QDir::NoSymLinks);
dir.setSorting(QDir::Size | QDir::Reversed);

const QFileInfoList& list = dir.entryInfoList();
for (const QFileInfo& item : list) {
    // ...
}
```

在上面的示例代码中可以看到，QDir 整体是非常符合面向对象的，我们使用 QDir 对象，对目录执行各种操作，涉及到具体的文件（目录是特殊的文件），我们可以使用 QFileInfo 类获取具体文件的信息。

除了以上演示涉及到的方法，QDir 还有很多其他方法，使用 `count()` 方法统计目录内文件和目录的数量，使用 `remove()` 方法删除指定的目录，这里就不一一列举了。

QDir 支持设置多种过滤，过滤(Filter) 和 排序(Sorting) 都会影响到 entryInfoList 方法返回的内容。

QDir 接受的过滤器枚举:

| 枚举 | 枚举值 | 描述 |
| --- | --- | --- |
| QDir::Dirs | 0x001 | 列出与过滤器匹配的目录。 |
| QDir::AllDirs | 0x400 | 列出所有目录；即不要将过滤器应用于目录名称。 |
| QDir::Files | 0x002 | 列出文件。 |
| QDir::Drives | 0x004 | 列出磁盘驱动器（在Unix下忽略）。 |
| QDir::NoSymLinks | 0x008 | 不要列出符号链接（不支持符号链接的操作系统忽略）。 |
| QDir::NoDotAndDotDot | NoDot | NoDotDot | 不要列出特殊条目“.”和“..”。 |
| QDir::NoDot | 0x2000 | 不要列出特殊条目“.”。 |
| QDir::NoDotDot | 0x4000 | 不要列出特殊条目“..”。 |
| QDir::AllEntries | Dirs | Files | Drives | 列出目录、文件、驱动器和符号链接（除非您指定系统，否则不会列出损坏的符号链接）。 |
| QDir::Readable | 0x010 | 列出应用程序具有读取访问权限的文件。可读值需要与Dirs或Files合并。 |
| QDir::Writable | 0x020 | 列出应用程序具有写入访问权限的文件。可写值需要与Dirs或Files合并。 |
| QDir::Executable | 0x040 | 列出应用程序具有执行访问权限的文件。可执行值需要与Dirs或Files合并。 |
| QDir::Modified | 0x080 | 仅列出已修改的文件（在Unix上忽略）。 |
| QDir::Hidden | 0x100 | 列出隐藏的文件（在Unix上，以“.”开头的文件）。 |
| QDir::System | 0x200 | 列出系统文件（包括Unix、FIFO、套接字和设备文件；在Windows上，包括.lnk文件） |
| QDir::CaseSensitive | 0x800 | 过滤器应该区分大小写。 |

QDir 接受的排序枚举:

| 枚举 | 枚举值 | 描述 |
| --- | --- | --- |
| QDir::Name | 0x00 | 按名称排序。 |
| QDir::Time | 0x01 | 按时间（修改时间）排序。 |
| QDir::Size | 0x02 | 按文件大小排序。 |
| QDir::Type | 0x80 | 按文件类型（扩展名）排序。 |
| QDir::Unsorted | 0x03 | 不要排序。 |
| QDir::NoSort | -1 | 默认情况下未排序。 |
| QDir::DirsFirst | 0x04 | 先放目录，然后放文件。 |
| QDir::DirsLast | 0x20 | 先放文件，然后放目录。 |
| QDir::Reversed | 0x08 | 颠倒排序顺序。 |
| QDir::IgnoreCase | 0x10 | 不区分大小写的排序。 |
| QDir::LocaleAware | 0x40 | 使用当前区域设置对项目进行适当排序。 |

## std::filesystem

上面介绍了 Qt 的设计风格，而标准库的设计风格是组合大于继承，标准库提供各种非常具体的类或者函数，将一个系列的操作拆分为各个子项，最终完成任务。

这里使用一个小例子来说明一下:

```cpp
std::filesystem::path tmp{"/tmp"};
if (!std::filesystem::exists(tmp)) {
    std::cout << "目录不存在" << std::endl;
    return;
}

const bool result = std::filesystem::create_directories(tmp);
if (!result) {
    std::cout << "目录创建失败，也许是已经存在。" << std::endl;
}

for (auto const& dir_entry : std::filesystem::directory_iterator(tmp)) {
    if (dir_entry.is_directory()) {
        // ...
    }
}

if (std::filesystem::is_directory(tmp)) {
    // ...
}
```

可以看到，C++ 标准库使用多个类共同完成了对目录的检查和遍历，这种基于组合的方式可以带来更多的灵活性，如果需要对某个部分进行修改，只需要继承特定类就可以完成，如果是 Qt 的 QDir，则不是很轻松。

在使用标准库的时候需要注意的是，标准库通常不会约束使用者，使用当前的例子举例，QDir 提供了过滤器枚举，可以帮助开发者简单的实现文件过滤功能，但是 std::filesystem 则不提供这种接口，标准库提供了机制，但是不提供策略，开发者需要使用标准库提供的各种接口，**组合** 出自己的业务，所以如果想要使用标准库的时候也能实现过滤和排序，就只能自己提供相应的操作。

```cpp
std::filesystem::path tmp{ "/tmp" };
std::vector<std::filesystem::directory_entry> list;
std::vector<std::string> filter{ ".jpg", ".txt" };
std::filesystem::directory_iterator iter{ tmp };
std::copy_if(iter.begin(),
             iter.end(),
             std::back_inserter(list),
             [filter](std::filesystem::directory_entry entry) {
                return !entry.is_directory() && 
                       std::find_if(filter.begin(),
                                    filter.end(),
                                    [entry](std::string s) {
                                      return entry.path().extension() == s;
                                    }
                       );
              }
);
```

~~可以看到，代码变得异常丑陋（逃~~

标准库提供了一些比较方便的函数，例如 std::copy_if、 std::back_inserter 和 std::find_if 等，还有一个较为常用的 std::transform。标准库提供了迭代器抽象，这样我们可以使用迭代器对象和迭代器算法，方便的进行各种遍历、复制和转换。

从上面的例子可以看出，Qt 确实为开发者提供了很好的帮助，这是 Qt 作为一个平台力所能及的工作，当然，即使我们使用 Qt，也还是可以写出上面一样的代码。

## 总结

以上就是简单的 QDir 和 std::filesystem 的不同，综合来看，Qt 库和标准库其实各有优缺，他们的目标和面向的开发者是不同的，Qt 最近一直在尝试使用标准库的内容来代替自己的一部分组件，最新的Qt6 就已经升级到了 C++ 17 标准，将 qSort 宏改为了使用 std::sort，也许在不久的将来，我们再也不用为使用 Qt 还是标准库而争论或者站队。

> **引用资料**
> [std::filesystem https://en.cppreference.com/w/cpp/filesystem](https://en.cppreference.com/w/cpp/filesystem)
> [QDir https://doc.qt.io/qt-5/qdir.html](https://doc.qt.io/qt-5/qdir.html)

原文链接：[https://blog.justforlxz.com/2022/03/04/qdir-stdfilesystem/](https://blog.justforlxz.com/2022/03/04/qdir-stdfilesystem/)
