---
title: "epics examples"
date:  2024-05-07T09:56:17+02:00
# weight: 1
# aliases: ["/first"]
tags: ["Chinese"]
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

缘由: 最近看到了[Gabriel](https://github.com/gabrielfedel)创建的项目

- [epics-examples](https://gitlab.esss.lu.se/e3/epics-examples)
- [database-examples](https://github.com/epics-docs/database-examples)

感觉是应用费曼学习法的良机. 于是决定创建此文, 记录相关内容.

正如Gabriel所写:

> This repository aims to collect examples of EPICS Databases to help newcomers (or not so new) get an idea on how to use different records.

即便已使用epics多年, 对一些record type依然不熟悉, 或源于需求少或源于版本更新带来的功能变更. 往往导致有简单方法却使用了更复杂的实现方式.

此文分为三部分, 一部分是epics database的示例, 另一部分是常用epics module的示例, 还有一些其它内容的示例. 一些epics module (如busy), 会创建了新的record type, 依然归入module. 而一些复杂的module, 如asyn, stream device则难以简单完成示例, 因此会另起专文记述.

由于内容较多, 会在闲暇时陆续更新, 此文大概会花费数年才能完成.


## Module

### autosave
- [epics autosave](/posts/epics-autosave)

### caputlog

### iocstats

### recsync

### calc

### busy

### sequencer

## Others

### Access Security
- [epics access security](/posts/epics-access)

### record patch
可以把record定义在一个db file中, 然后在另一个db file中添加这个record的一些field (比如逻辑处理相关field). 但是要注意, `dbLoadRecords`的先后顺序.

也可以不写record的type, 用"*"来代替.
> 最近在GitHub issue上看到有人在提议添加功能来反向patch(即删除某些record), 估计不久后会加入主线.

### field modifiers
历史包袱又一例, 具体因果暂时未能厘清, 但大致源于以下一些限制:
- epics record name 长度限制 40 -> 60
- epicsString 大小限制为 `MAX_STRING_SIZEP` (40)
- CA 使用的`DBR_STRING`大小限制为`MAX_STRING_SIZEP` (40)

而record name又会被使用在db link中, (INP, DOL, OUT, FWD), 所以就出现了一个问题, 当某个link field使用了大于40字节的record name时, 无法通过caget caput来读写. 于是引入了`$`作为field modifier. 当在link field后面加上`$`后, 就不会使用string, 而是使用char数组.
```shell
$ caput -S long_string.INP$  SomeReallyLongRecordNameThatExceeds40Characters
Old : long_string.INP$ SomeReallyLongRecordNameThatExceeds40Ch NPP NMS
New : long_string.INP$ SomeReallyLongRecordNameThatExceeds40Characters NPP NMS

$ caput -S long_string.INP  SomeReallyLongRecordNameThatExceeds40Characters
Old : long_string.INP                SomeReallyLongRecordNameThatExceeds40Ch
Error from put operation: Invalid element count requested

$ caget -S long_string.INP$
long_string.INP$ SomeReallyLongRecordNameThatExceeds40Characters NPP NMS

$ caget -S long_string.INP
long_string.INP                SomeReallyLongRecordNameThatExceeds40Ch
```

类似的, `lsi`和`lso` record的VAL field也可以使用`$`来修改CA的行为.
## Database (record type)
-------------
### Analog Array Input Record (aai)
### Analog Array Output Record (aao)
### Analog Input Record (ai)
### Analog Output Record (ao)
### Array Subroutine Record (aSub)
### Binary Input Record (bi)
### Binary Output Record (bo)
可以用`HIGH`field来判断ioc是否存活. 如下例, `i_am_alive`持续写入`deadIfZero`, 如果使用hardware interrupt来process`i_am_alive`, 当持续5秒未收到interrupt, 则`deadIfZero`会因为`HIGH`field, 在5秒后自动归零.
```epics
record(bo, "i_am_alive") {
    field(DTYP, "Soft Channel")
    field(SCAN, "1 second")
    field(DOL, "1")
    field(UDF, "0")
    field(ZNAM, "0")
    field(ONAM, "1")
    field(OUT, "deadIfZero.VAL PP")
}

record(bo, "deadIfZero") {
    field(DTYP, "Soft Channel")
    field(ZNAM, "0")
    field(ONAM, "1")
    field(HIGH, "$(DEAD_SECONDS=5)")
}
```
### Calculation Output Record (calcout)
### Calculation Record (calc)
### Compression Record (compress)
### Data Fanout Record (dfanout)
### Event Record (event)
### Fanout Record (fanout)
### Histogram Record (histogram)
### 64bit Integer Input Record (int64in)
### 64bit Integer Output Record (int64out)
### Long Input Record (longin)
### Long Output Record (longout)
### Long String Input Record (lsi)
### Long String Output Record (lso)
### Multi-Bit Binary Input Direct Record (mbbiDirect)
### Multi-Bit Binary Input Record (mbbi)
### Multi-Bit Binary Output Direct Record (mbboDirect)
### Multi-Bit Binary Output Record (mbbo)
### Permissive Record (permissive)
### Printf Record (printf)
### Select Record (sel)
### Sequence Record (seq)
### State Record (state)
### String Input Record (stringin)
### String Output Record (stringout)
### Sub-Array Record (subArray)
### Subroutine Record (sub)
### Waveform Record (waveform)
