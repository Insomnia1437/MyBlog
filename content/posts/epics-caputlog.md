---
title: "epics caputlog"
date:  2024-05-27T17:23:15+02:00
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
