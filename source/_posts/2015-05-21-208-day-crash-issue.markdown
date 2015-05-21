---
layout: post
title: "Linux Kernel 208.5d Panic Issue"
date: 2015-05-21 15:39:33 +0800
comments: true
categories:
- Kernel
---

最近线上碰到的问题，虽说早就有解决方案了，但是陆陆续续还是碰到很多没有升级内核的机器挂掉。记录一下以下内容，仅供参考。

现象是机器运行到一定天数(刚开始反馈集中在 208.5 天左右)就会主动挂掉，报告文章末尾的 CallTrace，所幸社区几年前就有了解决方案：

Upstream Patch
--------------

```
commit 4cecf6d401a01d054afc1e5f605bcbfe553cb9b9
Author: Salman Qazi <sqazi@google.com>
Date:   Tue Nov 15 14:12:06 2011 -0800

    sched, x86: Avoid unnecessary overflow in sched_clock

    (Added the missing signed-off-by line)

    In hundreds of days, the __cycles_2_ns calculation in sched_clock
    has an overflow.  cyc * per_cpu(cyc2ns, cpu) exceeds 64 bits, causing
    the final value to become zero.  We can solve this without losing
    any precision.

    We can decompose TSC into quotient and remainder of division by the
    scale factor, and then use this to convert TSC into nanoseconds.

    Signed-off-by: Salman Qazi <sqazi@google.com>
    Acked-by: John Stultz <johnstul@us.ibm.com>
    Reviewed-by: Paul Turner <pjt@google.com>
    Cc: stable@kernel.org
    Signed-off-by: Peter Zijlstra <a.p.zijlstra@chello.nl>
    Link: http://lkml.kernel.org/r/20111115221121.7262.88871.stgit@dungbeetle.mtv.corp.google.com
    Signed-off-by: Ingo Molnar <mingo@elte.hu>
```
<!--more-->

Red Hat Fix
-----------

+ Z-stream since 2.6.32-220.5.1.el6
+ Y-stream Since 2.6.32-226.el6

Ref Links
---------

* [http://kenichiokuyama.blogspot.hk/2011/12/schedclock-overflow-after-2085-days-in.html](http://kenichiokuyama.blogspot.hk/2011/12/schedclock-overflow-after-2085-days-in.html)

Call Trace
----------

```
[776183.791021] divide error: 0000 [#1] SMP
[776183.795055] last sysfs file: /sys/devices/system/cpu/cpu15/cache/index2/shared_cpu_map
[776183.803045] CPU 9
[776183.804967] Modules linked in: <snip>
[776183.844841]
[776183.846415] Modules linked in: <snip>
[776183.886217] Pid: 0, comm: swapper Not tainted <kernel>
[776183.895358] RIP: 0010:[<ffffffff81059e15>] [<ffffffff81059e15>] find_busiest_group+0x485/0xb50
[776183.904258] RSP: 0018:ffff880028283c80 EFLAGS: 00010246
[776183.909703] RAX: 0000000000000000 RBX: 0000000000000000 RCX: ffff88002828f900
[776183.917021] RDX: 0000000000000000 RSI: ffc2c20d09ff2179 RDI: 000000001dcd6500
[776183.924341] RBP: ffff880028283de0 R08: 0002c1ef5f24b5a6 R09: ffff88002828f910
[776183.931658] R10: 0000000000000000 R11: 0000000000000000 R12: ffff880028295f01
[776183.938977] R13: ffff88002828f900 R14: 0000000000015f40 R15: ffffffffffffffff
[776183.946296] FS: 0000000000000000(0000) GS:ffff880028280000(0000) knlGS:0000000000000000
[776183.954568] CS: 0010 DS: 0018 ES: 0018 CR0: 000000008005003b
[776183.960447] CR2: 00000000008cb028 CR3: 0000000c61a84000 CR4: 00000000000006e0
[776183.967776] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[776183.975095] DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
[776183.982418] Process swapper (pid: 0, threadinfo ffff880664a52000, task ffff880c6412f4c0)
[776183.990691] Stack:
[776183.992839] ffff880028283e40 ffff880028283d80 ffff880c7fc29600 ffff880028283e64
[776184.000247] <0> 0000000000000040 ffff880028283e58 0000000960cf4700 ffff88002828f5e0
[776184.008157] <0> 0000000028283ce0 ffff88002828f910 00000001ffffffff 0000000000000009
[776184.016303] Call Trace:
[776184.018884] <IRQ>
[776184.021138] [<ffffffff8105af96>] rebalance_domains+0x146/0x520
[776184.027193] [<ffffffff8105b3bc>] run_rebalance_domains+0x4c/0x150
[776184.033506] [<ffffffff8106b1e0>] __do_softirq+0xc0/0x1e0
[776184.039038] [<ffffffff8100c18c>] call_softirq+0x1c/0x30
[776184.044483] [<ffffffff8100dd55>] do_softirq+0x65/0xa0
[776184.049755] [<ffffffff8106af2c>] irq_exit+0x7c/0x90
[776184.054856] [<ffffffff814b0710>] smp_apic_timer_interrupt+0x70/0x9d
[776184.061343] [<ffffffff8100bb53>] apic_timer_interrupt+0x13/0x20
[776184.067485] <EOI>
[776184.069738] [<ffffffff813bdc8a>] ? poll_idle+0x2a/0x70
[776184.075097] [<ffffffff813bdc73>] ? poll_idle+0x13/0x70
[776184.080454] [<ffffffff813bec86>] ? menu_select+0xa6/0x330
[776184.086072] [<ffffffff813bdbc9>] cpuidle_idle_call+0x99/0x130
[776184.092042] [<ffffffff81009888>] cpu_idle+0xa8/0xe0
[776184.097143] [<ffffffff814a0d64>] ? start_secondary+0x1b4/0x2a0
[776184.103194] [<ffffffff814a0d72>] start_secondary+0x1c2/0x2a0
[776184.109072] Code: d8 b8 01 00 00 00 48 c1 eb 0a 48 85 db 0f 45 c3 41 89 45 08 48 8b 8d 00 ff ff ff 48 8b 45 a8 8b 51 08 48 c1 e0 0a 48 89 d3 31 d2 <48> f7 f3 48 8b 55 b0 48 89 45 a0 31 c0 48 85 d2 74 0f 48 8b 45
[776184.129029] RIP [<ffffffff81059e15>] find_busiest_group+0x485/0xb50
[776184.135529] RSP <ffff880028283c80>
[776184.139477] BUG: scheduling while atomic: swapper/0/0x10000100
[776184.139485] ---[ end trace 32313550323d1731 ]---
[776184.139586] Kernel panic - not syncing: Fatal exception in interrupt
[776184.139589] Pid: 0, comm: swapper Tainted: G D ---------------- <kernel>
[776184.139591] Call Trace:
[776184.139592] <IRQ> [<ffffffff81062e85>] ? panic+0xa5/0x1a0
[776184.139598] [<ffffffff81352006>] ? netoops+0x1c6/0x2a0
[776184.139601] [<ffffffff81064ac5>] ? kmsg_dump+0x135/0x180
[776184.139604] [<ffffffff81062ae5>] ? oops_exit+0x25/0x30
[776184.139606] [<ffffffff814abafb>] ? oops_end+0xeb/0x100
[776184.139609] [<ffffffff8100f07b>] ? die+0x5b/0x90
[776184.139612] [<ffffffff814ab226>] ? do_trap+0x136/0x150
[776184.139616] [<ffffffff8100c80f>] ? do_divide_error+0x8f/0xb0
[776184.139618] [<ffffffff81059e15>] ? find_busiest_group+0x485/0xb50
[776184.139623] [<ffffffff8142124f>] ? ip_rcv+0x26f/0x380
[776184.139626] [<ffffffff8100bdbb>] ? divide_error+0x1b/0x20
[776184.139629] [<ffffffff81059e15>] ? find_busiest_group+0x485/0xb50
[776184.139633] [<ffffffff8105af96>] ? rebalance_domains+0x146/0x520
[776184.139636] [<ffffffff8105b3bc>] ? run_rebalance_domains+0x4c/0x150
[776184.139639] [<ffffffff8106b1e0>] ? __do_softirq+0xc0/0x1e0
[776184.139642] [<ffffffff8100c18c>] ? call_softirq+0x1c/0x30
[776184.139644] [<ffffffff8100dd55>] ? do_softirq+0x65/0xa0
[776184.139647] [<ffffffff8106af2c>] ? irq_exit+0x7c/0x90
[776184.139650] [<ffffffff814b0710>] ? smp_apic_timer_interrupt+0x70/0x9d
[776184.139653] [<ffffffff8100bb53>] ? apic_timer_interrupt+0x13/0x20
[776184.139654] <EOI> [<ffffffff813bdc8a>] ? poll_idle+0x2a/0x70
[776184.139660] [<ffffffff813bdc73>] ? poll_idle+0x13/0x70
[776184.139663] [<ffffffff813bec86>] ? menu_select+0xa6/0x330
[776184.139666] [<ffffffff813bdbc9>] ? cpuidle_idle_call+0x99/0x130
[776184.139669] [<ffffffff81009888>] ? cpu_idle+0xa8/0xe0
[776184.139672] [<ffffffff814a0d64>] ? start_secondary+0x1b4/0x2a0
[776184.139676] [<ffffffff814a0d72>] ? start_secondary+0x1c2/0x2a0
[776184.325813] Modules linked in: <snip>
[776184.367794] CPU 1:
[776184.370036] Modules linked in: <snip>
[776184.412014] Pid: 0, comm: swapper Tainted: G D ---------------- <kernel>
[776184.423621] RIP: 0010:[<ffffffff813bdc8a>] [<ffffffff813bdc8a>] poll_idle+0x2a/0x70
[776184.431644] RSP: 0018:ffff88066495bea8 EFLAGS: 00000246
[776184.437128] RAX: ffff88066495bfd8 RBX: ffff88066495bed8 RCX: 0042e00b649a5000
[776184.444484] RDX: 000000000001f6a9 RSI: ffff88002821d610 RDI: ffffffff81a2fbc0
[776184.451844] RBP: ffffffff8100bb4e R08: 0000000000000000 R09: 0000000000000000
[776184.459201] R10: 0000000000000000 R11: 0000000000000000 R12: 0042dfee0e6ae400
[776184.466567] R13: 0000000000000000 R14: 0000000000000000 R15: 0000000561f5fe16
[776184.473927] FS: 0000000000000000(0000) GS:ffff880028200000(0000) knlGS:0000000000000000
[776184.482241] CS: 0010 DS: 0018 ES: 0018 CR0: 000000008005003b
[776184.488159] CR2: 00000000427b7078 CR3: 0000000651611000 CR4: 00000000000006e0
[776184.495520] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[776184.502878] DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
[776184.510236] Call Trace:
[776184.512861] [<ffffffff813bdc73>] ? poll_idle+0x13/0x70
[776184.518261] [<ffffffff813bec86>] ? menu_select+0xa6/0x330
[776184.523918] [<ffffffff813bdbc9>] ? cpuidle_idle_call+0x99/0x130
[776184.530096] [<ffffffff81009888>] ? cpu_idle+0xa8/0xe0
[776184.535408] [<ffffffff814a0d64>] ? start_secondary+0x1b4/0x2a0
[776184.541499] [<ffffffff814a0d72>] ? start_secondary+0x1c2/0x2a0
```
