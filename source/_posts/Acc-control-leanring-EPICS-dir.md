---
title: EPICS目录结构
date: 2019-12-14 22:27:48
categories:
  - "Accelerator Control"
tags:
  - "EPICS"
  - "Control"
  - "KEK"
  - "Accelerator"
  - "Chinese"
---

```python
# @Time    : 2019-12-14
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

## EPICS 目录结构

初学者对于如何放置EPICS base目录 ioc目录 extensions目录总是感到混乱，因此总结下我的使用经验。

### 我的目录方案

先分享下我的目录放置位置，然后再解释原因，在`/home/yourusername/epics`目录下，我安装了多个EPICS Base版本。


- R3.15.6
  - base
    - configure
    - configure/os
    - documentation
    - src
    - startup
  - ioc
    - ioc-vxworks-1
    - ioc-linux-x86_64-1
    - ioc-vxworks-2
    - ...
  - extensions
    - configure
    - src
      - ca-gateway-2.1.2
      - procServ-2-8-0App
      - msi-7
      - ...
- R3.14.12.5
  - base
    - config
    - configure
    - configure/os
    - documentation
    - src
    - startup
  - ioc
  - extensions
- R7.0.3
  - base
  - ioc
  - extensions
- ...

以3.15.6版本为例，base目录放置base，编译之后基本不需改动，ioc目录下可以创建多个ioc application，便于管理。如需切换base，只需要修改`EPICS_BASE` 环境变量即可，以下为我的shell环境变量设置

```Makefile
export  EPICS_BASE='/home/sdcswd/epics/R314128/base'
export  EPICS_HOST_ARCH='linux-x86_64'
export  EPICS_EXTENSIONS="$EPICS_BASE/../extensions"
```

还有一个好处在于extensions安装时。在[此处：EPICS Main Page](https://epics.anl.gov/download/extensions/index.php)下载Extensions Top后解压至此目录不需修改`configure/RELEASE`文件下的`EPICS_BASE`，因为默认设置和我们的目录设置相同。

```Makefile
# Location of external products
EPICS_BASE=$(TOP)/../base
#EPICS_EXTENSIONS = /usr/local/epics/extensions
EPICS_EXTENSIONS=$(TOP)
```

### EPICS module 和 extension

关于module 和 extension 分不清楚的问题，我查阅了部分资料，得出的结论是实际上现在二者基本也不作区分，原则上extensions需要放置在`extensions/src`目录下，而module可放置在任意目录。

### 版本管理

可能你已经敏锐的意识到这种目录设置虽然方便管理不同版本的EPICS Release，但是对于不同版本的extensions或modules还是很麻烦，尤其是如今非常多module之间存在依赖关系，子module升级后引用它的module也不得不重新编译，模块间的交叉引用对于数十年的大型项目管理很可能成为灾难。

因此下一篇文章我会介绍一下关于EPICS模块管理的模块[Support Module Manager (SUMO)](https://goetzpf.bitbucket.io/sumo/introduction.html)

> **20191215更新：**

经过周末数个小时的研究，`SUMO`模块学习曲线过高，文档描述逻辑复杂，不建议使用。

