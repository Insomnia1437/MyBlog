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
-------------
### Compression Record (compress)
-------------
### Data Fanout Record (dfanout)
-------------
### Event Record (event)
-------------
### Fanout Record (fanout)
-------------
### Histogram Record (histogram)
-------------
### 64bit Integer Input Record (int64in)
-------------
### 64bit Integer Output Record (int64out)
-------------
### Long Input Record (longin)
-------------
### Long Output Record (longout)
-------------
### Long String Input Record (lsi)
-------------
### Long String Output Record (lso)
-------------
### Multi-Bit Binary Input Direct Record (mbbiDirect)
-------------
### Multi-Bit Binary Input Record (mbbi)
-------------
### Multi-Bit Binary Output Direct Record (mbboDirect)
-------------
### Multi-Bit Binary Output Record (mbbo)
-------------
### Permissive Record (permissive)
-------------
### Printf Record (printf)
-------------
### Select Record (sel)
-------------
### Sequence Record (seq)
-------------
### State Record (state)
-------------
### String Input Record (stringin)
-------------
### String Output Record (stringout)
-------------
### Sub-Array Record (subArray)
-------------
### Subroutine Record (sub)
-------------
### Array Subroutine Record (aSub)

