---
title: 粒子加速器控制-EPICS Guide
categories:
  - Accelerator Control
tags:
  - EPICS
  - Control
  - Accelerator
  - Chinese
abbrlink: 3c837064
date: 2020-11-22 16:48:20
---

```python
# @Time    : 2020-11-22
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

## EPICS AppDevGuide

> 本系列主目录：
> {% post_link Acc-control-learning-catalog 粒子加速器控制%}

这篇文章用于记录一些EPICS Application Developer's Guide中的内容，由于个人习惯，使用的版本为EPICS [3.15.5](https://epics.anl.gov/base/R3-15/5-docs/AppDevGuide/AppDevGuide.html)。因为已经写过如何编译epics base和epics ioc，对于这些基础流程就不多做介绍。（btw，发现EPICS居然已经支持iOS了，似乎是iOS模拟器，有空可以自己玩下）

<!-- more -->

### CH2 Getting Started

IOC Application 类型主要用的是两种`ioc`和`support`。主要差别在于Makefile里的rule不同。如果使用`example`类型，那在它的Makefile里两种rule都包括在内。

```shell
$ makeBaseApp.pl -l
Valid application types are:
        caClient
        caServer
        example
        ioc
        support
Valid iocBoot types are:
        example
        ioc
```

对于`support`类型：

```makefile
# xxxRecord.h will be created from xxxRecord.dbd 
DBDINC += xxxRecord 
DBD += myexampleSupport.dbd 
 
LIBRARY_IOC += myexampleSupport 
 
myexampleSupport_SRCS += xxxRecord.c 
myexampleSupport_SRCS += devXxxSoft.c 
myexampleSupport_SRCS += dbSubExample.c 
 
myexampleSupport_LIBS += $(EPICS_BASE_IOC_LIBS)
```

对于`ioc`类型：

```makefile
PROD_IOC = myexample 
 
DBD += myexample.dbd 
 
# myexample.dbd will be made up from these files: 
myexample_DBD += base.dbd 
myexample_DBD += xxxSupport.dbd 
myexample_DBD += dbSubExample.dbd 
 
# <name>_registerRecordDeviceDriver.cpp will be created from <name>.dbd 
myexample_SRCS += myexample_registerRecordDeviceDriver.cpp 
myexample_SRCS_DEFAULT += myexampleMain.cpp 
myexample_SRCS_vxWorks += -nil- 
 
# Add locally compiled object code 
myexample_SRCS += dbSubExample.c 
 
# Add support from base/src/vxWorks if needed 
myexample_OBJS_vxWorks += $(EPICS_BASE_BIN)/vxComLibrary 
 
myexample_LIBS += myexampleSupport 
myexample_LIBS += $(EPICS_BASE_IOC_LIBS)
```



