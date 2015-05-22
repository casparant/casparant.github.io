---
layout: post
title: 'WizNote: 终于找到比较干净的 Qt 的笔记软件'
categories:
- Z-Turn
tags: []
published: true
comments: true
---
Linux 上记笔记，向来很纠结。以前用过一段时间 [Zim](http://zim-wiki.org/index.html)，后来变成了 KDE 党，只好果断抛弃了。找到 [BasKet](http://basket.kde.org/)，功能还是很强大的，可是格式一团糟。无法以 Plain Text 存储文字，拷贝来拷贝去的时候经常格式混乱。也尝试在 Web 端存东西，结果还是发现不习惯，而且离线无法访问。

然后今天工作的时候有同事提到客户报告鄙厂[产品](http://www.aliyun.com/)的问题，碰到了故障（当然在内核组大牛们的努力下该问题已经解决了），我就好奇去看了一下报告者信息，然后就发现了这个笔记软件：[Wiz](http://wiz.cn/index.html). 看起来还挺 Geek 的，有手机端有 Linux 端有 Mac 端。于是去 gentoo-zh 的 Overlay 找了一圈（是的，5年过去了我还在 Gentoo 的不归路上慢慢走着），[找到了](https://github.com/microcai/gentoo-zh/blob/master/app-text/wiznote/wiznote-2.0.64.ebuild)。

看起来样子还不错，还比较清爽，也可以格式化为 Plain Text。功能上比 BasKet 少一些，但是用着还是挺舒服的。

最不能忍受的问题是没有任务栏图标，于是接着搜。

找到了一年前某人的一个没有 Merge 的 [Pull Request](https://github.com/WizTeam/WizQTClient/pull/80)，然后稍微修改了一下补丁，打到源代码里，发现可以用了。

修改后的 ebuild 请[猛戳](https://github.com/casparant/caspar-gentoo/blob/master/app-text/wiznote/wiznote-2.1.0_beta.ebuild)。

最后，有更靠谱的笔记软件请推荐。
