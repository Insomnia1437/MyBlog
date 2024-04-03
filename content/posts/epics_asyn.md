---
title: "EPICS aysn"
date:  2024-04-01T17:11:20+02:00
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
disableHLJS: false # to disable highlightjs
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
# @Software: VS Code/MacDown
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

# asyn

> 阅读本文需要对epics中的概念比较熟悉. 建议阅读`EPICS AppDevGuide`. 也许是历史原因, asyn文档中的概念很混乱. 很大程度上是源于一词多义. 比如driver, support, implement等词, 既可以表达asyn中的专用概念, 也会作寻常用法使用. 比如对driver一词的使用. 有时候是epics driver support, 有时候又是port driver, 有时候又写成device driver. 而port driver, 一般来说指一系列hardware driver, 但又容易与为了减少代码编写量而引入的`asynPortDriver`这一个c++ class的名字混淆. 因此我决定撰写此文便于将来的自己理解.

## aysn的出现

20世纪90年代中期对于epics驱动开发有这么几个问题

1. 每一个新的device都需要编写多个record类型的device support, 比如ai, ao, mbbi, mbbo等
2. 当多个record访问一个device时, 需要处理同步和互斥的问题
3. 支持不同访问协议的同一设备需要重复编写驱动.

于是Marty Kraimer提出了asyn的想法, 和Mark Rivers一起开发了最初版本的asyn.

asyn如何解决这些问题?
1. 使用接口, 在device support和driver之间定义了一套通用的接口. 比如connect, read, write, report等等. 这些函数需要由底层驱动实现.
2. 引入了port的概念. 一个port对应一条通信路径, driver会register port, device support会连接到一个port
3. asynManager负责port的管理, 同时处理同步与互斥
4. 提供了一套标准的epics device support, 涵盖了常用的大多数record type. 这样引入新设备时候只需要专注于实现与设备通信的接口.

## asyn 中的一些概念
- user, 在epics环境下, 就是指某个record type的device support.
- port driver, 也就是hardware driver, 表示实际与hardware交互的代码. port driver必须实现asynCommon, 然后实现多个communication interface, 比如asynOctet, asynInt32.

在asyn中, user指使用asyn port的一端，一般来说就是epics record. 而port driver则是与各式各样hardware交互的底层驱动, 在port driver中，需要实现一些接口, 比如标准接口`asynCommon`, 命令读写接口`asynOctet`。但如果协议比较复杂, 可以继续添加中间层, 减少port driver的编写代码量. 

例如gpib, 就添加了`asynGpib`和`asynGpibPort`两个接口, 前者由epics record support调用, 然后会调用`asynGpibPort`中的函数, 而`asynGpibPort`中的函数由port driver实现. 这样的话就可以把一些不涉及具体GPIB硬件的函数移到asynGpib中.
## asyn 流程
asynManager作为中间层. 服务于epics device support和hardware driver.

### 对于epics device support
- 在epics record init过程中会调用`pasynManager->createAsynUser(processCallback, 0)`, 获取一个没有绑定port的asynUser
- 解析record的INP或者OUT field 指定的port name和port addr
- 调用`connectDevice`来让asynUser连接上这个port. 因为asynManager知道所有的port driver, 所以它依靠port name就能顺利把port的信息放到asynUser里. 
- 这时候asynUser就可以调用`pasynInterface = pasynManager->findInterface(pasynUser, asynInt32Type, 1)` 得到想调用的interface. 比如此时的`pasynInterface->pinterface` 就是一个`asynInt32`的接口, 使用这个接口才可以与hardware通信. 
- 对于asynchronous的设备, 肯定不能直接读取, 于是就在record被process(例如通过`caput`)的时候, 调用`pasynManager->queueRequest(pasynUser, 0, 0)`, 把请求放到port thread里
- asynManager调用Callback函数`processCallback`(`createAsynUser`时候注册的). 
- 在Callback函数中会调用`pasynInterface->pinterface->read`来读写hardware.


### 对于port driver
- 实现与hardware交互的接口, 比如connect serial的时候就要使用OS API连接到tty.
- 注册一个iocsh函数, 方便启动ioc时候用户设置port的名字和各种参数(比如IP或serial的baud rate)
- 这个iocsh函数会调用`registerPort`, 在asynManager中, 会新生成一个线程, 用来处理这个port. 
- 调用`registerInterface`注册实现的接口, 其中asynCommon接口必须提供. 在port thread中会调用asynCommon提供的`pasynCommon->connect(drvPvt,pasynUser)`方法连接hardware. 常用的还有`aysnOctet`接口中的read, write函数, `asynInt32`中的read, write函数

## asyn interface
interface, 在asyn中更强调其在面向对象编程(OO)中的特点, 即抽象方法. 在c语言中需要使用函数指针达成这一目标(在OO环境中需谨慎使用"实现"一词). 在asyn的不同层级中互相调用这些抽象方法, 有些抽象方法由asyn中的一些component(比如asynManager实现了asynManager接口)实现, 而另一些抽象方法需要最终实现为与hardware交互的代码. 比如connect, read, write等方法.
### 常用接口
- asynCommon, 包含三个函数, report, connect, disconnect
- asynTrace, debug用, 不用管细节.
- asynLockPortNotify, 用于addrChangeDriver, 不过似乎有bug(R4-44-2)
- asynDrvUser, 用于传递参数给port driver. 似乎也是最近引入的. 常用用法: 在record的INP field中填写一个参数, 然后这个参数会被传递给port driver, port driver决定在收到这个参数后如何处理返回值. 这个接口主要用在`asynPortDriver`中, 为了使用这个接口还专门开发了两个class: paramList class和paramVal class (其实就是用了一个`std::vector`, 记录了参数名和其index)用于处理这个参数.
- asynXXX, 一系列接口, 比如asynOctet, asynInt32, asynFloat64Array, asynOption 等等. port driver需要根据hardware的情况选择性实现一部分.

## interrupt 处理
支持interrupt的接口会有`registerInterruptUser`函数, 例如

```c
typedef struct asynInt32 {
    asynStatus (*write)(void *drvPvt, asynUser *pasynUser, epicsInt32 value);
    asynStatus (*read)(void *drvPvt, asynUser *pasynUser, epicsInt32 *value);
    asynStatus (*getBounds)(void *drvPvt, asynUser *pasynUser,
                           epicsInt32 *low, epicsInt32 *high);
    asynStatus (*registerInterruptUser)(void *drvPvt,asynUser *pasynUser,
                           interruptCallbackInt32 callback, void *userPvt,
                           void **registrarPvt);
    asynStatus (*cancelInterruptUser)(void *drvPvt, asynUser *pasynUser,
                           void *registrarPvt);
} asynInt32;
```

### 对于asynManager
提供了`registerInterruptSource `函数, 产生一个与interface type相关的`interruptBase`的变量. 里面有一个链表`callbackList`.

提供了`addInterruptUser`函数, 把record希望注册的Callback函数放到`interruptBase `中的链表中.

提供了`interruptStart`函数, 从`interruptBase`中取出链表中的Callback元素, 然后一个个执行.
### 对于epics device support
当某个record希望响应interrupt时, 需要调用`addInterruptUser `.

record init时候调用`registerInterruptUser`, 虽然函数是个接口, 但各个asynXXXBase已经实现了, 可以在port driver中覆盖, 但似乎都使用默认值即可. 不像read write函数一样需要被覆盖. `registerInterruptUser`函数在asynXXXBase中实现, 该版本会调用`pasynManager->addInterruptUser(pasynUser,pinterruptNode)`, 其中interruptNode内部包含了一个Callback函数. 把这个Callback函数加入到一个链表中. 这样interrupt发生时, 就可以用Callback来process自己.
### 对于port driver
注册interface时候, 也注册了interface的`registerInterruptUser`函数. 

调用`pasynManager->registerInterruptSource(portName,&pdrvPvt->asynInt32, &pdrvPvt->asynInt32Pvt);` 其实就是由asynManager生成一个`interruptBase`类型的变量, 然后分配给`asynInt32Pvt`. 

当interrupt发生时, 会依次调用这个链表中所有的Callback函数. 这个过程一般发生在isr里, 而isr自然是由port driver实现. 在isr里调用`interruptStart`函数, 得到这个链表, 然后调用Callback去process这个record.
## asynXXXBase
关于`asynXXXBase.c`, 其实就是简化了注册`asynXXX`接口过程, 定义了一些默认的读写方法, 调用`registerInterface`. 在port driver里, 可以这样写

```c
    pasynInt32->write = int32Write;
    pasynInt32->read = int32Read;
    pasynInt32->getBounds = int32GetBounds;
    pdrvPvt->asynInt32.interfaceType = asynInt32Type;
    pdrvPvt->asynInt32.pinterface  = pasynInt32;
    pdrvPvt->asynInt32.drvPvt = pdrvPvt;
    status = pasynInt32Base->initialize(pdrvPvt->portName, &pdrvPvt->asynInt32);
```

而`asynXXXArrayBase.h`更进一步, 定义了一个超大macro.
但由于port driver里还是需要重新实现read write函数, 所以实际上没省略多少代码量. 然后又搞出了一个`asynStandardInterfacesBase`, 由它实现实际的`registerInterface`, 这部分已经集成在`asynPortDriver`里.

未完待续.
## some evil struct

```c
typedef struct interruptNode{
    ELLNODE node;
    void    *drvPvt;
}interruptNode;
typedef struct interruptBase {
    ELLLIST      callbackList;
    ELLLIST      addRemoveList;
    BOOL         callbackActive;
    BOOL         listModified;
    port         *pport;
    asynInterface *pasynInterface;
}interruptBase;

typedef struct interruptNodePvt {
    ELLNODE  addRemoveNode;
    BOOL     isOnList;
    BOOL     isOnAddRemoveList;
    epicsEventId  callbackDone;
    interruptBase *pinterruptBase;
    interruptNode nodePublic;
}interruptNodePvt;
```





