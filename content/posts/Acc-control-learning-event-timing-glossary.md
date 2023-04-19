---
title: 粒子加速器控制-粒子加速器定时系统常见概念
date: '2020-02-10 18:53:02'
categories:
  - Accelerator Control
tags:
  - Timing System
  - MRF
  - Chinese
  - Control
abbrlink: 4362c88b
---

```python
# @Time    : 2020-02-10
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```
# 粒子加速器定时系统

> 本系列主目录：
>  [粒子加速器控制](/posts/acc-control-learning-catalog)
------
在阅读加速器定时系统的手册或者论文时候会看到各种名词和数据，由于概念命名的不统一，会对初学者造成困扰，因此我总结一下常见的一些概念。

> 1. 如果对定时系统没有什么基础的了解，强烈推荐阅读***Timo Korhonen*** 在1999年***ICALEPCS***会议上发表的这篇文章：[REVIEW OF ACCELERATOR TIMING SYSTEMS](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.113.6272&rep=rep1&type=pdf) 里面详细介绍了定时系统的发展，目标，分类，技术以及事件定时系统的基础概念。
> 2. 常用的一些无歧义的概念此处不列出，比如EVG, EVR, Fan-out等。
> 3. 本文只列基于事件的定时系统的概念，基于时间戳同步的定时系统（比如White Rabbit 项目[White Rabbit at cern](https://white-rabbit.web.cern.ch/)）在此不予介绍。
> 4. 如果有什么错误或纰漏，欢迎发邮件斧正。

<!-- more -->

----

## 名词解释

- RF frequency 或者 Accelerating frequency，本质上就是RF的频率，环（包括**S**torage **R**ing，booster 或者 **D**amping **R**ing）高频频率通常在500 MHz左右。
- Repetition frequency 或者  Fiducial rate或injection cycle或trigger rate，指注入重复周期，通常为几Hz。例如瑞士SLS为3.125Hz，即每320ms注入一次。上海SSRF为2Hz，每500ms注入一次，而日本SuperKEKB为50Hz，每20ms注入一次。可以理解为每秒钟重复多少次注入过程（但不一定代表注入束团的数量，比如在SuperKEKB，一个pulse内可以有两个bunch）。
- Circumference，环周长，单位为米。
- orbit frequency 或者 revolution frequency，回旋频率，一秒钟粒子在环里转多少圈。由环的周长和粒子速度确定。
- orbit cycle 或者 revolution cycle，回旋周期，回旋频率的倒数，粒子在环里转一圈的时间。
- harmonic number 或者 bucket number 或者 bunch number，即谐波数，由回旋频率和RF频率共同确定，谐波数也决定了环内最多有多少个“bucket”，每个bunch都要注入进一个bucket，然后在环内升能。
- coincidence frequency 或者 common frequency 不知道怎么翻译，相干频率或者重合频率？其实就是两个环的回旋频率的最大公约数（也就是两个环回旋周期的最小公倍数），只有在这个频率下extract一个环的bunch才能恰好进入另一个环的对应bucket。
- Event rate 或者 Event clock 或 clock frequency，即Event timing system中发送事件的频率，MRF系列产品支持25MHz~125MHz。为了和环锁相，通常由环RF高频频率分频得到。
- Timing jitter或 clock precision，信号的抖动，通常要求抖动在几十皮秒(picosecond)的数量级。
- **Bucket Selection**，顾名思义就是要选择注入到环里的哪一个bucket，根据BPM测得的束团数据或运行需求选择bucket之后，计算并设置电子枪的延迟。
- Bunch spacing，连续注入的两个bunch，相隔的bucket数量。

## 以SuperKEKB为例

下面以列出SuperKEKB为例，列出其主环Main Ring 和 正电子Damping Ring的一些定时参数：

| MR 参数              | 值       | 单位   |
| -------------------- | :------- | ------ |
| Repetition frequency | 50       | 赫兹   |
| Circumference        | 3016.315 | 米     |
| RF frequency         | 508.9    | 兆赫兹 |
| Harmonic Number      | 5120     | 个     |
| Revolution cycle     | 10.06    | 毫秒   |
| Bunch spacing        | 96.3     | 纳秒   |

由表中可知，主环运行在50赫兹，共有5120个bucket，相邻两次注入时会相隔49个bucket（96.3 ns）。

| DR 参数              | 值    | 单位   |
| -------------------- | :---- | ------ |
| Repetition frequency | 50    | 赫兹   |
| Length               | 135.5 | 米     |
| RF frequency         | 508.9 | 兆赫兹 |
| Harmonic Number      | 230   | 个     |
| Revolution cycle     | 452   | 纳秒   |
| Bunch spacing        | 96.3  | 纳秒   |

## Bucket Selection

在设计加速器定时系统时，首先要考虑两个环回旋频率的同步关系，其次也要考虑linac高频与环高频之间的同步关系。除此之外也要注意与AC50电网信号的同步，此处细节较复杂，以后再介绍。

> 对于定时系统的实现细节可以参阅这几篇论文：
>
> - H. Kaji et al. “ BUCKET SELECTION SYSTEM FOR SuperKEKB”, PASJ 2015 THP100
> - H. Kaji et al. “INSTALLATION AND COMMISSIONING OF NEW EVENT TIMING SYSTEM FOR SuperKEKB”, PASJ 2015 FROL15

![SuperKEKB bucket selection](/images/skb.jpg)

SuperKEKB linac的高频频率为2856 Mhz，为了和环高频508.9 MHz同步，选择coincidence frequency为10.38 Mhz（96 ns）。关于上图的 493 μs 的计算，见下表：



| 注入机会 | Delay值 | MR bucket# |
| -------- | ------- | ---------- |
| 1        | 0 ns    | 0          |
| 2        | 96 ns   | 49         |
| 3        | 192 ns  | 98         |
| ..       | ..      | ..         |
| 105      | 10.0 μs | 25         |
| ..       | ..      | ..         |
| 209      | 20.0 μs | 1          |
| ..       |         |            |
| 5120     | 493 μs  | 5071       |