---
title: 第一届小浣熊杯修 bug 大赛
date: 2023-03-09 16:29:07
tags: ['小浣熊杯']
authors: ["BLumia"]
---

deepin v23 beta 的发布在即，为了能够使相关的 bug 能够得以更快解决，并促进研发团队的协作变得更高效，我们（开源社区中心）决定在 deepin 员工内部举办小规模的 bug hunting 性质的比赛，并命名该比赛为“小浣熊杯修 bug 大赛”。而首届“小浣熊杯”也于昨天顺利落幕，那么就让我们一起了解一下这个比赛吧！

<!--more-->

## 比赛介绍

如上所述，“小浣熊杯“的比赛大致内容即为在比赛时间内对已有缺陷进行修复。我们将所有参赛的员工划分为多个组，每个组除研发外也配备一个测试人员。在比赛时间周期内，研发从指定的缺陷看板中挑选自己”中意“的 BUG 进行修复，并在修复后将修复公布在相关群内，由 **其他组** 的测试人员进行测试。当修复被测试人员验证没有问题后，即可进行计分。最终，会以本组研发人员所修复的数量与测试人员所完成的测试数量相组合，并计算小组人均得分，最后以小组人均得分的高低决定最终排名。

小组得分的计算公示为：

```plain
小组得分 = (本组研发修复的缺陷数量 + 本组测试所验证的缺陷数量 / 3) / 小组总人数
```

比赛会在 https://github.com/linuxdeepin/.bug-game/ 中进行，每次比赛会创建一个看板来跟进整个比赛的实时情况，并创建一个对应的 issue 记录相关进展与结果。

另外，考虑到比赛过程中对缺陷的修复不需要经过其他研发人员的 code review ，因而可能存在实际的代码质量问题，故相关的 PR 均不要求在比赛结束前合入，相关提交仍需按照正常流程，经过有效 review 获得 approval 后合入。

## 首届状况介绍

![比赛过程看板截图](https://user-images.githubusercontent.com/13449038/223312494-6039b4a2-309a-44be-a8ac-61edffa9f963.png)

首届比赛共划分了四个小队参赛，比赛时间从 3 月 7 日开始，为期两天。比赛过程与结果在 [这个 Issue](https://github.com/linuxdeepin/.bug-game/issues/1) 中汇总，最终的缺陷修复情况也可以参见 [这个看板](https://github.com/orgs/linuxdeepin/projects/26)。比赛过程中“修 bug”队一度领先，随后被”进击的小浣熊“队反超，最终经过了两天的”激烈比拼“后，本次比赛总计处理了 42 个 Issue，由”进击的小浣熊“队以 18.333 分的总积分获得人均积分第一夺冠。“修 bug”队紧随其后，“呆呆鹅” 与 “bug 收割小分队” 获得随后的名次。

比赛过程中，各个小组对已有 bug 的挑选与“占坑”以及测试人员对新提交修复的“抢单”是过程中最有趣的事情之一。快速挑选便于修复的 BUG 并进行有效的修复成为了获胜的关键之一，根据表象快速分析推测问题的能力，以及在陌生项目[^1]中快速尝试定位和修复问题的能力也变得至关重要。作为花絮，有的小组也在选择 BUG 的过程中连续发现自己所选择的缺陷实际早已在版本迭代中被修复，耗费了较多时间而造成了相对的失利，但这个过程也对现存 BUG 的有效性验证有很大的帮助[^2]。

赛后，我们进行了比赛的颁奖，获胜队伍获得了比赛限定奖杯与荣誉证书，以及一个 deepin 主题背包。获奖队伍也进行了合影：

![荣誉证书](https://user-images.githubusercontent.com/10095765/224207416-e1fa516d-8993-4d1c-8d1f-81114c0766ce.png)

![颁奖截图](https://user-images.githubusercontent.com/13449038/223911631-376c0d58-17a5-4401-843d-414dc287b7be.png)

无论是否获奖，我们都感谢各个参赛队伍的积极参与，也希望各位能在后续的比赛中能够获得优异的成绩。

[^1]: 注：缺陷的修复不限于自己所维护的项目，研发人员也可以尝试修复由其他项目组所维护的缺陷。
[^2]: 或许在后续的比赛中应当为此类也算作计分项。