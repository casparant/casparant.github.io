---
layout: post
title: MCE 的一些零散记录
categories:
- Kernel
tags: []
published: true
comments: true
---
以前也测过 MCE 相关的东西，但是惭愧，一直不清不楚。这两天稍微整理了一下相关知识，感觉见识还是很浅，记些零碎的东西在这，权当博客复活第一篇。如有理解错误，欢迎指出。

CPU 检测到硬件错误时，内核会报 Machine-check，根据硬件错误是否可以修正(CE, Correctable Error; UCE/UE, Un-correctable Error)，内核做出不同反应。CE 的话内核只把相关信息写到一个字符设备 `/dev/mcelog` 中，UE 的话是会在记录相关信息之余，还会做出不同处理，比如中断当前遇错的应用程序，或者 Panic. `[TODO: 这块相关的内核代码还得再看几遍]`

`/dev/mcelog` 中的信息可以用用户空间工具 __[mcelog](https://github.com/andikleen/mcelog)__ 来读取。__mcelog__ 比较有意思的一个参数是 `--dmi`，可以尝试解析 ADDR 字段以获取诸如内存出厂信息，DIMM 位置等有用的信息，可是很遗憾在实际环境中这些 DIMM 信息基本上都是错的。 (mcelog 的 man page说了，不要怪 Linux，得怪那稀奇古怪的主板厂商……)

man page 还有一个地方提到：

> The kernel prefers old messages over new. If the log buffer overflows only old ones will be kept.

<!--more-->查看内核代码中 `arch/x86/include/asm/mce.h`，有：

``` C
#define MCE_LOG_LEN 32

struct mce_log {
        char signature[12]; /* "MACHINECHECK" */
        unsigned len;       /* = MCE_LOG_LEN */
        unsigned next;
        unsigned flags;
        unsigned recordlen;     /* length of struct mce */
        struct mce entry[MCE_LOG_LEN];
};
```

这里的 `MCE_LOG_LEN` 就是 man page 中提到的 log buffer 长度。

内核中还有个 `mce_inject` 模块（`CONFIG_X86_MCE_INJECT=m/y`）,可以用于产生一些假的 MCE 以测试相关功能。需要配合用户空间工具 __[mce-inject](https://github.com/andikleen/mce-inject/)__ 来使用。

如果要做一个比较完整的测试，__[mce-test](https://github.com/andikleen/mce-test)__ 工具可以帮忙，其中会涉及一些 RAS 特性，比如 ___hwpoison___(`Documentation/vm/hwpoison.txt`)，不过我现在的活儿不怎么管测试，暂时就没去回顾这个测试工具了。

先记这些，回头把细节都搞懂了再补充……
