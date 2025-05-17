---
title: "Chm Macos"
date:  2025-05-17T08:14:27+09:00
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
> 许久未更新blog, Hugo的版本更新导致出现一堆配置问题, 稍微花了些时间解决后可以进入主题了

最近需要阅读某些设备的手册, 但这些设备只提供chm格式的文件, 在Windows和Linux平台自然没有问题, 但macOS上如何阅读就稍微需要花点工夫. 检索之下发现了这个项目, 看着挺满足我的需求的.

[https://github.com/rzvncj/xCHM](https://github.com/rzvncj/xCHM) 

那就安装吧, 支持MacPorts, 但是不支持brew, 似乎只能编译安装了.

首先安装依赖
```shell
$ brew install wxwidgets
$ brew install chmlib
# 然后安装
$ ./configure && make
checking whether make sets $(MAKE)... yes
checking for a BSD-compatible install... /usr/bin/install -c
checking for a race-free mkdir -p... mkdir -p
checking for a sed that does not truncate output... /usr/bin/sed
checking whether NLS is requested... yes
checking for msgfmt... /opt/homebrew/bin/msgfmt
checking for gmsgfmt... /opt/homebrew/bin/msgfmt
checking for xgettext... /opt/homebrew/bin/xgettext
checking for msgmerge... /opt/homebrew/bin/msgmerge
checking for gcc... gcc
checking whether the C compiler works... yes
checking for C compiler default output file name... a.out
checking for suffix of executables...
checking whether we are cross compiling... no
checking for suffix of object files... o
checking whether the compiler supports GNU C... yes
checking whether gcc accepts -g... yes
checking for gcc option to enable C11 features... none needed
checking whether gcc understands -c and -o together... yes
checking build system type... aarch64-apple-darwin24.4.0
checking host system type... aarch64-apple-darwin24.4.0
checking for ld used by gcc... /Library/Developer/CommandLineTools/usr/bin/ld
checking if the linker (/Library/Developer/CommandLineTools/usr/bin/ld) is GNU ld... no
checking for shared library run path origin... done
checking how to run the C preprocessor... gcc -E
checking for egrep -e... /usr/bin/grep -E
checking for CFPreferencesCopyAppValue... yes
checking for CFLocaleCopyCurrent... yes
checking for GNU gettext in libc... no
checking for iconv... yes
checking for working iconv... yes
checking how to link with libiconv... -liconv
checking for GNU gettext in libintl... no
checking whether to use NLS... no
checking for wx-config... /opt/homebrew/bin/wx-config
checking for wxWidgets version >= 3.0.0... yes (version 3.2.8)
checking for wxWidgets static library... no
checking whether sleep supports fractional seconds... yes
checking filesystem timestamp resolution... 2
checking whether build environment is sane... yes
checking for gawk... no
checking for mawk... no
checking for nawk... no
checking for awk... awk
checking whether make supports the include directive... yes (GNU style)
checking whether make supports nested variables... yes
checking xargs -n works... yes
checking dependency style of gcc... gcc3
checking for g++... g++
checking whether the compiler supports GNU C++... yes
checking whether g++ accepts -g... yes
checking for g++ option to enable C++11 features... none needed
checking dependency style of g++... gcc3
checking for stdio.h... yes
checking for stdlib.h... yes
checking for string.h... yes
checking for inttypes.h... yes
checking for stdint.h... yes
checking for strings.h... yes
checking for sys/stat.h... yes
checking for sys/types.h... yes
checking for unistd.h... yes
checking for int32_t... yes
checking for int16_t... yes
checking for uint16_t... yes
checking for uint32_t... yes
checking for uint64_t... yes
checking for chm_lib.h... no
configure: error: Can't find the CHMLIB header.
```

呃...缺少头文件, 找一找, 明明在那呢! 看来是路径问题

```shell
$ ls /opt/homebrew/Cellar/chmlib/0.40/include
chm_lib.h lzx.h
```

问了下LLM, 看来需要指定搜索路径, 包含brew安装的程序, 头文件在brew的目录下也的确有一份复制
```shell
$ ls /opt/homebrew/include/chm_lib.h
/opt/homebrew/include/chm_lib.h
```

那就手动添加指定头文件搜索路径
```shell
$ export CFLAGS="-I/opt/homebrew/include"
... # 省略部分输出
checking for sys/types.h... yes
checking for unistd.h... yes
checking for int32_t... yes
checking for int16_t... yes
checking for uint16_t... yes
checking for uint32_t... yes
checking for uint64_t... yes
checking for chm_lib.h... yes
checking for chm_open in -lchm... no
configure: error: Can't find/use -lchm. Please install CHMLIB first.
```

还是不行, 又说缺少链接库, 那就也指定链接库的路径
```shell
$ export LDFLAGS="-L/opt/homebrew/lib"
$ ./configure
checking whether make sets $(MAKE)... yes
checking for a BSD-compatible install... /usr/bin/install -c
checking for a race-free mkdir -p... mkdir -p
checking for a sed that does not truncate output... /usr/bin/sed
checking whether NLS is requested... yes
checking for msgfmt... /opt/homebrew/bin/msgfmt
checking for gmsgfmt... /opt/homebrew/bin/msgfmt
checking for xgettext... /opt/homebrew/bin/xgettext
checking for msgmerge... /opt/homebrew/bin/msgmerge
checking for gcc... gcc
checking whether the C compiler works... yes
checking for C compiler default output file name... a.out
checking for suffix of executables...
checking whether we are cross compiling... no
checking for suffix of object files... o
checking whether the compiler supports GNU C... yes
checking whether gcc accepts -g... yes
checking for gcc option to enable C11 features... none needed
checking whether gcc understands -c and -o together... yes
checking build system type... aarch64-apple-darwin24.4.0
checking host system type... aarch64-apple-darwin24.4.0
checking for ld used by gcc... /Library/Developer/CommandLineTools/usr/bin/ld
checking if the linker (/Library/Developer/CommandLineTools/usr/bin/ld) is GNU ld... no
checking for shared library run path origin... done
checking how to run the C preprocessor... gcc -E
checking for egrep -e... /usr/bin/grep -E
checking for CFPreferencesCopyAppValue... yes
checking for CFLocaleCopyCurrent... yes
checking for GNU gettext in libc... no
checking for iconv... yes
checking for working iconv... yes
checking how to link with libiconv... -liconv
checking for GNU gettext in libintl... yes
checking whether to use NLS... yes
checking where the gettext function comes from... external libintl
checking how to link with libintl... -lintl -Wl,-framework -Wl,CoreFoundation
checking for wx-config... /opt/homebrew/bin/wx-config
checking for wxWidgets version >= 3.0.0... yes (version 3.2.8)
checking for wxWidgets static library... no
checking whether sleep supports fractional seconds... yes
checking filesystem timestamp resolution... 2
checking whether build environment is sane... yes
checking for gawk... no
checking for mawk... no
checking for nawk... no
checking for awk... awk
checking whether make supports the include directive... yes (GNU style)
checking whether make supports nested variables... yes
checking xargs -n works... yes
checking dependency style of gcc... gcc3
checking for g++... g++
checking whether the compiler supports GNU C++... yes
checking whether g++ accepts -g... yes
checking for g++ option to enable C++11 features... none needed
checking dependency style of g++... gcc3
checking for stdio.h... yes
checking for stdlib.h... yes
checking for string.h... yes
checking for inttypes.h... yes
checking for stdint.h... yes
checking for strings.h... yes
checking for sys/stat.h... yes
checking for sys/types.h... yes
checking for unistd.h... yes
checking for int32_t... yes
checking for int16_t... yes
checking for uint16_t... yes
checking for uint32_t... yes
checking for uint64_t... yes
checking for chm_lib.h... yes
checking for chm_open in -lchm... yes
checking that generated files are newer than configure... done
configure: creating ./config.status
config.status: creating Makefile
config.status: creating src/Makefile
config.status: creating art/Makefile
config.status: creating po/Makefile.in
config.status: creating m4/Makefile
config.status: creating mac/Makefile
config.status: creating data/Makefile
config.status: creating man/Makefile
config.status: creating config.h
config.status: executing po-directories commands
config.status: creating po/POTFILES
config.status: creating po/Makefile
config.status: executing depfiles commands
```
没有报错了, 编译!
```shell
$ make
```
一堆warning == 无事发生!
安装+运行
```shell
$ make install
$ /usr/local/bin/xchm 
```
运行效果:

![xchm](/images/xchm-macos.png)
