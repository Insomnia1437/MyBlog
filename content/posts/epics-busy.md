---
title: "epics busy"
date:  2024-05-10T18:26:06+02:00
# weight: 1
# aliases: ["/first"]
tags: ["Chinese", "EPICS"]
author: "DW"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
# description: "Desc Text."
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    # URL: "https://github.com/Insomnia1437/MyBlog/tree/master/content"
    URL: ""
    Text: "Source" # edit text
    appendFilePath: true # to append file path to Edit link
---
```python
# @Language: Markdown
# @Software: VS Code/MacDown/Typora/Vim
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

> See [epics examples](/posts/epics-examples)

当设定一个motor时, 我们需要确定设置已经完成, 再进行下一步操作(比如开启快门). 采集数据, 完成后然后关闭快门. 这中间, motor移动的时间不确定, 采集数据的时间不确定. 而使用轮询的方式又显得不可靠.

## 安装
```
example_DBD += busySupport.dbd
example_LIBS += busy
```

## 用法
首先使用put-callback修改一个record的值, 这个record的FLNK或OUT link指向busy record. 当busy record的值为1时, callback不会返回. 直到busy record的值变为0(通过CA 或者 asyn device support等), callback才会完成.

如果使用CA的话, 使用`-c`指定asynchronous put, `-w`指定等待时间, 默认为1秒.

```shell
caput –c –w 20 DemoDevice 5
caput –c –w 20 DemoDevice 1
caput –c DemoDevice 5
caput –c DemoDevice 4
```

```epics
# Demo of 'busy' record
#
# Basic idea:
# When writing a "1" to the busy record,
# it remains in the active/busy/processing state
# until something writes a "0" to it.
#
# This example simulates a slow device
# with a "DemoDevice" setpoint
# and a "DemoDeviceReadback" readback
# that slowly follows the setpoint.
#
# Writing to the setpoints sets the busy record.
# Since the DemoDevice has an FLNK & OUT chain
# to the busy record, performing a "put callback"
# via channel access will thus hang while
# the busy record is active.
record(ai, "DemoDevice")
{
    field(DESC, "Setpoint")
    field(FLNK, "DemoDeviceStart")
}

record(ao, "DemoDeviceStart")
{
    field(DOL, "1")
    field(OUT, "DemoDeviceBusy PP")
}

record(busy, "DemoDeviceBusy")
{
    field(DESC, "BUSY Record")
}

# Readback slowly follows the setpoint
record(calc, "DemoDeviceReadback")
{
    field(DESC, "Readback")
    field(INPA, "DemoDeviceReadback")
    field(INPB, "DemoDevice")
    field(CALC, "(A>B)?(A-0.1):((A<B)?(A+0.1):B)")
    field(SCAN, ".1 second")
    field(PREC, "2")
    field(FLNK, "DemoDeviceDone")
}

# When readback matches the setpoint,
# busy record is cleared and a pending
# "put callback" will complete
record(calcout, "DemoDeviceDone")
{
    field(INPA, "DemoDeviceReadback")
    field(INPB, "DemoDevice")
    field(CALC, "ABS(A-B)>0.2")
    field(OOPT, "Transition To Zero")
    field(OUT,  "DemoDeviceBusy PP")
}
```

## asyn

busy record有三种device support,
```
device(busy,CONSTANT,devBusySoft,"Soft Channel")
device(busy,CONSTANT,devBusySoftRaw,"Raw Soft Channel")
device(busy,INST_IO,asynBusyInt32,"asynInt32")
```

对于asyn device support, 在使用autosave时初始化有一些问题, 详见: https://epics-modules.github.io/busy/busyReleaseNotes.html
