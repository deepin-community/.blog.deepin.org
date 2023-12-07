---
title: TLP 电源管理简述
date: 2023-12-07
draft: false
authors: ["longlong"]
tags: ["Tools"]
---

## 简述
在上一篇[浅析Linux电源配置](https://blog.deepin.org/posts/analyzing-the-linux-power-configuration/)之后，我们一直在深入探索如何进一步优化我们系统的续航和性能表现，今天它来了：

TLP 是适用于 Linux 的功能丰富的命令行实用程序，无需深入研究技术细节即可节省笔记本电脑电池电量。之前我们的系统使用的laptopmode,但是相较于TLP还有有部分劣势：比如tlp脚本是被动唤醒，可以以较小的开销完成电源管理相关内容。而且TLP文档支持非常完善，所以可以方便用户自行调整相关配置。以下是TLP官方文档内容的和我自己的理解的结合，各位系统用户可以结合自己的实际情况diy自己的电源策略文件，也可以将好的电源配置在[deepin 论坛](https://bbs.deepin.org/)中分享。

<!--more-->

## 工作原理

-   TLP 基本上所做的是调整影响功耗的内核设置，内核态的配置文件存储在RAM中，所以并不具备持久性。TLP将配置存储在用户态中，在内核启动时对其进行配置
-   TLP 处理的大多数内核设置都作为 sysfs 节点导出到用户空间，即 /sys/ 下的文件。`tlp-stat` 的输出将显示路径。
-   TLP 提供两组独立的设置，称为配置文件，一组用于电池 （BAT），另一组用于交流操作。这意味着 TLP 不仅在启动时，而且在每次电源更改时都必须应用适当的配置文件（可以据此实现AC BT切换电源调度状态）

### TLP触发事件（信号）

-   充电器插入（交流供电）：应用AC配置文件
-   充电器已拔下（电池供电）： 应用BAT配置文件
-   已插入 USB 设备：激活设备的 USB 自动挂起模式（可以在配置文件设置例外或拒绝连接）
-   系统启动（boot）：应用与当前电源 AC/BAT 相对应的设置配置文件。应用充电阈值并根据您的个人设置切换蓝牙、Wi-Fi 和 WWAN 设备（在默认配置中禁用）
-   系统关机 (power off)：保存或切换蓝牙、Wi-Fi 和 WWAN 设备状态，并根据您的个人设置禁用 USB 自动挂起（在默认配置中禁用）
-   系统重启： 相当于关机再启动
-   系统挂起到 ACPI 睡眠状态 S0ix（空闲待机）、S3（挂起到 RAM）或 S4（挂起到磁盘）：保存蓝牙、Wi-Fi 和 WWAN 设备状态，并根据您的个人设置关闭可移动光盘驱动器的电源（在默认配置中禁用）。
-   系统从 ACPI 睡眠状态 S0ix（空闲待机）、S3（挂起到 RAM）或 S4（挂起到磁盘）恢复： 应用与当前电源 AC/BAT 相对应的设置配置文件。恢复充电阈值以及蓝牙、Wi-Fi 和 WWAN 设备状态，具体取决于您的个人设置（在默认配置中禁用）。
-   LAN、Wi-Fi、WWAN 连接/断开连接或笔记本电脑插接/未插接：根据您的个人设置启用或禁用内置蓝牙、Wi-Fi 和 WWAN 设备（在默认配置中禁用）

> 除了上述事件之外，TLP 不会对设置进行动态或自适应更改
> 特别是，TLP 绝不会因 CPU 负载、电池电量或其他原因而调整设置（如果我们需要去实现这一部分，则可以，则可以通过添加一个信号的方式来实现）

## 安装

`sudo apt install tlp`

## 使用

### 启动

安装后TLP将在系统启动的时候自动启动，如果你不想重启系统，可以使用`sudo tlp start`来启动tlp，也可以使用此命令来应用更改。

### 状态

`tlp-stat -s`  TLP是bash脚本，所以不存于daemon进程

### 命令行

#### TLP：

`sudo tlp bat` 应用电池配置文件并进入手动模式  手动模式意味着对电源的更改将被忽略，直到下一次重新启动或发出 tlp start 以恢复自动模式

`sudo tlp ac`应用交流配置文件并进入手动模式

`sudo tlp usb` 对所有的ubs设备应用自动挂起

`sudo tlp bayoff` 关闭 MediaBay/Ultrabay 中的光驱电源

`sudo tlp setcharge [<START_CHARGE_THRESH> <STOP_CHARGE_THRESH>] [BAT0|BAT1|BAT<x>|CMB0|CMB1]` 可以设定对指定电池开始充电百分比和结束充电的百分比，以达到养护电池的目的（如果不带参数 会重置电池管理方案）（命令只能暂时更改，如果需要持久化更改 需要修改配置文件）

`sudo tlp fullcharge [BAT0|BAT1|BAT<x>|CMB0|CMB1]` 设定电池充满

`tlp diskid` 显示已经配置驱动器的磁盘ID

以下部分为ThinkPad专属

`sudo tlp chargeonce [BAT0|BAT1]` 将电池充电至停止充电阈值一次，这个阈值是使用setcharge设置的

`sudo tlp discharge [BAT0|BAT1]` 让电池在交流电源下完全放电

`sudo tlp recalibrate [BAT0|BAT1]`校准电池

#### TLP-RDW

`sudo tlp-rdw [ enable | disable ]` 启用或关闭无线电管理功能

```bash
bluetooth [ on | off | toggle ]
nfc [ on | off | toggle ]
wifi [ on | off | toggle ]
wwan [ on | off | toggle ]
```

启用、禁用、切换或检查内置蓝牙、NFC、Wi-Fi 和 WWAN（3G/UMTS、4G/LTE 或 5G）无线电的状态，如果不带参数则为当前硬件状态（硬件需要支持rfkill）

#### TLP-STAT

`sudo tlp-stat` 查看TLP配置信息，系统信息和内核省电设置以及电池数据

`sudo tlp-stat [-b /--battery]` 查看电池信息，部分电池加`-v`参数可以查看电压

`sudo tlp-stat [-c /--config]`查看配置信息

`sudo tlp-stat --cdiff` 查看默认配置和用户配置之间的差异

`sudo tlp-stat [-d /--disk]` 查看硬盘配置信息

`sudo tlp-stat [-e/ --pcie]` 查看Pcie配置信息

## 配置

TLP最重要的就是其配置文件，可以说，TLP是否节电的关键。 TLP 使用两个根据电源自动应用的设置配置文件：

-   以`_AC`结尾的参数在连接交流电源的时候生效
-   以`_BAT`结尾的参数在使用电池的时候有效
-   既不以`  _AC  `结尾也不以 `_BAT` 结尾的参数适用于这两个配置文件

### 配置文件

按指定顺序从以下文件中读取设置：

-   Intrinsic defaults 固有默认值（这个配置为TLP自带，不可被更改）
-   `/etc/tlp.d/*.conf`：插入式自定义片段，按词法（字母顺序）顺序读取，不过建议可以使用一般配置命名方法（00\_xxxx.conf)
-   `/etc/tlp.conf`：用户配置

> 如果多个参数相同，但在同一文件中也存在相同的参数，则最后一个匹配项优先，这也意味着，`/etc/tlp.conf` 中的参数将覆盖其他任何内容，因为它是最后读取的
> 默认的`/etc/tlp.conf `中的所有参数都被禁用，删除前导 # 以激活您的更改
> /etc/tlp.d/ 目录中的配置文件由用户创建：
> \* 文件名必须以 .conf 结尾，否则文件将被忽略
> \* 00-template.conf 作为示例提供

### 参数默认值

配置中有两种参数，一种是具有默认值的，会在本文档中说明，并且在`/etc/tlp.conf`中有`Default`前缀。还有一种没有默认值的。

### 参数语法

配置文件由参数和注释行组成。

#### 参数行

```bash
PARAMETER=value
```

如果value包含空格，则需要使用双引号

```bash
key="111 1111 1111"
```

#### 注释行

以`#`开头，在1.6版本后可以在参数行后接`#`作为注释

#### 禁用功能

-   没有默认值：使用注释或者删除即可
-   有默认值： 赋空值即可 eg：`key=""`

#### 使用`+=`追加配置

和bash的环境变量一样，支持使用`+=`作为追加配置

> 使用root权限编辑配置文件，在保存更改后可以使用重启，拔插ac电源或者使用`sudo tlp start`命令激活配置

### 配置详解

#### 基础操作

| 参数名称                     | 默认参数值 | 描述                                                                                                                     |
| ------------------------ | ----- | ---------------------------------------------------------------------------------------------------------------------- |
| TLP\_ENABLE              | 1     | 设置为0可禁用TLP（需要重新启动）。未配置时的默认值：1                                                                                          |
| TLP\_WARN\_LEVEL         | /     | 控制如何发出有关无效设置的警告：&#xA;0 - 禁用&#xA;1 - 向系统日志/日志报告后台任务（启动、恢复、电源更改）&#xA;2 - 外壳命令向终端报告（标准）&#xA;3 - 1和2的组合。未配置时的默认值：3         |
| TLP\_DEFAULT\_MODE       | /     | 定义TLP的默认操作模式（AC或BAT），以防无法检测到电源。仅涉及某些台式机和嵌入式硬件。                                                                         |
| TLP\_PERSISTENT\_DEFAULT | 0     | 选择如何确定操作模式：&#xA;0 – 根据实际电源应用设置配置文件（默认）&#xA;1 – 始终使用TLP\_DEFAULT\_MODE设置。未配置时的默认值：0                                     |
| TLP\_PS\_IGNORE          | /     | 确定工作模式时要忽略的电源等级：（用作错误检测到操作模式 AC 或 BAT 的笔记本电脑的解决方法）&#xA;AC&#xA;BAT&#xA;USB - 仅限版本 1.4 及更高版本。仅限版本 1.4 及更高版本：输入多个类，以空格分隔。 |

#### 音频

| 参数名称                           | 默认参数值 | 描述                                                                              |
| ------------------------------ | ----- | ------------------------------------------------------------------------------- |
| SOUND\_POWER\_SAVE\_ON\_AC/BAT | 1     | 设置为0可禁用音频省电模式（需要重新启动）。未配置时的默认值：1（AC），1（BAT）- 版本 1.4 及更高版本，0（AC），1（BAT）- 版本 1.3。 |
| SOUND\_POWER\_SAVE\_CONTROLLER | Y     | Y – 关闭控制器和声音芯片的电源&#xA;N – 控制器保持活动状态。未配置时的默认值：Y。                                 |

注释： `SOUND_POWER_SAVE_ON_AC/BAT` 指的是`SOUND_POWER_SAVE_ON_AC` 和 `SOUND_POWER_SAVE_ON_BAT`

#### 电池保养

| 参数名称                           | 参数值 | 描述                        |
| ------------------------------ | --- | ------------------------- |
| START\_CHARGE\_THRESH\_BAT\<x> | 75  | 电池充电水平低于该水平，连接充电器时将开始充电。  |
| STOP\_CHARGE\_THRESH\_BAT\<x>  | 80  | 电池充电水平，超过该水平，充电器连接时充电将停止。 |

这些参数用于设置笔记本电脑主/内部电池（BAT0）和辅助电池（BAT1）的充电阈值。启动充电阈值表示在连接充电器时，电池充电水平低于该值时将开始充电。停止充电阈值表示在充电器连接时，电池充电水平超过该值时将停止充电。这些阈值始终具有较低的可用电池容量，因此默认情况下禁用这些设置，并且必须通过删除前导 # 来显式启用这些设置。

#### 光驱

| 参数名称                      | 默认参数值 | 描述                                                 |
| ------------------------- | ----- | -------------------------------------------------- |
| BAY\_POWEROFF\_ON\_AC/BAT | 0     | 控制光驱在交流电源和电池供电时是否关闭电源。&#xA;1：保持光驱开启状态&#xA;0：关闭光驱电源 |
| BAY\_DEVICE               | sr0   | 指定光驱设备。                                            |

#### 硬盘

| 参数名称                                | 默认参数值                             | 描述                                                                                                                                                                                                                                                  |
| ----------------------------------- | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| DISK\_DEVICES                       | "nvme0n1 sda"                     | 定义参数作用的磁盘设备。多个设备用空白分隔。                                                                                                                                                                                                                              |
| DISK\_APM\_LEVEL\_ON\_AC/BAT        | "254 254"（AC）&#xA;"128 128" (BAT) | 设置“高级电源管理级别”。可能的值介于1和255之间。&#xA;1 – 最大省电/最低性能 – 重要提示：此设置可能会导致磁盘驱动器磨损增加，因为读写磁头卸载过多&#xA;128 – 省电和磨损之间的折衷（电池的 TLP 标准设置）&#xA;192 – 防止某些 HDD 的磁头过度卸载&#xA;254 – 最小省电/最大性能（交流电的 TLP 标准设置）&#xA;255 – 禁用 APM（某些磁盘型号不支持）&#xA;keep – 用于跳过特定磁盘的此设置的特殊值（同义词：`_`） |
| DISK\_APM\_CLASS\_DENYLIST          | "usb ieee1394"                    | 从高级电源管理（APM）中排除磁盘类。可能的值：sata、ata、usb、ieee1394。默认为“usb ieee1394”。                                                                                                                                                                                    |
| DISK\_SPINDOWN\_TIMEOUT\_ON\_AC/BAT | "0 0"                             | 磁盘空闲时主轴电机停止的超时值。有效设置：0（已禁用）、1..240（5秒到20分钟）、241..251（30分钟到5.5小时）。                                                                                                                                                                                   |
| DISK\_IOSCHED                       | "keep keep"                       | 两个参数为&#xA;多队列 （blk-mq） 调度器：`mq-deadline` 、`none`、`kyber`、`bfq`、`keep`&#xA;单队列调度程序：`deadline`、`cfq`、`bfq`、`noop`、`keep`&#xA;如果未配置，默认情况下所有磁盘将使用内核的默认调度程序。                                                                                             |
| SATA\_LINKPWR\_ON\_AC/BAT           | "med\_power\_with\_dipm"          | 设置SATA链路的电源管理模式。可能的值包括：max\_performance、medium\_power、med\_power\_with\_dipm、min\_power。默认为med\_power\_with\_dipm。                                                                                                                                  |
| SATA\_LINKPWR\_DENYLIST             | "host1"                           | 从AHCI链路电源管理（ALPM）中排除SATA磁盘的主机列表。默认为空。                                                                                                                                                                                                               |
| AHCI\_RUNTIME\_PM\_ON\_AC           | "on"                              | 控制NVMe、SATA、ATA和USB磁盘以及SATA端口的运行时电源管理。可能的值包括：auto（启用）、on（禁用）                                                                                                                                                                                        |
| AHCI\_RUNTIME\_PM\_ON\_BAT          | "auto"                            | 同上                                                                                                                                                                                                                                                  |
| AHCI\_RUNTIME\_PM\_TIMEOUT          | 15                                | 磁盘或端口挂起前的不活动时间（秒）。仅在激活AHCI\_RUNTIME\_PM\_ON\_AC/BAT时有效。默认为15。                                                                                                                                                                                       |

注释：DISK\_IOSCHED 如果使用是NVME设备时，最好使用无IO调度程序来减少CPU开销（none和noop）

#### 文件系统

| 参数名称                              | 默认参数值                 | 描述                                                              |
| --------------------------------- | --------------------- | --------------------------------------------------------------- |
| DISK\_IDLE\_SECS\_ON\_AC/BAT      | 0 (AC), 2 (battery)   | 笔记本电脑模式等待磁盘空闲的秒数，然后再次将脏缓存块从 RAM 同步到磁盘。值大于0将激活内核笔记本电脑模式。请勿更改此设置。 |
| MAX\_LOST\_WORK\_SECS\_ON\_AC/BAT | 15 (AC), 60 (battery) | 将文件系统缓冲区中未保存的数据写入磁盘的超时时间（秒）。                                    |

#### 图形显卡

| 参数名称                                 | 默认参数值                           | 描述                                                                                                     |
| ------------------------------------ | ------------------------------- | ------------------------------------------------------------------------------------------------------ |
| INTEL\_GPU\_MIN\_FREQ\_ON\_AC/BAT    | 0                               | 设置 Intel GPU 的最小频率。可能的值取决于硬件。通过运行 `tlp-stat -g` 命令查看可用频率。                                              |
| INTEL\_GPU\_MAX\_FREQ\_ON\_AC/BAT    | 0                               | 设置 Intel GPU 的最大频率。可能的值取决于硬件。通过运行 `tlp-stat -g`命令查看可用频率。                                               |
| INTEL\_GPU\_BOOST\_FREQ\_ON\_AC/BAT  | 0                               | 设置 Intel GPU 的睿频频率。可能的值取决于硬件。通过运行 `tlp-stat -g` 命令查看可用频率。                                              |
| RADEON\_DPM\_PERF\_LEVEL\_ON\_AC/BAT | auto                            | 控制 AMD GPU 的动态电源管理（DPM）性能级别。支持 amdgpu（仅限 TLP 版本 1.4 及更高版本）和 radeon 驱动程序。可能的值包括 auto、low、high。默认值：auto。 |
| RADEON\_DPM\_STATE\_ON\_AC/BAT       | performance (AC), battery (BAT) | 控制 AMD GPU 的电源管理方法。可能的值包括 battery、balanced、performance。默认值：performance（AC）、battery（BAT）。               |
| RADEON\_POWER\_PROFILE\_ON\_AC/BAT   | default                         | 控制 AMD GPU 的时钟。仅在旧版 ATI 硬件上受 radeon 驱动程序支持（DPM 不可用）。可能的值包括 low、mid、high、auto、default。默认值：default。      |

这些参数允许用户调整 Intel GPU 和 AMD GPU 在交流电和电池模式下的性能和电源管理行为。在配置这些参数时，建议参考硬件规格和运行 `tlp-stat -g` 查看可用频率。

#### kernel

| 参数名称          | 默认参数值 | 描述                                                                |
| ------------- | ----- | ----------------------------------------------------------------- |
| NMI\_WATCHDOG | 0     | 激活内核 NMI 看门狗定时器。设置为 0 表示禁用，有助于节省电源。设置为 1 表示启用，对于内核调试和看门狗守护程序是相关的。 |

不建议关闭watchdog 否则可能导致内核崩溃后无法自动重启和内核调试

#### 网络

| 参数名称                  | 默认参数值     | 描述                                                                                                                                                  |
| --------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| WIFI\_PWR\_ON\_AC/BAT | off (AC), | 设置 Wi-Fi 的电源保存模式。可能的值包括 off（禁用）和 on（启用）。默认值：off（AC）、on（BAT）。                                                                                        |
|                       | on (BAT)  | 提示：支持已弃用的配置值 1=off/5=on，以实现向后兼容性。                                                                                                                   |
| WOL\_DISABLE          | Y         | 控制是否禁用 Wake-on-LAN（LAN 唤醒）。可能的值包括 Y（禁用）和 N（不禁用，保持 BIOS 默认）。默认值：Y。&#xA;注意：更改为 WOL\_DISABLE=N 后，需要重新启动才能使新设置生效（或在 shell 中使用 `sudo ethtool -s wol g`）。 |

这些参数允许用户配置Wi-Fi的电源保存模式和控制Wake-on-LAN（LAN唤醒）功能。

#### 平台

| 参数名称                          | 默认参数值       | 描述                                                                                                                  |
| ----------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------- |
| PLATFORM\_PROFILE\_ON\_AC/BAT | performance | 选择平台配置文件以控制系统的功率/性能级别、散热和风扇速度的运行特性。可能的值包括 performance、balanced、low-power。默认值：performance（AC）、low-power（BAT）。        |
| MEM\_SLEEP\_ON\_AC/BAT        | s2idle      | 选择系统挂起模式。可能的值包括 s2idle（空闲待机）和 deep（挂起到 RAM）。注意：更改挂起模式可能导致系统不稳定和数据丢失。请使用 tlp-stat -s 检查系统上不同模式的可用性。如果不确定，请坚持使用系统默认值。 |

其实如果能使用S3休眠那就更好，不过现在很多厂商并不支持S3,所以如果能用S2那就用S2吧。

#### 处理器

| 参数名称                                    | 默认参数值                                       | 描述                                                                                                                                           |
| --------------------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| CPU\_DRIVER\_OPMODE\_ON\_AC/BAT         | active (amd-pstate), active (intel\_pstate) | 选择 CPU 缩放驱动程序操作模式。配置取决于活动驱动程序：对于 amd-pstate（Active 模式），可能的值为 active 和 passive；对于 intel\_pstate（Active 模式），可能的值为 active、passive 和 guided。     |
| CPU\_SCALING\_GOVERNOR\_ON\_AC/BAT      | powersave                                   | 选择用于自动频率缩放的 CPU 缩放调节器。配置取决于活动驱动程序。可能的值包括 performance、powersave、conservative、ondemand、userspace 和 schedutil。默认值：powersave（AC）、powersave（BAT）。 |
| CPU\_SCALING\_MIN/MAX\_FREQ\_ON\_AC/BAT | 0, 9999999                                  | 设置可用于缩放调控器的最小/最大频率。可能的值取决于您的 CPU。请查阅`tlp-stat -p`的输出以获取可用频率。                                                                                 |
| CPU\_ENERGY\_PERF\_POLICY\_ON\_AC/BAT   | balance\_performance                        | 设置 CPU 能耗/性能策略。可能的值包括 performance、balance\_performance、default、balance\_power 和 power。默认值：balance\_performance（AC）、balance\_power（BAT）。      |
| CPU\_MIN/MAX\_PERF\_ON\_AC/BAT          | 0, 100                                      | 定义 Intel CPU 的最小/最大 P 状态，表示为总可用处理器性能的百分比。建议仅用于限制 CPU 的功耗。可能的值在 0 到 100 之间。默认值：0 到 100（AC）、0 到 30（BAT）。                                       |
| CPU\_BOOST\_ON\_AC/BAT                  | 1                                           | 配置 CPU “turbo boost”（Intel）或“turbo core”（AMD）功能。可能的值为 0（禁用）和 1（允许）。请注意，值为 1 不会激活提升，只是允许它。默认值：1（AC）、0（BAT）。                                   |
| CPU\_HWP\_DYN\_BOOST\_ON\_AC/BAT        | 1                                           | 配置 Intel CPU HWP 动态提升功能。可能的值为 0（禁用）和 1（启用）。要求 Intel Core i 第 6 代（“Skylake”）或更新的 CPU，在活动模式下具有 intel\_pstate 扩展驱动程序。默认值：1（AC）、0（BAT）。          |

这些参数允许用户配置 CPU 的性能和功耗特性，包括缩放驱动程序操作模式、调节器、频率范围、能耗/性能策略、P 状态范围、提升功能以及 HWP 动态提升功能。

> 部分电脑的BIOS会干预PState 所以需要检查自己的CPU是否支持

#### 无线设备

| 参数名称                                        | 默认参数值 | 描述                                                    |
| ------------------------------------------- | ----- | ----------------------------------------------------- |
| RESTORE\_DEVICE\_STATE\_ON\_STARTUP         | 0     | 在启动时从上次关机中恢复无线电设备状态。可能的值为 0（禁用）和 1（启用）。默认值：0。         |
| DEVICES\_TO\_DISABLE\_ON\_STARTUP           | ""    | 在启动时禁用内置无线电设备。可能的值包括 bluetooth、wifi 和 wwan，多个设备用空白分隔。 |
| DEVICES\_TO\_ENABLE\_ON\_STARTUP            | ""    | 在启动时启用内置无线电设备。可能的值与上述相同，用于启用在默认情况下禁用的设备。              |
| DEVICES\_TO\_ENABLE\_ON\_AC                 | ""    | 插入交流电源时启用内置无线电设备。可能的值与上述相同。                           |
| DEVICES\_TO\_DISABLE\_ON\_BAT               | ""    | 在更改为电池电源时禁用内置无线电设备，无论其连接状态如何。可能的值与上述相同。               |
| DEVICES\_TO\_DISABLE\_ON\_BAT\_NOT\_IN\_USE | ""    | 在更改为电池电源时禁用未连接的内置无线电设备。可能的值与上述相同。                     |

这些参数允许用户配置在系统启动、关闭或更改电源状态时如何处理内置的蓝牙、Wi-Fi 和 WWAN 设备。可通过设置禁用或启用这些设备，以及在何种条件下执行这些操作。

#### 无线配置向导（自动化配置）

| 参数名称                                      | 参考参数值       | 描述                                          |
| ----------------------------------------- | ----------- | ------------------------------------------- |
| DEVICES\_TO\_DISABLE\_ON\_LAN\_CONNECT    | "wifi wwan" | 当建立 LAN 连接时，禁用蓝牙、Wi-Fi 和 WWAN 设备。多个设备用空白分隔。 |
| DEVICES\_TO\_DISABLE\_ON\_WIFI\_CONNECT   | "wwan"      | 当建立 Wi-Fi 连接时，禁用 WWAN 设备。                   |
| DEVICES\_TO\_DISABLE\_ON\_WWAN\_CONNECT   | "wifi"      | 当建立 WWAN 连接时，禁用 Wi-Fi 设备。                   |
| DEVICES\_TO\_ENABLE\_ON\_LAN\_DISCONNECT  | "wifi wwan" | 当断开 LAN 连接时，启用蓝牙、Wi-Fi 和 WWAN 设备。多个设备用空白分隔。 |
| DEVICES\_TO\_ENABLE\_ON\_WIFI\_DISCONNECT | ""          | 当断开 Wi-Fi 连接时，启用所有设备。                       |
| DEVICES\_TO\_ENABLE\_ON\_WWAN\_DISCONNECT | ""          | 当断开 WWAN 连接时，启用所有设备。                        |
| DEVICES\_TO\_ENABLE\_ON\_DOCK             | ""          | 在对接后，启用所有设备。                                |
| DEVICES\_TO\_DISABLE\_ON\_DOCK            | ""          | 在对接后，禁用所有设备。                                |
| DEVICES\_TO\_ENABLE\_ON\_UNDOCK           | "wifi"      | 在取消对接后，启用 Wi-Fi 设备。                         |
| DEVICES\_TO\_DISABLE\_ON\_UNDOCK          | ""          | 在取消对接后，禁用所有设备。                              |

这些参数允许用户配置在特定事件触发时如何处理内置的蓝牙、Wi-Fi 和 WWAN 设备。用户可以根据 LAN、Wi-Fi 或 WWAN 的连接状态、对接或取消对接等事件来启用或禁用这些设备。

#### PCIE电源配置

| 参数名称                          | 默认参数值                    | 描述                                                                                                                      |
| ----------------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| RUNTIME\_PM\_ON\_AC           | on                       | 控制 PCIe 设备的运行时电源管理。可能的值：auto（启用）或 on（禁用）。未配置时的默认值：on（AC）。                                                               |
| RUNTIME\_PM\_ON\_BAT          | auto                     | 控制 PCIe 设备的运行时电源管理。可能的值：auto（启用）或 on（禁用）。未配置时的默认值： auto（BAT）。                                                           |
| RUNTIME\_PM\_DENYLIST         | ""                       | 从运行时电源管理中排除列出的 PCIe 设备地址。使用 lspci 查找地址。                                                                                 |
| RUNTIME\_PM\_DRIVER\_DENYLIST | "mei\_me nouveau radeon" | 从运行时电源管理中排除分配给所列驱动程序的 PCIe 设备。使用 tlp-stat -e 查找驱动程序。                                                                    |
| RUNTIME\_PM\_ENABLE           | ""                       | 为列表中的 PCI（e） 设备地址永久启用（自动）运行时 PM。这优先于所有先前的运行时 PM 设置。使用 lspci 获取地址。                                                       |
| RUNTIME\_PM\_DISABLE          | ""                       | 为列表中的 PCI（e） 设备地址永久禁用（on）运行时 PM。与 RUNTIME\_PM\_ENABLE 类似，不过是禁用。使用 lspci 获取地址。                                           |
| PCIE\_ASPM\_ON\_AC            | default                  | 设置 PCIe ASPM 省电模式。可能的值：default（推荐）、performance（性能）、powersave（省电）和 powersupersave（PowerSuperSave，超级省电）。未配置时的默认值：default。 |
| PCIE\_ASPM\_ON\_BAT           | default                  | 设置 PCIe ASPM 省电模式。可能的值：default（推荐）、performance（性能）、powersave（省电）和 powersupersave（PowerSuperSave，超级省电）。未配置时的默认值：default。 |

这些参数允许用户配置与 PCIe 设备相关的运行时电源管理和 ASPM 等功能。用户可以根据电源来源、设备地址、驱动程序等来调整这些设置，以实现更好的功耗管理。（建议不要对nvidia驱动进行调整，可能会引发意外）

#### USB

| 参数名称                                    | 默认参数值 | 描述                                                                    |
| --------------------------------------- | ----- | --------------------------------------------------------------------- |
| USB\_AUTOSUSPEND                        | 1     | 在启动时和插入时为 USB 设备设置自动挂起模式。可能的值：1（启用）或 0（禁用）。未配置时的默认值：1。                |
| USB\_DENYLIST                           | ""    | 从自动挂起模式中排除 USB 设备 ID。使用 tlp-stat -u 查找 ID。多个 ID 用空格分隔。                |
| USB\_EXCLUDE\_AUDIO                     | 1     | 从自动挂起模式中排除音频设备：1（排除）或 0（不排除）。未配置时的默认值：1。                              |
| USB\_EXCLUDE\_BTUSB                     | 0     | 从自动挂起模式中排除蓝牙设备：1（排除）或 0（不排除）。未配置时的默认值：0。                              |
| USB\_EXCLUDE\_PHONE                     | 0     | 将智能手机从自动挂起模式中排除以启用充电：1（排除）或 0（不排除）。未配置时的默认值：0。                        |
| USB\_EXCLUDE\_PRINTER                   | 1     | 从自动挂起模式中排除打印机：1（排除）或 0（不排除）。未配置时的默认值：1。                               |
| USB\_EXCLUDE\_WWAN                      | 0     | 从自动挂起模式中排除内置 WWAN 设备：1（排除）或 0（不排除）。未配置时的默认值：0。                        |
| USB\_ALLOWLIST                          | ""    | 为已被上述任何设置排除的 USB 设备 ID 重新启用自动挂起模式。使用 `tlp-stat -u` 查找 ID。多个 ID 用空格分隔。 |
| USB\_AUTOSUSPEND\_DISABLE\_ON\_SHUTDOWN | 0     | 在系统关闭时禁用 USB 自动挂起模式：1（启用）或 0（禁用）。未配置时的默认值：0。                          |

#### Trace Mode

`TLP_DEBUG="arg bat disk lock nm path pm ps rf run sysfs udev usb"`

## 结语

我们对于系统的优化不仅于此，现阶段tlp的配置策略仅对于部分有能力的用户公开，后续经过充分的测试和调优之后，会提供几份默认的配置给普通用户使用。并将来将这些配置文件GUI化，集成于深度定制项目中，为用户提供更为方便直观的操作体验。

从这一阶段对于电源优化的探索可以看出，deepin系统的电源管理方案优化不仅是为了解决用户反馈的问题，更是一种对用户需求的回应和尊重。在未来，deepin系统将继续秉持用户至上的原则，不断提升系统的性能和用户体验，为广大用户提供更加优秀的操作系统产品。
