---
title: "epics sequencer"
date:  2024-05-10T16:13:17+02:00
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

> See [epics examples](/posts/epics-examples)

## 介绍
自从去年夏天[bessy](http://www-csr.bessy.de/)被网络攻击之后, sequencer的网站 (http://www-csr.bessy.de/control/SoftDist/sequencer/) 就打不开了. Ralph建立了一个镜像仓库, 可以从此处下载源码. (http://www-csr.bessy.de/control/SoftDist/sequencer/)

我个人没有用过sequencer, 简单逻辑我习惯通过db完成, 稍微复杂一点的用subroutine调用c函数.

但它的确适合用于诸如磁铁初始化的过程. 优点在于区分了Control flow和data flow.
## 安装
sequencer需要安装re2c.

在ioc中加入sequencer的方式
```makefile
# Build sncExample into exampleSupport
sncExample_SNCFLAGS += +r
example_DBD += sncExample.dbd
# A .stt sequence program is *not* pre-processed:
exampleSupport_SRCS += sncExample.stt
exampleSupport_LIBS += seq pv
example_LIBS += seq pv
```

## 用法
SNL 代表 State Notation Language, sequencer就是用来定义状态机. 首先判断当前状态, 然后依据条件判断是否进入另一个状态.

```c
program sncExample
double v;
assign v to "{user}:aiExample";
monitor v;

ss ss1 {
    state init {
        when (delay(10)) {
            printf("sncExample: Startup delay over\n");
        } state low
    }
    state low {
        when (v > 5.0) {
            printf("sncExample: Changing to high\n");
        } state high
    }
    state high {
        when (v <= 5.0) {
            printf("sncExample: Changing to low\n");
        } state low
    }
}
```