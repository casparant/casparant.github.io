---
layout: post
title: "Use Non-export Symbol"
date: 2015-05-21 18:38:25 +0800
comments: true
categories:
- Kernel
---

最近在折腾 [LIO](http://linux-iscsi.org)，里面的模块好多，有核心模块(`target_core_mod`)、前端驱动模块(`vhost-scsi`, `iscsi`, etc)、后端驱动模块(`target_core_iblock`, `target_core_file`, etc)。我想要实现某种程度的模块“热升级”，具体来说，比如`target_core_mod`这个核心模块升级了之后，想要和旧模块共存，新的前端和后端驱动在接入的时候直接使用新模块。

于是想当然地认为只要把编译出来的模块改个名字就好了。然而事情并没那么简单，中间碰到两个问题，首先就是本文要讲到的，导出符号(EXPORT\_SYMBOL)重复的问题。其实问题很好理解，我改了个名字之后的模块(比如：`target_core_mod_new.ko`，在代码中还是会导出相同的符号，所以解决方法就是在新的模块中不导出这些符号，删掉所有的`EXPORT_SYMBOL()`宏。

可是问题又来了，我的后端驱动模块(比如：`target_core_iblock.ko`)需要依赖新的模块的函数，在无法导出符号的情况下，我没有办法使用后端驱动模块。稍微搜了一下，找到`kallsyms_lookup_name()`函数([定义](http://lxr.free-electrons.com/source/kernel/kallsyms.c?v=2.6.33#L170))，这个函数传入一个类似 ___module:symbol___ 形式的参数，用于读取`/proc/kallsyms`中的符号表，找到对应的符号，然后返回符号的地址(即函数的入口)。

于是我拿了这个函数开开心心地去改代码了。以`target_complete_cmd()`这个函数为例，我在后端驱动模块的代码需要调用这个函数的地方：
<!--more-->

``` C
    target_complete_cmd(cmd, status);
    kfree(ibr);
```

把它改为这样：

``` C
    /* 参考 target_core_mod 里的函数原型，定义函数指针变量 */
    void (*target_complete_cmd_ptr)(struct se_cmd *, u8);
    /* 得到的是 target_complete_cmd 函数入口地址 */
    target_complete_cmd_ptr = (void *)kallsyms_lookup_name(
        "target_core_mod_new:target_complete_cmd");
    (*target_complete_cmd_ptr)(cmd, status);
    kfree(ibr);
```

稍微验证了一下，发现可以用了，于是开始改其他的。

结果——WTF! 一共有40+个被引用的导出符号！一个一个改的话，这后端驱动的代码还能看么？

思来想去，觉得，该改的还是得改。决定写个宏包装一下，尽量提高一下可读性。

以下省略我反复碰壁的过程，直接上成品(some ideas was inspired by [this post](http://constellation.github.io/blog/2014/12/07/importing-hidden-kernel-symbols/), thanks!):

首先，新建一个头文件，把宏和一些函数指针定义都给丢到里面，比如叫`import_sym.h`:

``` C
#include <linux/kallsyms.h>

/* 先定义一个结构体存放所有可能用到的函数指针 */
struct symbol_set {
    <snip...>
    void (*target_complete_cmd)(struct se_cmd *, u8);
    <snip...>
};

/* 这个宏用于查找符号地址，并且做一点基本的错误检查 */
#define IMPORT_SYMBOL(name)                                             \
        sym.name = (typeof(name) *)kallsyms_lookup_name(                \
                        "target_core_mod_new:" #name);                  \
        pr_debug("symbol name: " #name ", symbol addr found: 0x%lx\n",  \
                (unsigned long)sym.name);                               \
        if (!sym.name)                                                  \
                return -EINVAL;

/*
 * 下面这两个宏会拼成一个函数，放在前后端驱动中，前面这个宏还定义了一个
 * 全局静态结构用于存放symbol的集合
 */
#define IMPORT_SYMBOL_START                                             \
static struct symbol_set sym;                                           \
static int __init_syms_lookup(void)                                     \
{

#define IMPORT_SYMBOL_END                                               \
        return 0;                                                       \
}

/* 下面这个宏用来替换前后端驱动中每个要引用原导出符号的语句 */
#define IMPORTED(name) (*sym.name)
```

接下来改前后端驱动的代码，还是以`target_core_iblock`为例：

``` C
#include "import_sym.h"

IMPORT_SYMBOL_START
IMPORT_SYMBOL(target_complete_cmd);
/* 这里还可以继续添加IMPORT_SYMBOL宏以查找更多需要用的符号 */
IMPORT_SYMBOL_END

<snip...>
/* 把 target_complete_cmd(cmd, status); 换掉 */
IMPORTED(target_complete_cmd)(cmd, status);

<snip...>
static int __init iblock_module_init(void)
{
    /* 在模块初始化的时候就找齐所有需要的符号地址 */
    int ret = __init_syms_lookup();
    if (ret)
        return ret;
    return IMPORTED(transport_subsystem_register)(&iblock_template);
}
```

这样就可以了。

由于不再使用`EXPORT_SYMBOL()`，做出来的前后端驱动模块会不再显式依赖`target_core_mod`，必须得确保 core 模块先加载，才能让前后端驱动模块加载之后找到符号地址。卸载的时候 core 模块要最后卸载，否则会碰到空指针引发 Kernel Panic。这算是这个方案的一个不完美之处吧。
