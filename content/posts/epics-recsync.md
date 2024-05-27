---
title: "epics recsync"
date:  2024-05-27T17:24:25+02:00
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
