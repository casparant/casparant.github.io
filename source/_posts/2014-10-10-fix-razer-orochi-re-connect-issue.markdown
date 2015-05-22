---
layout: post
title: Fix Razer Orochi re-connect issue
categories:
- Z-Turn
tags: []
published: true
comments: true
---

Issue
-----

雷蛇八歧大蛇 2013 版鼠标，短时间内不用自动休眠，系统中蓝牙连接断(预期行为)，鼠标从休眠恢复后系统中的蓝牙连接却无法自动重连，只能将鼠标置为配对模式，通过系统中的蓝牙工具手工连接。

Environment
-----------

```
Razer Orochi 蓝牙鼠标
Gentoo + KDE
kernel 3.10 ~ 3.16
bluez 4.101 ~ 5.23
bluedevil 1.3.2 ~ 2.0_rc1
```

Solution
--------

<!--more-->

通过 `hcidump` 工具发现：

```
> HCI Event: Connect Request (0x04) plen 10
    bdaddr F0:65:DD:94:9F:BB class 0x002580 type ACL
> HCI Event: Command Status (0x0f) plen 4
    Accept Connection Request (0x01|0x0009) status 0x00 ncmd 1
> HCI Event: Connect Complete (0x03) plen 11
    status 0x00 handle 13 bdaddr F0:65:DD:94:9F:BB type ACL encrypt 0x00
> HCI Event: Command Status (0x0f) plen 4
    Read Remote Supported Features (0x01|0x001b) status 0x00 ncmd 1
> HCI Event: Read Remote Supported Features (0x0b) plen 11
    status 0x00 handle 13
    Features: 0xbf 0x00 0xa0 0x78 0x18 0x1e 0x59 0x83
> HCI Event: Command Status (0x0f) plen 4
    Read Remote Extended Features (0x01|0x001c) status 0x00 ncmd 1
> HCI Event: Read Remote Extended Features (0x23) plen 13
    status 0x00 handle 13 page 1 max 1
    Features: 0x01 0x00 0x00 0x00 0x00 0x00 0x00 0x00
> HCI Event: Command Status (0x0f) plen 4
    Remote Name Request (0x01|0x0019) status 0x00 ncmd 1
> HCI Event: Remote Name Req Complete (0x07) plen 255
    status 0x00 bdaddr F0:65:DD:94:9F:BB name 'Razer Orochi 2013'
> HCI Event: Command Complete (0x0e) plen 10
    Link Key Request Negative Reply (0x01|0x000c) ncmd 1
    status 0x00 bdaddr F0:65:DD:94:9F:BB
> HCI Event: Disconn Complete (0x05) plen 4
    status 0x00 handle 13 reason 0x13
    Reason: Remote User Terminated Connection
```

出现 ___Link Key Request Negative Reply___ 的字样，怀疑是配对的时候使用自动 key，而蓝牙工具没有保存这个自动 key，导致鼠标从休眠恢复时试图以空 key 连接，从而失败。

知道了问题，解决方法就简单了，配对的时候手工输入 key/PIN 为 ___1234___，连接即可。

就这么一个破问题，折腾了我半天时间，内核、bluez、bluedevil 都尝试了一遍，还在人家的 Macbook 上测试了一番。差点就冲动下单换 Mac 了。 `#我不是土豪#`
