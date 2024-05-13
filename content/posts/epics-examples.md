---
title: "epics examples"
date:  2024-05-07T09:56:17+02:00
weight: 1
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
#### 用法
log可以有JSON格式, 需要epics 7.0.1以上.

server可以使用iocLogServer.
```makefile
<IOC app name>_DBD += caPutLog.dbd  # For standard format
<IOC app name>_DBD += caPutJsonLog.dbd  # For JSON format (Exists only if module is compiled with supported version of base)
<IOC app name>_LIBS += caPutLog  # Required for both output formats
```

需要首先配置[Access security](/posts/epics-access), 对于`DEFAULT`group, 添加`RULE(1,WRITE,TRAPWRITE)`. 然后在iocsh中, 设置
```shell
caPutLogInit "host[:port]" [config]
caPutJsonLogInit "host[:port]" [config]
# for example
caPutLogInit "localhost:7011" 0
caPutJsonLogInit $(CAPUTLOG_INET):$(CAPUTLOG_INET_PORT=8002) $(CAPUTLOG_OPTION=0)

# 重新加载配置
caPutLogReconf config
caPutJsonLogReconf config

# info
caPutLogShow level
caPutJsonLogShow level

# 修改日期格式
caPutLogSetTimeFmt "<date_time_format>"

# log to a PV
epicsEnvSet(EPICS_AS_PUT_LOG_PV, "pv.VAL$")
epicsEnvSet(EPICS_AS_PUT_JSON_LOG_PV, "pv.VAL$")

# debug
var caPutLogDebug,1
```
#### 示例
首先启动iocLogServer. 启动命令如下.
```shell
INSTALL_BIN=/home/sdcswd/epics/R7.0.8/base/bin/linux-x86_64

# To change the default values for the EPICS Environment parameters,
# uncomment and modify the relevant lines below.

# EPICS_IOC_LOG_PORT="6500" export EPICS_IOC_LOG_PORT
EPICS_IOC_LOG_FILE_NAME="ioclog.log" export EPICS_IOC_LOG_FILE_NAME
# EPICS_IOC_LOG_FILE_LIMIT="1000000" export EPICS_IOC_LOG_FILE_LIMIT

if [ $1 = "start" ]; then
    if [ -x $INSTALL_BIN/iocLogServer ]; then
        echo "Starting EPICS Log Server "
        $INSTALL_BIN/iocLogServer &
    fi
else
    if [ $1 = "stop" ]; then
        pid=`ps -e | sed -ne '/iocLogSe/s/^ *\([1-9][0-9]*\).*$/\1/p'`
        if [ "${pid}" != "" ]; then
            echo "Stopping EPICS Log Server "
            kill ${pid}
        fi
    fi
fi
```
然后在st.cmd中
```shell
dbLoadTemplate("db/user.substitutions")

epicsEnvSet(EPICS_IOC_LOG_INET, "localhost")
#epicsEnvSet(EPICS_IOC_LOG_PORT, "$(LOG_INET_PORT)")
epicsEnvSet(iocLogDisable, "$(LOGDISABLE=0)")

asSetFilename("access.acf")
iocLogPrefix("${IOC} ")

cd "${TOP}/iocBoot/${IOC}"
iocInit

caPutLogInit "localhost:7004" 0
iocLogInit()
```

然后在ioclog.log文件中, 可以同时得到iocLog(前两行)和caPutLog(后两行)的内容.
```
localhost:55142 Sun May 12 15:26:47 2024 iocexample iocPause: IOC suspended
localhost:55142 Sun May 12 15:26:47 2024 iocexample iocRun: IOC restarted
localhost:55128 Sun May 12 15:26:57 2024 iocexample 12-May-24 15:26:51 zephyrus sdcswd sdcswd:circle:step new=2 old=1
localhost:55128 Sun May 12 15:27:02 2024 iocexample 12-May-24 15:26:57 zephyrus sdcswd sdcswd:circle:step new=1 old=2
```
### iocstats

### recsync
用于Channel Finder. 也就是把ioc中的record信息, epics environment以及info tag发送到一个数据库.

但目前整个服务显得很繁杂, 要先开启Elasticsearch, 然后开启Channel Finder Service, 然后开启recsync中的server, 也叫`RecCeiver`, (这个名字很糟糕, 因为它其实是一个中间服务, 负责接受ioc发过来的内容, 然后发给Channel Finder Service), 最后开启带有recsync中的client功能(`RecCaster`)的ioc. client使用`5049`UDP端口.

server会调用[pyCFClient module](https://github.com/ChannelFinder/pyCFClient), 创建Python class `ChannelFinderClient`, 与Channel Finder Service通信, 通信时使用了默认的`8080`端口.

可以使用[https://github.com/ChannelFinder/RecSync-env](https://github.com/ChannelFinder/RecSync-env)来简化部署.

以下为测试recsync的流程, 不涉及Channel Finder Service.
#### RecCaster
在ioc中添加reccaster依赖,
```makefile
test_DBD += reccaster.dbd
test_LIBS += reccaster
```
在st.cmd中, 可以配置RecCaster
```shell
var(reccastTimeout, 5.0)
var(reccastMaxHoldoff, 5.0)
epicsEnvSet("CONTACT", "mycontact")
epicsEnvSet("BUILDING", "mybuilding")
epicsEnvSet("SECTOR", "mysector")

addReccasterEnvVars("CONTACT", "SECTOR")
addReccasterEnvVars("BUILDING")

iocInit
```
#### RecCeiver
开启RecCeiver的流程
```shell
$ cd recsync/server
$ python3 -m venv venv
$ source venv/bin/activate
$ pip3 install --upgrade pip
$ pip3 install requests
$ pip3 install .
$ pip3 list
Package            Version
------------------ --------
attrs              23.2.0
Automat            22.10.0
certifi            2024.2.2
channelfinder      3.0.0
charset-normalizer 3.3.2
constantly         23.10.4
hyperlink          21.0.0
idna               3.7
incremental        22.10.0
pip                24.0
recceiver          1.5
requests           2.31.0
setuptools         66.1.1
simplejson         3.19.2
six                1.16.0
Twisted            24.3.0
typing_extensions  4.11.0
urllib3            2.2.1
zope.interface     6.3
# 或者直接指定PYTHONPATH
$ export PYTHONPATH=${PWD}:${PWD}/build/lib:${channelfinder_path}
$ twistd -r poll -n recceiver -f demo.conf
```

### calc

### busy
- [epics busy](/posts/epics-busy)

### sequencer
- [epics sequencer](/posts/epics-sequencer)

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

### Channel Filters and IOC Database Link types

- [filter](https://epics.anl.gov/base/R7-0/8-docs/filters.html)
- [links](https://epics.anl.gov/base/R7-0/8-docs/links.html)

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
