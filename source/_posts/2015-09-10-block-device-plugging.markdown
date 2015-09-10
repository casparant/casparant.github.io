---
layout: post
title: "Block Device Plugging"
date: 2015-09-10 18:31:25 +0800
comments: true
categories:
- Kernel
---

关于 block 设备的 plug 和 unplug 机制，可以参考[这篇](http://lxr.free-electrons.com/source/Documentation/block/biodoc.txt#L1011)内核文档。简单来说就是内核提供了一个 plug 机制，请求被放到一个空队列里之后，可以将队列处于 "plugged" 状态，此时队列并不真正向下层设备发射(issue)请求，而是攒够足够多的请求之后，将队列 "unplug" 之后才发射，在这期间，I/O 的调度算法将有机会对请求进行合并和排序。unplug 的时机既可以是一个手工的 `blk_unplug()` 操作，也可以由一个 `unplug_timer` 定时器超时触发。这整个过程听起来有点像在等机场大巴，到站停靠(plug)，装载乘客(攒request)，到点发车(unplug\_timeout)或者人满发车(blk\_unplug)。

很容易想到，unplug\_timer 这个定时器的启动会发生在 plug 阶段(见 block/blk-core.c 中的实现):

```
void blk_plug_device(struct request_queue *q)
{
        WARN_ON(!irqs_disabled());
        /*
         * don't plug a stopped queue, it must be paired with blk_start_queue()
         * which will restart the queueing
         */
        if (blk_queue_stopped(q))
                return;
        if (!queue_flag_test_and_set(QUEUE_FLAG_PLUGGED, q)) {
                mod_timer(&q->unplug_timer, jiffies + q->unplug_delay);
                trace_block_plug(q);
        }
}
EXPORT_SYMBOL(blk_plug_device);
```

而定时器超时函数见同一个文件下的实现：

```
void blk_unplug_timeout(unsigned long data)
{
        struct request_queue *q = (struct request_queue *)data;
        trace_block_unplug_timer(q);
        kblockd_schedule_work(q, &q->unplug_work);
}
```

这个函数是在 `blk_queue_make_request` 定义的时候设置的(见 block/blk-settings.c)：

```
void blk_queue_make_request(struct request_queue *q, make_request_fn *mfn)
{
        <snip>
        q->unplug_thresh = 4;           /* hmm */
        q->unplug_delay = msecs_to_jiffies(3);  /* 3 milliseconds */
        if (q->unplug_delay == 0)
                q->unplug_delay = 1;
        q->unplug_timer.function = blk_unplug_timeout;
        q->unplug_timer.data = (unsigned long)q;
        <snip>
}
```    

可以看到除了定义了超时函数，还定义了超时的时间，是 3ms. 这个 3ms 可以用来排查一系列奇怪的 I/O 性能问题。比如说在 iodepth 为1的情况下，发现 IOPS 是很固定的 333，那么多半就是因为请求队列没有及时 unplug 而导致定时器超时自动 unplug 了。通过简单计算得知，因为每个队列需要 3ms 传输，那一秒钟只能传送 333 个队列，即 IOPS == 333.

如果要主动 unplug，就需要调用 `blk_unplug()` 函数，或者 `generic_unplug_device()` 函数，从通用性来说，调用前者会比较好，看如下代码(block/blk-core.c):

```
void blk_unplug(struct request_queue *q)
{
        /*
         * devices don't necessarily have an ->unplug_fn defined
         */
        if (q->unplug_fn) {
                trace_block_unplug_io(q);
                q->unplug_fn(q);
        }
}
```

 `q->unplug_fn` 一般不需要自行设置，如果没有定义，会使用默认的函数，即 `generic_unplug_device()`。

看完了这几个函数，可以知道，plug 完之后及时 unplug 通常是避免 IO 延迟过高的良好手段。
