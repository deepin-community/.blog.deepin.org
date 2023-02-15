---
title: "修理 FreeRDP"
date: 2018-12-17T14:00:00+08:00
draft: false
authors: ["hualet"]
tags: ["RDP"]
---

头段时间入了一个大坑儿，大概被坑了有一个月之久，出来之后同事还不忘嘲讽一番：”这么个事情就搞了一个月，看吧，你果然是老了“。听了这句话，心里真是百般滋味，但转念一想，”我年轻的时候做事好像也不怎么快“，顿时也就释怀了 🙂

<!--more-->

这个坑就是”给 FreeRDP 的 RAIL 模式添加托盘支持“，当然，跟所有的需求一样，这么具有总结性而又直指根源的需求描述，绝对不是它最原始的模样——我刚接到这个坑的时候，它是这样的：FreeRDP 的 RAIL 模式下，应用的托盘在我们 DDE 下不显示。请注意这里说得是**不显示**，而不是后来发现的**压根儿没有支持**！

### FreeRDP

说到这，可能有读者还不了解 FreeRDP 和 RAIL，所以先简单介绍一下。

RDP 其实是一个协议名称，全称 Remote Desktop Protocol（远程桌面协议），是微软公司开发的一套用于远程桌面展示和操作的协议，FreeRDP 就是它在开源世界的实现咯。而 RAIL 的全称是 Remote Application Integrated Locally （远程应用本地集成），其实就是非常类似大家熟悉的虚拟机的”无缝模式“，通过将应用的显示跟本地环境相融合，让用户完全感受不到这个应用其实不在本机运行——就是这么一种技术。

问题也就出在这，我当时第一反应是这么老的技术实现肯定比较完整了，托盘没有显示出来应该是跟 DDE 的兼容性有点小问题，稍微修一下就完了，三下五除二的事情，所以满口答应了下来……

## 经过

既然答应了，硬着头皮也要顶下去的。何况调 BUG 这种事情——不管是不是我们自己的问题——在深度都是家常便饭。慢慢地，调各种项目的 BUG 竟然成了我的一种乐趣——每次开始接手一个新的项目的时候，我都把自己当成了福尔摩斯或者胡八一，或者也可以是其他全世界最聪明的那类人 ?，在通过代码找寻问题线索的过程中，慢慢成为这个项目世界中的主宰，解开真相……

额……不好意思，白日梦又发作了一会儿。总之，这次也不例外，而且刚好这次在调问题的过程中有记录几个关键环节，所以打算把中间的过程写成日记性质的记录，看看能不能有更好的阅读效果：

### 2018-11-14

从”沈老板“那收到需求，说 FreeRDP 在我们系统上有问题，应用的托盘显示不出来，QQ之类的程序关闭了窗口以后就没办法显示出来了，无法使用。这丫的又拿刘老大来压我……呵呵，想削他。不过看在他快要当爸爸的份上，还是算了。问了下时间要求，大概需要两周左右有初步的结果。不过我自己最近没有什么时间，先把锅丢给了印象中还比较熟悉网络协议的 @Blumia 同学。

### 2018-11-15

从 @Blumia 那收到反馈，可能 FreeRDP 没有实现托盘图标这部分的功能，我怕他一个人搞不定，简单翻了翻 FreeRDP 的项目 wiki 和 RDP 的一些介绍，给了他，让他先帮忙找一下需要补充实现部分的代码结构。

### 2018-11-16

没时间处理。

@Blumia 搭了测试环境。

**中间几天两****个****人都没有时间处理 FreeRDP****的****事情。**

### 2018-11-22

留了少部分时间，看了 FreeRDP 的代码，大概找到了托盘图标相关处理应该在的位置。

* RAIL 主要接口的实现都在 `xf_rail.c` 中。
* 托盘图标相关的处理在 `xf_rail_no``t``ify_icon_*` 相关的函数，这些函数在 `xf_rail_re``gi``s``t``er_``u``pdate_call``b``a``c``ks` 里面被注册到 `rdpWind``o``wUpdate` 对象上。
* 部署了一份测试服务器的虚拟机。
* 在 `wf_rail.``c` 中发现一个 `PrintRailI``c``onInfo` 函数，放在 `xf_``r``ail_notify_i``c``on_common` 中打印了获取到的图标的信息，发现能正常获取一些图标的数据。
* xf 应该是 x11 freerdp 的缩写，而 wf 应该是 windows freerdp 的缩写。
**中间又是几天没有时间处理 FreeRDP 的事情。**

### 2018-11-27

有半天的时间看 FreeRDP 的代码，同时跟 FreeRDP 的邮件列表发了[邮件](https://sourceforge.net/p/freerdp/mailman/message/36477842/)询问相关技术问题，主要是为了验证自己的想法，没有指望有回复或者什么比较大用处的信息，只是希望如果自己想法是错的，有人及时纠正一下。

* 图标显示的问题不打算优先处理，现在的问题变成如何让服务端知道了本地用户点了托盘图标。
* 搜了一下 event 相关的文件，发现 `x``f_event.c` ，怀疑 X 相关的事件都是在这里面处理的，这个也不用急着去证明，先看看 client 怎么让 server 感知本地的事件。
* 没有头绪，只好看了一下 RAIL 的 [主要协议](https://msdn.microsoft.com/en-us/library/cc242568/)，发现 `Cli``e``nt Notify Ev``en``t`。
* 怀疑托盘图标在 client 端（本地端）的事件是通过 ClientNotifyEvent 发送给 server 端的。
* ClientNotifyEvent 相关：
    * `rail_main.``c` 中的 `Vi``r``tualChann``e``lE``n``tryEx` 应该是 RDP 中 RAIL 相关的 channel 处理的函数。
### 2018-11-28

上午继续看了 FreeRDP 的代码。

* `V``i``rtualCha``n``nelEntryEx` 中给 `RailClientContext` 设置的哪些成员函数，有些函数（Server开头的）都是需要真正的 client 去实现的，Client 开头的函数（包括 ClientNotifyEvent）都是默认有实现，但是这些 Client 开头的函数都是在哪调用的呢？
找到重要线索：

```plain
/**
* The position of the X window can become out of sync with the RDP window
* if the X window is moved locally by the window manager.  In this event
* send an update to the RDP server informing it of the new window position
* and size.
*/
void xf_rail_adjust_position(xfContext* xfc, xfAppWindow* appWindow)
{
RAIL_WINDOW_MOVE_ORDER windowMove;

if (!appWindow->is_mapped || appWindow->local_move.state != LMS_NOT_ACTIVE)
    return;

/* If current window position disagrees with RDP window position, send update to RDP server */
if (appWindow->x != appWindow->windowOffsetX ||
    appWindow->y != appWindow->windowOffsetY ||
    appWindow->width != appWindow->windowWidth ||
    appWindow->height != appWindow->windowHeight)
{
    windowMove.windowId = appWindow->windowId;
    /*
     * Calculate new size/position for the rail window(new values for windowOffsetX/windowOffsetY/windowWidth/windowHeight) on the server
     */
    windowMove.left = appWindow->x;
    windowMove.top = appWindow->y;
    windowMove.right = windowMove.left + appWindow->width;
    windowMove.bottom = windowMove.top + appWindow->height;
    xfc->rail->ClientWindowMove(xfc->rail, &windowMove);
}
}
```
* 其中有主动调用 `RailClientContext` 的 `ClientWindowMove` 函数。这个函数又是 `xf_``e``v``en``t.``c` 中 `xf_``e``v``en``t_C``o``nfig``u``r``e``Notify` 有调用，再加上这个函数的注释说明，差不多能证明所有的 X事件相关的都是在 `xf_event.c` 中处理的，跟之前的猜测一致。
* 那样的话如果想发送事件到 server，应该就是在 `xf_event.c` 中收到我们自己创建的托盘图标的点击事件后，发送一个 ClientNofityEvent 。
尝试在收到 `xf_r``a``il_``no``ti``f``y_``i``con_u``p``d``a``te` 的时候主动调用一次 `Cli``ent``NotifyEvent` 看看会发生什么。

```plain
RAIL_NOTIFY_EVENT_ORDER notifyEvent;
notifyEvent.windowId = orderInfo->windowId;
notifyEvent.notifyIconId = orderInfo->notifyIconId;
notifyEvent.message = NIN_SELECT;

xfContext* xfc = (xfContext*) context;
xfc->rail->ClientNotifyEvent(xfc->rail, &notifyEvent);
```
* 选择 message 为 `NIN_SELECT` 是因为根据 `r``ail.h` 里面仅有的零星注释，只能推测这个可能是针对托盘的。
* 试了下，没有任何反应。尝试换成 NIN_KEYSELECT ，更不行。
* 硬着头皮又翻了一下协议，发现 `Notificat``i``o``n``I``c``on Info``r``mation` 这段，随便翻了一下，看起来没有有用信息。
* 偶发奇想搜了一下 select 关键字想看一下这个到底是什么意思，偶然发现 `Cli``en``t Notify E``ve``nt PDU` 这一节（能跟源码 `r``ai``l``.h` 里面的一些注释对应上），里面有 WM_LBUTTONDOWN 、 WM_LBUTTONUP 等针对托盘图标的动作定义。
* 真是对自己做事情毛毛躁躁的行为无语了，要不是心血来潮，差点就错过这么重要的信息。
* 把 message 改成 `WM_LBUTTONDOWN` 和 `WM_LBUTTONUP` ，满怀期待。
* 测试还是无效果……仍旧不死心，怀疑测试程序（@Blumia 同学搞的一个音乐程序）的稳健程度。
* 使用 TIM 再试，还是不行，不能唤出主窗口。
* 感觉走进了死胡同。
### 2018-11-29

继续看 FreeRDP 的问题，主窗口隐藏后不能显示的问题太奇怪了，得找一个简单点的程序，排除复杂影响。

* 让 @zccrs 写了一个简单的窗口程序，定时隐藏、显示窗口。
* 发现程序窗口隐藏后无法再显示出来……
* 赶紧给上游报了一个 [issu](https://github.com/FreeRDP/FreeRDP/issues/5078)[e](https://github.com/FreeRDP/FreeRDP/issues/5078) ，希望上游能修复。但是也不能期望上游很快能修复这个问题，所以自己还是尝试看代码……
* 上游回复还挺快的，但是对方好像是 FreeRDP 目前的维护者，说自己对 RAIL 这部分协议本身还不是特别熟悉。
* 不能依赖的上游不是好上游，继续看代码，发现在窗口的显示隐藏主要是通过 `WINDOW_STATE_ORDER` 中的 `s``h``o``wState` 控制的，处理的函数是 `xf_``r``ail_window_``c``o``m``mon` ，里面调用了 `xf_ShowWin``d``ow` 这个函数，但是这个函数在该显示窗口的时候只是调整了一下窗口的最大化、最小化状态，并没有 Map 这个本地窗口。
* 加了 `XMapW``in``dow(xf``c``->di``s``pl``a``y,` `appWi``n``dow->ha``n``dl``e``)` 这行，满心期待 bugfix。
* 编译代码测试，发现窗口连关闭都不能关闭了……
* xfContext 的 appWindow 是本地窗口的一个抽象表示。
### 2018-12-03

觉得这个事情没有什么太大的希望了，不过既然已经知道托盘图标的显示方式和事件的发送，但是没有实际实现，到时候”沈老板“来问，也不好说都是在脑子里，干脆先把之前测通但是没有实现的内容实现一下。

* 托盘窗口加好了，事件也都加上了。
* 眼看着都快要完美了，就差那么一点问题没有解决，实在是不甘心，继续死磕那个问题。
* 尝试了各种手段调试，跟整个程序的命令传递，都没有能解决问题。
### 2018-12-04

调试了一天，一遍又一遍看窗口事件，一点一点排除事件处理函数，终于发现了上游犯的一个低级错误，我很怀疑当时作者有没有测试一下 🙁

做了修复，提交了 [PR](https://github.com/FreeRDP/FreeRDP/pull/5097/)，并且顺利合并。

心情终于舒畅了。

**中间有事请假一天**

### 2018-12-06

托盘图标也画上了，不过怎么感觉颜色有点偏。

调了一下颜色的格式（RGBA -> BGRA），图标显示正常了。

事情终于告一段路了。

## 结束

折腾了这么长时间，事情终于搞定了，这应该是最近一年里面时间拉的最长的 BUG 了。

实现算是完了，也能使用。但是还有一些细节没有特别完善，已提交提交到上游 [一个新](https://github.com/FreeRDP/FreeRDP/pull/5110)[的](https://github.com/FreeRDP/FreeRDP/pull/5110)[PR](https://github.com/FreeRDP/FreeRDP/pull/5110) ，希望能早日合并造福一方用户。

## 感想

感觉我之前对 wine 有偏见，一直比较拒绝使用（或者大量使用）wine 的东西，但是实际上在修复 FreeRDP 的过程中，我竟然觉得这也是一种不错的解决方案……仔细想想，还是 wine 方便一点，至少不需要依赖一个服务端。

准备入坑 wine 啦 ~(≧▽≦)/~

 

