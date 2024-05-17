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

-------------
## Module
### autosave
- [epics autosave](/posts/epics-autosave)
-------------
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
-------------
### iocstats
-------------
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
-------------
### calc
-------------
### busy
- [epics busy](/posts/epics-busy)
-------------
### sequencer
- [epics sequencer](/posts/epics-sequencer)

## Others
-------------
### Access Security
- [epics access security](/posts/epics-access)
-------------
### record patch
可以把record定义在一个db file中, 然后在另一个db file中添加这个record的一些field (比如逻辑处理相关field). 但是要注意, `dbLoadRecords`的先后顺序.

也可以不写record的type, 用`*`来代替.
> 最近在GitHub issue上看到有人在提议添加功能来反向patch(即删除某些record), 估计不久后会加入主线.

-------------
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

-------------
### Channel Filters and IOC Database Link types

- [filter](https://epics.anl.gov/base/R7-0/8-docs/filters.html)
- [links](https://epics.anl.gov/base/R7-0/8-docs/links.html)

-------------
### breakpoint table
**TODO**

-------------
### Echoless IOC Comments
启动ioc时, 可以在st.cmd文件中使用`#-`来不让注释行输出. 非常有用.
示例
```shell
$ cat st.cmd
# print this comment
#- not print this comment
dbLoadRecords a.db

$ softIocPVA st.cmd
# print this comment
dbLoadRecords a.db
iocInit
Starting iocInit
############################################################################
## EPICS R7.0.8
## Rev. 2024-03-27T10:23+0100
## Rev. Date build date/time:
############################################################################
iocRun: All initialization complete
epics>
iocInit
```

-------------
## Database (record type)
### dbCommon
| field   | information                                                                      |
| :------ | :------------------------------------------------------------------------------- |
| `NAME`  | 60 characters                                                                    |
| `DESC`  | 40 characters                                                                    |
| `ASG`   | 29 characters                                                                    |
| `SCAN`  | `menuScan`                                                                       |
| `PINI`  | `menuPini`                                                                       |
| `PHAS`  | 不建议使用                                                                       |
| `EVNT`  | `Event` scan类型                                                                 |
| `DTYP`  | Device Type                                                                      |
| `TSE`   | Time Stamp Event, 0, -1, -2, \[1-255\]                                           |
| `TSEL`  | Time Stamp Link                                                                  |
| `TIME`  | epicsTimeStamp time                                                              |
| `DISS`  | Disable Alarm Sevrty, `menuAlarmSevr`                                            |
| `DISV`  | Disable Value, 只是不process, value依然可以改变, 与`DISA`比较, 相当的话disable   |
| `DISA`  | Disable, DBF_SHORT                                                               |
| `SDIS`  | Scanning Disable, INLINK, 读取后放入`DISA`                                       |
| `MLOK`  | Monitor lock, epicsMutexId                                                       |
| `MLIS`  | Monitor List, ELLLIST                                                            |
| `BKLNK` | Backwards link tracking, ELLLIST                                                 |
| `DISP`  | Disable putField, 非0时不可通过CA或PVA修改field(除DISP)                          |
| `PROC`  | Force Processing, 写入时process record, 写0也可                                  |
| `STAT`  | Alarm Status, `menuAlarmStat`                                                    |
| `SEVR`  | Alarm Severity, `menuAlarmSevr`                                                  |
| `AMSG`  | Alarm Message                                                                    |
| `NSTA`  | New Alarm Status, 用于内部处理过程, 处理完成后将最严重的级别更新到`STAT`         |
| `NSEV`  | New Alarm Severity                                                               |
| `NAMSG` | New Alarm Message                                                                |
| `ACKS`  | Alarm Ack Severity, highest unacknowledged alarm severity                        |
| `ACKT`  | Alarm Ack Transient, Alarm Ack Transient, `menuYesNo`                            |
| `LCNT`  | Lock Count, 记录`PACT`为`TRUE`时的数量, 超过10, 那就触发`SCAN_ALARM`             |
| `PACT`  | Record active, 为`TRUE`时代表record正被process, 类型是`DBF_UCHAR`而非`menuYesNo` |
| `PUTF`  | dbPutField process, 不详                                                         |
| `RPRO`  | Reprocess, 不详                                                                  |
| `ASP`   | Access Security Pvt, struct asgMember                                            |
| `PPN`   | pprocessNotify, putNotify callback, struct processNotify                         |
| `PPNR`  | pprocessNotifyRecord, next record for PutNotify, struct processNotifyRecord      |
| `SPVT`  | Scan Private,  struct scan_element                                               |
| `RSET`  | struct typed_rset                                                                |
| `DSET`  | unambiguous_dset                                                                 |
| `DPVT`  | Device Private, void *dpvt                                                       |
| `RDES`  | Address of dbRecordType, struct dbRecordType                                     |
| `LSET`  | Lock Set, struct lockRecord                                                      |
| `PRIO`  | Scheduling Priority, `menuPriority`                                              |
| `TPRO`  | Trace Processing, 非0时打印trace信息                                             |
| `BKPT`  | Break Point, 调试用断点, epicsUInt8 bkpt                                         |
| `UDF`   | Undefined                                                                        |
| `UDFS`  | Undefined Alarm Severity, `menuAlarmSevr`                                        |
| `UTAG`  | Time Tag                                                                         |
| `FLNK`  | Forward Process Link                                                             |

对于`TSE`与`TSEL`:

1. 当`TSEL`指向另一个record的`TIME` field时, 使用获取到的`TIME`作为本record的timestamp. 如果是同一个ioc, `UTAG`也被复用.
2. 当`TSEL`指向另一个record其它field时, 那个field的值被放到`TSE`中.
   1. 当`TSE`为0, 会从current time providers中找到优先级最高的, 获取timestamp.
   2. 当`TSE`为-1, 会从event time providers中找到优先级最高的, 然后由这个provider来提供`event=-1`的timestamp. 需要调用`generalTimeEventTpRegister`来注册event provider.
   3. 当`TSE`为-2, 从device support中获取timestamp和user tag, 由device support来决定读取time provider还是event provider还是不注册provider直接读硬件.
   4. 当`TSE`为\[1, 255\], 从event providers中找到优先级最高的, 然后由这个 provider提供对应event发生时的timestamp.

相关函数
```c
// base中的一些macro
#define generalTimeCurrentTpName generalTimeCurrentProviderName
#define generalTimeEventTpName generalTimeEventProviderName

#define epicsTimeEventCurrentTime 0
#define epicsTimeEventBestTime -1
#define epicsTimeEventDeviceTime -2

int epicsShareAPI epicsTimeGetEvent(epicsTimeStamp *pDest, int eventNumber)
{
    if (eventNumber == epicsTimeEventCurrentTime) {
        // TSE为0的情况
        return epicsTimeGetCurrent(pDest);
    } else {
        // TSE为-1或[1-255]的情况
        return generalTimeGetEventPriority(pDest, eventNumber, NULL);
    }
}

// mrfioc2 处理longout record中TSE=-2的情况
static long process_longout(longoutRecord *prec)
{
    priv *p=static_cast<priv*>(prec->dpvt);
    long ret=0;

    if (prec->val>=0 && prec->val<=255)
        post_event(prec->val);

    if(prec->tse==epicsTimeEventDeviceTime){
        p->evr->getTimeStamp(&prec->time,p->event);
    }
}
```

对于disable record, 使用Access security可以禁止record被读写, 也可以用`DISV`和`SDIS`禁止record被process.
```
record(ai, ai){
  field(DISV, "1")
  field(DISS, "MAJOR")
  field(VAL,  "311")
  field(SDIS, "bi PP")
}

record(bi, bi){
  field(INP, "1")
  field(PINI, "YES")
}
```

#### Mostly Common
| field  | information                                                                                     |
| ------ | ----------------------------------------------------------------------------------------------- |
| `RVAL` | 对于input, 从hardware中读取的原始值, 待转换; 对于output, 转换后发送给hardware的值.              |
| `OVAL` | Output Value                                                                                    |
| `RBV`  | Readback Value                                                                                  |
| `DOL`  | Desired Output Link                                                                             |
| `OMSL` | `supervisory` or `closed_loop`                                                                  |
| `ORAW` | Previous Raw Value                                                                              |
| `PVAL` | Previous value                                                                                  |
| `IVOA` | INVALID output action, 三个选项`Continue normally`, `Don't drive outputs`和`Set output to IVOV` |
| `IVOV` | INVALID output value                                                                            |
| `NORD` | Number elements read, 用于waveform和aai                                                         |

#### Simulation
开启simulation mode时, 会忽略device support. 不知道有什么实际用处.

| field     | information                      |
| --------- | -------------------------------- |
| `SIOL`    | Simulation Input/Output Link     |
| `SVAL`    | Simulation Value                 |
| `SIML`    | Simulation Mode Link             |
| `SIMM`    | Simulation Mode                  |
| `SIMS`    | Simulation Mode Severity         |
| `OLDSIMM` | Prev. Simulation Mode            |
| `SSCN`    | Sim. Mode Scan                   |
| `SDLY`    | Sim. Mode Async Delay            |
| `SIMPVT`  | Sim. Mode Private, epicsCallback |
```
           --  (SIMM = NO?)
         /         (if supported and directed by device support,
        /              INP -> RVAL -- convert -> VAL),
                   (else INP -> VAL)
 SIML -> SIMM

         \
           --  (SIMM = YES?) SIOL -> SVAL -> VAL
           \
             -- (SIMM = RAW?) SIOL -> SVAL -> RVAL -- convert -> VAL
```

-------------
### Analog Input Record (ai)
#### Unit conversion
当device support为`Raw Soft Channel`时才会转换.

| field  | information                       |
| ------ | --------------------------------- |
| `RVAL` | Current Raw Value                 |
| `ROFF` | Raw Offset                        |
| `ASLO` | Adjustment Slope                  |
| `AOFF` | Adjustment Offset                 |
| `LINR` | Linearization	MENU, `menuConvert` |
| `ESLO` | Raw to EGU Slope                  |
| `EOFF` | Raw to EGU Offset                 |
| `EGUL` | Engineer Units Low                |
| `EGUF` | Engineer Units Full               |
| `PBRK` | Pointer to brkTable               |
| `LBRK` | Last brkTable                     |


1. (`RVAL` + `ROFF`)
2. 如果ASLO不为0, 乘以`ASLO`
3. 再加上`AOFF`
4. 判断`LINR`的值:
   1. 如果为`NO CONVERSION`, 结束转换
   2. 如果为`SLOPE`, 乘以`ESLO`, 再加上`EOFF`
   3. 如果为`LINEAR`, 由`special_linconv`函数根据用户设置的`EGUL`和`EGUF`, 以及hardware的实际量程来计算出`ESLO`和`EOFF`, 具体计算规则根据不详.
   4. 其它选项, 使用breakpoint table.


#### Smoothing Filter
`SMOO`值为0-1, 0的话代表不使用filter, 1的话代表infinite smoothing, 计算公式如下.

`VAL = VAL * SMOO + (1 - SMOO) * New Data`


#### Display
| field  | information                                                                                                                                      |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `PREC` | Display Precision                                                                                                                                |
| `EGU`  | Engineering Units, 16 characters                                                                                                                 |
| `HOPR` | High Operating Range, 设置一些控件的上限                                                                                                         |
| `LOPR` | Low Operating Range, 设置一些控件的下限                                                                                                          |
| `HIHI` |                                                                                                                                                  |
| `LOLO` |                                                                                                                                                  |
| `HIGH` |                                                                                                                                                  |
| `LOW`  |                                                                                                                                                  |
| `HHSV` |                                                                                                                                                  |
| `LLSV` |                                                                                                                                                  |
| `HSV`  |                                                                                                                                                  |
| `LSV`  |                                                                                                                                                  |
| `HYST` | 加入hysteresis, 比如low limit是10, `HYST`为1, 当val从11降为10.5时无alarm, 但当val从9升为10.5时, 依然触发alarm, 直到vale > 10 + 1时, alarm才消失. |
| `AFTC` | Alarm Filter Time Constant, 类似与`HYST`, 不过是当val持续一定时间后再触发alarm.                                                                  |
| `AFVL` | Alarm Filter Value, `AFTC`计算过程中使用的变量.                                                                                                  |
| `MDEL` | Monitor Deadband, value monitors, 当val变化超过deadband时才会发送monitor. 0的话每次val改变都触发, -1时record process时触发.                      |
| `ADEL` | Archive Deadband, archive monitors                                                                                                               |
| `LALM` | Last Value Alarmed                                                                                                                               |
| `ALST` | Last Value Archived                                                                                                                              |
| `MLST` | Last Val Monitored                                                                                                                               |
| `INIT` | Initialized?                                                                                                                                     |

#### 示例

```epics
record(ai, ai){
  field(VAL,  "13")
  field(HOPR, "0")
  field(LOPR, "100")
  field(PREC, "3")
  field(HIHI, "90")
  field(HIGH, "80")
  field(LOW,  "20")
  field(LOLO, "10")
  field(HHSV, "MAJOR")
  field(HSV,  "MINOR")
  field(LSV,  "MINOR")
  field(LLSV, "MAJOR")
  field(PINI, "RUNNING")
  field(HYST, "1")
}
```

以下为`HSYT`的示例, 可以看到VAL从13变为21时, 依然有low alarm.
![ai record alarm](/images/epics-example-ai.png)

-------------
### Analog Output Record (ao)

ao的field和ai基本相同. 一些特殊的field:
- `OIF`: 可选项为`Incremental`和`Full`, 也就是output之前加上`PVAL`的值.
- `DRVL`: Drive Low Limit, 输出的下限
- `DRVH`: Drive High Limit 输出的上限
- `OROC`: Output Rate of Change, 设置与上次输出的`OVAL` field相比能改变的最大值.


对于record中的一些preview value, 命名有些奇怪, 有时候是`PVAL`, 有时候又是`OVAL`, 即Old value.
在ao record中, `OVAL`代表Output value, 所以不称呼为Old value可以理解.
但又把`ORAW`叫Previous Raw Value, 把`ORBV`叫Previous Readback Value. 真是很让人困惑

-------------
### Analog Array Input Record (aai)
初始化时可以用支持JSON的link. 引号在使用Relaxed JSON时可以省略, 但规则不详.

`MPST`与`APST`控制何时触发monitor, 他们使用了`HASH`field来判断值是否发生变化.
但鉴于可能的哈希碰撞(Hash Collision), 所以最好使用`Always`, 也就是每次process就会触发monitor.
```
record(aai, aai){
  field(NELM, "10")
  field(FTVL, "SHORT")
  field(APST, "On Change")
  field(MPST, "Always")
#  field(INP,  "[3,1,1]")
  field(INP,  "{const: [315, 10, 0, 0, 1]}")
}

record(aao, aao){
  field(OMSL, "closed_loop")
  field(DOL,  "aai")
  field(PINI, "RUNNING")
  field(FTVL, "SHORT")
  field(NELM, "10")
  field(OUT,  "wf")
}
record(waveform, wf){
  field(FTVL, "LONG")
  field(NELM, "20")
}
```

-------------
### Analog Array Output Record (aao)
示例见`aai`

-------------
### Waveform Record (waveform)
示例见`aai`

比`aai`多的field: 基本都不再使用了.
- `RARM`: Rearm the waveform
- `BUSY`: Busy Indicator

-------------
### Binary Input Record (bi)
`COSV`可以用来设置value改变时的alarm, 可以设置为`MINOR or MAJOR`.
如下例, 当value为0时, 触发MAJOR, value从0变1时, 触发MINOR, value从1变1时, 触发NO_ALARM.

device support可以使用`MASK`field. 不详.
```
record(bi, bi){
  field(VAL,  "0")
  field(ZNAM, "OFF")
  field(ONAM, "ON")
  field(ZSV,  "MAJOR")
  field(OSV,  "NO_ALARM")
  field(COSV, "MINOR")
  field(PINI, "YES")
}
```

-------------
### Binary Output Record (bo)
可以用`HIGH`field来判断ioc是否存活. 如下例, `i_am_alive`持续写入`deadIfZero`, 如果使用hardware interrupt来process`i_am_alive`, 当持续5秒未收到interrupt, 则`deadIfZero`会因为`HIGH`field, 在5秒后自动归零.
```epics
record(bo, "i_am_alive") {
    field(DTYP, "Soft Channel")
    field(SCAN, "1 second")
    field(DOL, "1")
    field(UDF, "0")
    field(ZNAM, "Dead")
    field(ONAM, "Alive")
    field(OUT, "deadIfZero.VAL PP")
}

record(bo, "deadIfZero") {
    field(DTYP, "Soft Channel")
    field(ZNAM, "Dead")
    field(ONAM, "Alive")
    field(HIGH, "$(DEAD_SECONDS=5)")
}
```
-------------
### Calculation Record (calc)
有12个input links.

```
# 简单的例子, heartbeat
record(calc, "calc"){
  field(VAL,  "0")
  field(CALC, "VAL?0:1")
  field(SCAN, "1 second")
}
# 复杂的例子, 产生连续正弦波, 初始相位可调
record(calc, "calc"){
  field(INPA, "1.57079632679489661923") # 初始相位为pi/2
  field(CALC, "A:=(A+D2R)>(2*PI)?0:(A+D2R); sin(A)")
  field(SCAN, ".1 second")
}
```

-------------
### Calculation Output Record (calcout)
相比与`calc`, 增加了一些output选项.
- `OOPT`: Output Execute Opt, 可选`Every Time`, `On Change`, `When Zero`, `When Non-zero`, `Transition To Zero`, `Transition To Non-zero`
- `DOPT`: Output Data Opt, 可选`CALC`, `OCAL`
- `OCAL`: Output Calculation, 和`CALC`一样包含运算表达式
- `OVAL`: Output Value, `OCAL`的输出结果
- `OEVT`: Event To Issue, 要触发的event
- `ODLY`: Output Execute Delay, 触发event和输出前的delay
- `DLYA`: Output Delay Active, delay过程中这个field会被设为1
- `INAV`-`INIV`: Input PV Status, 会被设为`Ext PV NC`, `Ext PV OK`, `Local PV`, `Constant`
- `OUTV`: Output PV Status, 同上
- `CLCV`: CALC Valid,
- `OCLV`: OCAL Valid,

1. 首先计算`CALC`, 得到的结果放到`VAL`里;
2. 然后判断`OOPT`
   1. 不满足什么也不做;
   2. 如果满足再判断`DOPT`,
      1. 如果用`CALC`, 那就把`VAL`赋值给`OVAL`
      2. 如果要用`OCAL`, 那就再计算得到`OVAL`
   3. 将`OVAL`输出到`OUT`

```
record(calcout, "co"){
  field(CALC, "VAL<10?VAL+1:0")
  field(OCAL, "90-89")
  field(OOPT, "When Zero")
  field(DOPT, "Use OCAL")
  field(OUT,  "res PP")
  field(SCAN, "1 second")
}

record(ao, "res") {
  field(TPRO, "1")
}
```

-------------
### Compression Record (compress)
两种工作模式, 当input link是数组时, 取所有数据然后计算. 如果不是, 那就每次process一次就把数据加入环形缓冲区进行计算.

使用的field, 其中关键的就是`NSAM`和`N`, 前者决定compress中存多少数据, 后者决定每多少个输入进行一次计算. `ILIL IHIL OFF`都只在`N to 1`类的算法中使用.
- `NSAM`: Number of Values
- `N`: N to 1 Compression
- `ILIL`: Init Low Interest Lim, 过滤数据的下限
- `IHIL`: Init High Interest Lim, 过滤数据的上限
- `OFF`: Offset, 从哪个数据开始算,
- `RES`: Reset

支持的compression algorithm, 也就是`ALG` field的值:
- `N to 1 Low Value`
- `N to 1 High Value`
- `N to 1 Average`
- `Average`
- `Circular Buffer`, 保持一个大小为`NSAM`的缓冲区.
- `N to 1 Median`

> `compress`的input link支持array field, 但只支持`N to 1`类的算法.

`BLAG`又可选缓冲区的类型, `FIFO Buffer` 或者 `LIFO Buffer`

`calc`每秒自增1, compress每隔10输入计算一次平均值, 最多存两个平均值.
```
record(calc, calc){
  field(CALC, "VAL+1")
  field(FLNK, "cp PP")
  field(SCAN, "1 second")
}
record(compress, cp) {
  field(INP,  "calc NPP")
  field(ALG,  "Average")
  field(BALG, "LIFO Buffer")
  field(NSAM, "2")
  field(N,    "10")
  field(PINI, "YES")
}
```
下例的效果, 计算10个输入的平均值, 然后放到`compress`的VAL field中. 使用`BALG`使得新值放到数组最前面.
```shell
$ camonitor cp
cp                             <undefined> UDF INVALID
cp                             2024-05-17 16:03:37.379709 4.5
cp 2024-05-17 16:03:47.379616 2 14.5 4.5
cp 2024-05-17 16:03:57.379640 2 24.5 14.5
```

-------------
### Event Record (event)
用来触发一个epics event, 比如下例会触发event 14.
```
record(event, event){
  field(INP, "14")
  field(TPRO, "1")
}

record(calc, calc){
  field(CALC, "VAL+1")
  field(SCAN, "Event")
  field(EVNT, "14")
  field(TPRO, "1")
  field(PRIO, "HIGH")
}
record(ai, ai){
  field(INP, "calc")
  field(SCAN, "Event")
  field(EVNT, "14")
  field(TPRO, "1")
}
```
如下所示`ai`和`calc`都被触发了, 但`ai`先触发, 所以`calc`变为1, 而`ai`依然为0.

即使加上`PRIO`也不影响, 可能与record load时的先后顺序有关, 不详.

所以event适用于无数据流关系的一些record触发, 而强数据流关系的还是用link合适.
```shell
epics> dbpf event.PROC 0
_main_: dbProcess of 'event'
DBF_UCHAR:          0 = 0x0
cbLow: dbProcess of 'ai'
cbHigh: dbProcess of 'calc'
# caput event.PROC 0
epics> ca:sdcswd@zephyrus: dbProcess of 'event'
cbLow: dbProcess of 'ai'
cbHigh: dbProcess of 'calc'
```
-------------
### Fanout Record (fanout)
`fanout`有16个输出link.

对于fanout类的record, 有三种`SELM`, 即Select Mechanism
- `All`:
- `Specified`: 从`SELL`得到值, 放到`SELN`, 然后加上`OFFS`, 然后触发对应的link. e.g., 1 -> LNK1.
- `Mask`: 把`SELN`右移`SHFT`位, 对应bit为1的会触发. e.g., 0x01 -> LNK0. 注意`SHFT`默认值为-1, 也就是默认会把`SELN`左移一位. 当`SELN`为2时, bit 3为1, `LNK2`被触发.


**Q**: 为什么`SHFT`默认值为-1?

**A**: 历史兼容性. 早期fanout的link是从1开始的, 而且只有6个输出link, 当新增`LNK0`后, 需要增加`SHFT`来兼容旧的record.

```epics
record(fanout, fo){
  # Specified, LNK1被触发
  field(SELM, "Specified")
  field(SELN, "0")
  field(OFFS, "1")

  # Mask, LNK1和LNK2被触发
  # field(SELM, "Mask")
  # field(SELN, "6")
  # field(SHFT, "0")

  field(LNK0, "calc0")
  field(LNK1, "calc1")
  field(LNK2, "calc2")
  field(LNK3, "calc3")
  field(LNK4, "calc4")
}
record(calc, calc0){
  field(CALC, "VAL+1")
}
record(calc, calc1){
  field(CALC, "VAL+1")
}
record(calc, calc2){
  field(CALC, "VAL+1")
}
record(calc, calc3){
  field(CALC, "VAL+1")
}
record(calc, calc4){
  field(CALC, "VAL+1")
}
```
-------------
### Data Fanout Record (dfanout)
有8个输出link.

对比`fanout`, 多了数据转发, 但是少了bit shift和offset field. 而且link field的命名也变为了`OUTA` -> `OUTH`.

- 对于`All`, 用于把一个value写到多个record.
- 对于`Specified`, 当`SELN`为0时, 不会输出, 为1时, 输出到`OUTA`, 和`fanout`不同.
- 对于`Mask`, LSB为1时, 输出到`OUTA`

```
record(dfanout, dfo){
  field(OMSL, "closed_loop")
  field(DOL,  "19")

  # Specified, SELN为0时无输出
  # field(SELM, "Specified")
  # field(SELN, "0")

  # Mask, 二进制为`101`, 所以输出到OUTA和OUTC
  field(SELM, "Mask")
  field(SELN, "5")

  field(OUTA, "calc0")
  field(OUTB, "calc1")
  field(OUTC, "calc2")
  field(OUTD, "calc3")
  field(OUTE, "calc4")
}
```
-------------
### Histogram Record (histogram)
计算histogram, 用到的field:
- `NELM`: Num of Array Elements
- `SVL`: Signal Value Location, 数据源
- `SGNL`: Signal Value, 数据
- `ULIM`: Upper Signal Limit, 低于上限才会被收录
- `LLIM`: Lower Signal Limit, 高于下限才会被收录
- `MDEL`: Monitor Count Deadband, 何时触发monitor, 与`MCNT`进行比较, `MCNT`大于`MDEL`时才触发monitor, 比如每收集100个数据触发一次monitor.
- `SDEL`: Monitor Seconds Dband, 定时检测`MCNT`, 当`MCNT`大于0时就触发monitor, 应该是用于数据稀少或非周期性到来的场景.
- `MCNT`: Counts Since Monitor, 每次有新数据就加1, 触发monitor之后就置0.
- `CMD`: Collection Control, 控制histogram, 清空或者暂停或者恢复读取. `Read`, `Clear`, `Start`, `Stop`
- `CSTA`: Collection Status, 内部状态位, 当它为`TRUE`时才会采集数据. 只有`Start`会设置它为`TRUE`. 而`Read`和`Clear`都只会清除已采集的数据, 也就是使用`Stop`的话必须使用`Start`才能开启采集. 当然也可以只用`Clear`来重新采集. `Read`命令和`Clear`没区别.
- `WDTH`: Element Width, 由`(ULIM - LLIM) / NELM` 计算width, 比如下限为4, 上限12, 四个bin, 那每个bin的`WDTH`就是2.

```
record(calc, "calc"){
  field(INPA, "1.57079632679489661923") # Phase
  field(CALC, "A:=(A+D2R)>(2*PI)?0:(A+D2R); sin(A)")
  field(SCAN, ".1 second")
  field(FLNK, "hist")
}

record(histogram, hist){
  field(SVL,  "calc NPP")
  field(NELM, "10")
  field(ULIM, "1")
  field(LLIM, "-1")
  field(MDEL, "20")
  field(SDEL, "0")
  field(CMD,  "Read")
}
```
以下为一个正弦波的直方图示例, 横坐标应该是从-1到1的十组, 但我不知道怎么修改Phoebus的横轴😞. 也许需要一个单独的waveform作为横轴吧.
![histogram record](/images/epics-example-hist.png)

另外, 发现了histogram的设计问题, 写入`Stop`后, 虽然的确停止采集了, 但返回值却依然是`Read`. 这样就无法使用`CMD`判断当前状态了. 必须要读取`CSTA`.

```shell
$ caput hist.CMD "Stop"
Old : hist.CMD                       Read
New : hist.CMD                       Read
```

-------------
### Long Input Record (longin)
32位整数, 支持`AFTC`.

-------------
### Long Output Record (longout)
32为整数, 支持`DRVH`和`DRVL`.

 支持`OOPT`. 不知为什么, `ao`, `int64out`都不支持`OOPT`.

-------------
### 64bit Integer Input Record (int64in)
和`longin`相似, 只不过支持64位长度.
```
record(longin, li){
}
record(int64in, i64){
}
```
允许超过32位上限
```shell
$ caput li 2147483647
Old : li                             111
New : li                             2147483647
sdcswd @ zephyrus in  ~/epics/R7.0.8/base [11:47]
$ caput li 2147483649
Old : li                             2147483647
CA.Client.Exception...............................................
New : li                             2147483647
    Warning: "Channel write request failed"
    Context: "op=1, channel=li, type=DBR_STRING, count=1, ctx="li""
    Source File: ../oldChannelNotify.cpp line 159
    Current Time: Thu May 16 2024 11:47:58.193643892
..................................................................
$ caput i64 2147483647
Old : i64                            0
New : i64                            2.14748e+09
sdcswd @ zephyrus in  ~/epics/R7.0.8/base [11:48]
$ caput i64 2147483999
Old : i64                            2.14748e+09
New : i64                            2.14748e+09
```
-------------
### 64bit Integer Output Record (int64out)
类似`longout`, 但`R7.0.8`依然不支持`OOPT`, 也许将来会加上.

-------------
### Multi-Bit Binary Input Record (mbbi)
读取32位无符号数或者字符串, 然后匹配对应的值.

因为是从hardware中读取值, 所以类似`ai`, 也要raw value和value的区分, raw value经过转换然后放入value. 具体转换规则要依据device support而定.

对于16个匹配项, 每个都有一个值, 一个string, 一个severity.
1-16的前缀分别为: `ZR, ON, TW, TH, FR, FV, SX, SV, EI, NI, TE, EL, TV, TT, FT, FF`. 分别加上`VL`, `ST`, `SV`就可以.

使用的field:
- `NOBT`: Number of Bits, 由此得到`MASK`, `((1 << NOBT) - 1)`
- `SHFT`: 移位, 对于raw value是右移位, **也会把`MASK`左移`SHFT`位**
- `RVAL`: 原始值&`MASK`的值
- `UNSV`: Unknown State Severity, 无匹配时的severity
- `COSV`: Change of State Svr

如下例, mbbi的值为`VAL&0xf0`, 因为设置了`SHFT`, 所以`MASK`移位为`0xf0`. 然后raw value就是原始值与`MASK`与操作的结果, 也就是`0x30. `. 然后把RVAL右移4位后进行匹配, 也就是`0x3`, 与`ONVL`匹配成功.

可以看到同时使用`SHFT`和`NOBT`的逻辑显得令人困惑. 所以感觉应该谨慎使用`SHFT`.
```
record(mbbi, mbbi){
  field(NOBT, "4")
  field(DTYP, "Raw Soft Channel")
  field(INP,  "0x1234")
  field(SHFT, "4")
  field(ZRVL, "0x4")
  field(ZRST, "VAL&0xf")
  field(ZRSV, "MINOR")
  field(ONVL, "0x3")
  field(ONST, "VAL&0xf0")
  field(ONSV, "MINOR")
  field(FFVL, "0x1234")
  field(FFST, "All")
  field(PINI, "RUNNING")
}
```

-------------
### Multi-Bit Binary Output Record (mbbo)
类似于`mbbi`, 输出32位无符号数. 但是多了output相关的一些field, 比如`DOL, OMSL, IVOA, IVOV`

例子: 用`DOL`来决定输出哪个匹配项, 也可以直接`caput mbbo 1`来选择.
```
record(mbbo, mbbo){
  field(DTYP, "Raw Soft Channel")
  field(DOL,  "15")
  field(OMSL, "closed_loop")
  field(PINI, "RUNNING")
  field(OUT,  "li")
  field(ZRVL, "111")
  field(ZRST, "aaa")
  field(ZRSV, "MINOR")
  field(ONVL, "222")
  field(ONST, "bbb")
  field(ONSV, "MINOR")
  field(FFVL, "32768")
  field(FFST, "fff")
}
record(longin, li){}
```

-------------
### Multi-Bit Binary Input Direct Record (mbbiDirect)
`mbbi`是匹配, `mbbiDirect`则是直接映射/操作bit.

把一个32位 **有符号数** 的每一位放到`B0-B1F`中. 当然mbbiDirect主要用于寄存器操作, 也无所谓有无符号.

同样也支持`NOBT`, `MASK`, `SHFT`.
```
record(mbbiDirect, mbbidir){
  #field(NOBT, "4")
  field(DTYP, "Raw Soft Channel")
  field(INP,  "0xaaaa")
  #field(SHFT, "4")
  field(PINI, "RUNNING")
}
```

-------------
### Multi-Bit Binary Output Direct Record (mbboDirect)
从DOL中取数, 然后输出.

从`R7.0.6.1`开始, 当`OMSL`为`closed_loop`时, 禁止直接`caput mbbodir.BO 1`. 需要先修改成`supervisory`. 然后直接写`B0-B1F`这些field. 直接`caput mbbo 0xffff`无效.
```
record(longout, lo){}
record(mbboDirect, mbbodir){
  field(PINI, "YES")
  field(DOL,  "0xaaaa")
  field(OMSL, "closed_loop")
  field(DTYP, "Raw Soft Channel")
  field(OUT,  "lo PP")
}
```

-------------
### Permissive Record (permissive)
**deprecated**

-------------
### Printf Record (printf)
用printf来输出格式化字符串. 可以输出到别的string record, 或者使用base提供的device support来输出到stream. 可用`@stdout, @stderr or @errlog`

> SIZV 不能超过32767, 不然会导致segmentation fault.

```
record(printf, pr){
  field(DTYP, "stdio")
  field(SIZV, "80")
  field(FMT,  "0: %s, 1: %f, 2: %f, 3: %s")
  field(INP0, {const: hello})
  field(INP1, {const: "3.14159"})
  field(INP2, {const: "pi:3.14159"})
  field(INP3, "non-exist-link")
  field(IVLS, "Whoops")
  field(PINI, "YES")
  field(OUT,  "@stdout")
}
```
-------------
### Select Record (sel)
12个输入link.

从`INPA-INPL`中得到数据放到field`A-L`中, 然后依据`SELM`来决定`VAL`.
- `Specified`: 使用`SELN`中的值(0-11), 决定使用哪个input link的值. 也可以使用`NVL`这个link来获得`SELN`. **此处为什么不继续使用`SELL`这个名称而要改为`NVL`呢?**
- `High Signal`: 选择`A-L`中的最大值
- `Low Signal`: 选择`A-L`中的最小值
- `Median Signal`: 选择`A-L`中的中位数

-------------
### Sequence Record (seq)
16个输入输出link, 相当于16个`ao`集成在一起.

从`DOL0-DOLF`得到value放到`DO0-DOF`中, 然后输出到`LNK0-LNKF`. 支持`SHFT`和`OFFS`. 支持delay field, `DLY0-DLYF`, 在取数之前等待一段时间.

如下例, 会输出10到`LNK0`, 11到`LNK1`.
```
record(seq, seq){
  field(SELM, "Mask")
  field(SELN, "3")
  field(SHFT, "0")
  field(OFFS, "0")

  field(DOL0, "10")
  field(DOL1, "11")
  field(DOL2, "12")
  field(DOL3, "13")
  field(DOL4, "14")
  field(LNK0, "calc0")
  field(LNK1, "calc1")
  field(LNK2, "calc2")
  field(LNK3, "calc3")
  field(LNK4, "calc4")
}
```
-------------
### State Record (state)
**deprecated**

-------------
### String Input Record (stringin)
40 characters

-------------
### String Output Record (stringout)
40 characters

-------------
### Long String Input Record (lsi)
65535 characters

`SIZV`: 默认41, 不能超过65535

有bug, 当设置`SIZV`大于32767时, 使用CA读取时会导致ioc segmentation fault.
```
record(lsi, lsi){
  field(SIZV, "32768")
  field(INP,  {const: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa})
  field(PINI, "YES")
}
```
-------------
### Long String Output Record (lso)
65535 characters

同样, SIZV大于32767时候就出问题
```
record(lso, lso){
  field(OMSL, "closed_loop")
  field(DOL,  {const: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb})
  field(SIZV, "32768")
  field(PINI, "YES")
}
```
-------------
### Sub-Array Record (subArray)
对于`subArray` record, 和`waveform`类似, 都有`FTVL, NELM, NORD`这些field.

额外的field包括
- `MALM`: 需要设置为`waveform`的size, 分配`subArray`元素的内存时会用`MALM`的大小
- `INDX`: index, 必须小于`MALM`.

注意它的实现只能获得前`MALM`个元素, 也就是说当`MALM`为1时, index只能为0. 如果要从一个size为1000的waveform中获得最后一个元素, 那`subArray`的大小也必须为1000, 即使它只关心那最后一个元素.

下例中, 当设置`MALM`为5, 那waveform中后五个元素就无法得到了.
```
record(subArray,"subarr") {
  field(INP,  "wf.VAL NPP NMS")
  field(MALM, "5")
  field(NELM, "2")
  field(INDX, "4")
  field(FTVL, "ULONG")
}
record(waveform,"wf") {
  field(PINI, "YES")
  field(INP,  "[3,1,4,1,5,9,2,6]")
  field(NELM, "10")
  field(FTVL, "ULONG")
  field(FLNK, "subarr")
}
```
-------------
### Subroutine Record (sub)

sub有12个input link.

用法如下, 很简单, `INAM`函数用来初始化一些设置, `SNAM`函数会在每次process时调用.
```
record(sub,"$(user):subExample")
{
  field(INAM,"mySubInit")
  field(SNAM,"mySubProcess")
}
```

-------------
### Array Subroutine Record (aSub)
aSub就复杂多了, 有20个input和20个output.

使用的field:
- `LFLG`: Subr. Input Enable, 是否在每次record process时从`SUBL`读取新的值, 默认`IGNORE`, 可选`READ`. *! 这时候选项又不是首字母大写了. (┬┬﹏┬┬)*
- `SUBL`: Subroutine Name Link, input link, 可以切换subroutine
- `EFLG`: Output Event Flag, 是否触发输出事件, 默认为`ON CHANGE`, 可选`NEVER`, `ALWAYS`
- `VAL`: subroutine的返回值, 状态码, 用于判断是否output, 0代表无故障.
- `OVAL`: Old return value
- `BRSV`: Bad Return Severity, 设置subroutine返回值不为0时的severity.
- `INAM`: 初始化时调用
- `SNAM`: process调用
- `INPA-INPU`: input link
- `A-U`: 输入值, 可以为数组
- `FTA-FTU`: 数据类型, 输入
- `NOA-NOU`: Max. elements, 在subroutine中使用`prec->NOT`要注意, `not`是C++的关键字, 所以必须大写来规避.
- `NEA-NEU`: Num. elements
- `OUTA-OUTU`: output link
- `VALA-VALU`: 由subroutine负责修改, 写入到output link
- `OVLA-OVLU`: 旧的`VALA-VALU`值, 用于比较是否变化.
- `FTVA-FTVU`: 数据类型, 输出
- `NOVA-NOVU`: Max. elements, 输出
- `NEVA-NEVU`: Num. elements, in `VALA-VALU`, 输出
- `ONVA-ONVU`: Old Num. elements, in `OVLA-OVLU`, 输出

```
record(aSub,"$(user):aSubExample")
{
  field(INAM,"myAsubInit")
  field(SNAM,"myAsubProcess")
  field(FTA,"DOUBLE")
  field(NOA,"10")
  field(INPA,"$(user):compressExample CPP")
}
```