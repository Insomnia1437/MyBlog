---
title: Real time performance
categories:
  - MEMO
  - OS
tags:
  - Linux
  - EPICS
  - Real-time
  - Chinese
abbrlink: '9135e595'
date: 2020-06-02 20:32:51
---

```python
# @Time    : 20200602
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

# 实时操作系统性能分析

因为最近需要测试下Raspberry Pi 4B作为EPICS IOC的实时性能，所以复习了下操作系统实时性相关的知识（用了本科的教材《操作系统概念（第七版）》郑扣根译）。本文不会再细讲CPU调度的内容，只会着重分析实时性相关内容。

<!-- more -->


### 中断处理

- Top half（或者：interrupt handler）： 对应于那些有实时性要求的中断，执行过程中关闭中断，比如接收网卡的数据，放入buffer，然后schedule它的bottom half，退出。
- bottom half：相对而言实时性要求没那么高，执行过程中可被top half中断，有多种实现的机制，比如Linux kernel 2.4之前使用的 `bottom-half` (`BH`，要注意区分，一种是理论，使用缩写的话代表是一种机制的实现方式)，`tasklet`，`workqueue`。一些代码的细节参考这篇文章[http://www.wowotech.net/irq_subsystem/soft-irq.html](http://www.wowotech.net/irq_subsystem/soft-irq.html).

> 二者处理过程中都不可sleep，不可访问user space，不可唤醒scheduler。

### 延迟测量

从中断发生到CPU开始处理都有哪些延迟，直接看图：

{% asset_img latency_measurement.jpg latency_measurement %}

Ref: [ECL2017-Slide](https://elinux.org/images/a/a9/ELC2017-_Effectively_Measure_and_Reduce_Kernel_Latencies_for_Real-time_Constraints_%281%29.pdf)

-----------------------

## 下面是一些`rt-tests`工具包里的一些程序

### pi_stress

`pi`是`priority inversion`的缩写。也就是优先级翻转。

优先级翻转大致是低优先级的任务使用了高优先级任务也同时需要的某资源，导致一个中优先级的任务抢占低优先级任务后，该资源无法被释放，导致高优先级任务反而比中优先级任务更晚执行。如果低优先级任务一直被抢占，也就会导致死锁。[wiki:Priority_inversion](https://en.wikipedia.org/wiki/Priority_inversion)

这种情况可以通过优先级继承（`priority inheritance`）避免，即当高优先级任务发现某资源被别的任务占用时，占用着资源的任务继承高优先级任务的优先级，避免被中优先级的任务抢占。

pi_stress就是利用了这个原理来stress CPU。设置多个pthread组，每个线程组高中低三个优先级的线程。看了一下源码，[pi_stress.c](https://github.com/LITMUS-RT/cyclictest/blob/master/src/pi_tests/pi_stress.c#L617-L729)，线程就只是空转CPU，用`pthread_barrier_wait`来同步执行过程。

### hackbench

hackbench也是stress CPU的工具，只是方式是通过创建进程或线程对，然后让他们通过pipe或者socket传递消息，顺便可以计算时间。看了下源码：[hackbench.c](https://github.com/LITMUS-RT/cyclictest/blob/master/src/hackbench/hackbench.c#L140)，发的就是`-`。（一个无趣的程序员）。

`man hackbench`
```
Hackbench is both a benchmark and a stress test for the Linux kernel scheduler. It's main job is to create a specified number of pairs of schedulable  entities  (either threads or traditional processes) which communicate via either sockets or pipes and time how long it takes for each pair to send data back and forth.
```

### Cyclictest

参数说明，看`cyclictest --help`

通常的命令:

`sudo cyclictest -S -m -p99 -i200 -l100000`

原理用人家原话就是：

`cyclictest measures the delta from when it's scheduled to wake up from when it actually does wake up`

其实也很简单，`-i`参数指定线程sleep time，休眠之前读一次系统时钟，唤醒之后再读一次，然后比较和休眠时间的差值，多出来的就是CPU调度的延迟了。核心逻辑代码在此：[cyclictest.c#L794-L864](https://github.com/LITMUS-RT/cyclictest/blob/master/src/cyclictest/cyclictest.c#L794-L864)

可以通过cyclictest的`-b`参数设置跟踪内核信息。

树莓派支持的内核trace选项

```shell
$ sudo cat /sys/kernel/debug/tracing/trace_options
print-parent
nosym-offset
nosym-addr
noverbose
noraw
nohex
nobin
noblock
trace_printk
annotate
nouserstacktrace
nosym-userobj
noprintk-msg-only
context-info
nolatency-format
record-cmd
norecord-tgid
overwrite
nodisable_on_free
irq-info
markers
noevent-fork
function-trace
nofunction-fork
nodisplay-graph
nostacktrace
notest_nop_accept
notest_nop_refuse
```

### CPU freq

测试中发现使用`chrt`命令调整程序优先级之后，latency依然不稳定，经过测试发现，是由于CPU没有工作在最高频率。

安装并测试`cpufrequtils`:

`apt install cpufrequtils`

内部包含了三个工具：

- /usr/bin/cpufreq-info
- /usr/bin/cpufreq-set
- /usr/bin/cpufreq-aperf

具体使用可以参考手册。

