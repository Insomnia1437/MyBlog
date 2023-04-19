---
title: 粒子加速器控制-入门指南
categories:
  - Accelerator Control
tags:
  - EPICS
  - Control
  - KEK
  - Linux
  - Accelerator
  - Chinese
abbrlink: 3ea1f950
date: 2019-12-10 21:43:08
---

```python
# @Time    : 2019-12-10
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

## Introduction

> 本系列主目录：
>  [粒子加速器控制](/posts/acc-control-learning-catalog)
------
下面列出**我认为**粒子加速器控制需要学习的一些知识，很多没有列出的细节可能是因为我一时想不起来，更可能是我也不太懂不敢妄言。
<!-- more -->
### step 0 入门知识

1. 习惯使用Unix(Linux)
   - ssh连接服务器与退出，选择自己喜欢的ssh软件
   - 常用命令：ls，cp，mv等等
   - 掌握如何使用Editor， vi，emacs等等
   - 中级技巧：grep，find，top，ping，管道，重定向，crontab等
   - 高级技巧：shell脚本编写
2. 了解基本编程语言
   - python的安装，库环境管理（conda，pip，virtualenv等等）
   - c/c++编译链接，make
3. 基础的网络知识
   - IP地址，子网掩码
   - DNS/hostname 
   - Firewall
4. 保持良好的记录习惯，留存日志等

### step 1 进阶知识

1. 了解EPICS概念
   - 了解常用record，如何编写自己的record，常用的field名及含义，link
   - 了解EPICS整体架构和层次
   - 什么是PV，OPI，IOC等
   - Channel Access (可选：EPICS 7中的PV Access)
   - EPICS的常用环境变量
2. 熟悉EPICS application的创建与编译
   - makeBaseApp.pl的使用
   - st.cmd的修改与运行 
3. 掌握常见EPICS extension的编译，比如procserv， gateway，Alarm Handler
4. 基本EPICS命令的使用
   - caget caput camonitor的使用
   - 编程语言中(如使用python pyepics库)如何调用上述命令
5. GUI
   - CSS(medem，edm) 升级版Phoebus（至少2019年是新的）
   - VDCT(姑且算它是吧)
   - PyDM

### step 2 硬件相关

1. 熟悉不同的机箱CAMAC,VME,MicroTCA等
2. PLC
3. 示波器的使用
4. stream device
    - 同步异步
    - 阻塞非阻塞
    - 熟悉各种总线接口 
5. Digital I/O
6. Analog I/O
7. FPGA

### step 3 数据传输与存储

1. 数据存储(Archive Data)
2. 大数据压缩传输
3. PV filter
4. Archive Appliance

### step 4 软件使用

> 视具体领域可能需要掌握的一些软件(写论文需要处理数据画图时)

1. Matlab
2. Labview
3. Python(numpy，matplotlib)
4. SAD(在KEK主要使用的一种加速器物理模拟程序，常用的还有Elegant，BMAD，PTC等)[可参见：Accelerator physics codes](https://en.wikipedia.org/wiki/Accelerator_physics_codes)

### step 5 系统管理员与开发者

> 面向资深加速器控制人员或EPICS开发者需要掌握的一些内容

1. 熟悉EPICS源代码，了解实现的机制，如callback，scan等
   - 了解CA的具体实现
   - 了解EPICS 7的具体实现
   - 编写Device Support
   - 编写Record Support
   - 编写Driver Support
2. 掌握服务器管理知识
   - 负载均衡
   - NFS
   - 磁盘管理与备份 RAID等
   - 网络诊断 IP管理 lsof netstat
   - IOC状态监控
   - 数据库管理
3. 控制系统的配置，如procserv， gateway
4. 监控和故障诊断系统
   - 温湿度监控
   - BPM beam bunch监控
   - RF系统监控
   - 磁铁系统监控
   - 安全连锁系统，机器保护，人身保护
   - 定时系统监控
