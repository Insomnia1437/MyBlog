---
title: "Dotfiles"
date:  2024-04-01T12:10:12+02:00
# weight: 1
# aliases: ["/first"]
tags: ["Chinese", "shell"]
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
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

# dotfiles管理

什么是dotfiles? 参见[https://wiki.archlinux.org/title/Dotfiles](https://wiki.archlinux.org/title/Dotfiles).
很多配置文件会以dot开头, 这些文件被称为dotfile. 如果我们经常在多台机器上工作, 这些机器往往有不同的硬件和操作系统, 我们会希望在不同的机器上使用同样的配置, 例如使用同样的shell快捷键, terminal有同样的UI, 因此如何在不同的机器之间同步与安装dotfiles的需求就应运而生.

## 常用的dotfiles

- shell, `.zshrc`, `.tcshrc`, `.bashrc`等
- Editor, `.vimrc`等
- WM/DE, `i3`, `sway`, `awesome`等
- Multipler, `.tmux.conf` or `.screenrc`
- Terminal, `alacritty.toml` or `kitty`
- git, `.gitconfig`

dotfiles需要放到指定的位置才会生效, 最常用的是`${HOME}`目录, 但如果软件依据[XDG标准](https://wiki.archlinux.org/title/XDG_Base_Directory)来读取自己的配置文件, 也可以放到`XDG_CONFIG_HOME`目录, 默认为`$HOME/.config`目录.

## 需求

- 可以选择某个机器安装哪些dotfiles. 例如一些机器上只需要安装`.zshrc`, 而有些只能使用`tcsh`和`bash`.
- 安装完dotfile后可以执行脚本, 比如安装完`.zshrc`会自动安装zsh插件和theme
- 一些环境需要设置proxy server
- 一些配置依赖于软件版本, 比如tmux旧版本不支持一些配置, git新版本引入了某配置, 需要根据安装机器的软件环境来自动决定
- 可以自动判断操作系统的架构(例如darwin-x64, linux-arm), 然后设置对应的`EPICS_HOST_ARCH`

## dotdrop
前段中的arch wiki页面中列出了一些dotfiles管理软件, 我个人使用[dotdrop](https://github.com/deadc0de6/dotdrop). 官方文档地址[dotdrop.readthedocs.io](https://dotdrop.readthedocs.io/en/latest/)
dotdrop使用python编写, 目前版本也支持使用apt brew 和pip直接安装. 但我依然沿用早期的git submodule方式使用. 下面的例子摘自官方文档, 我个人的使用方式参见[my dotfiles](https://github.com/Insomnia1437/dotfiles), 因为不希望自动更新submodule, 所以我对启动程序做了一些微小修改.

### 安装

```shell
## create the repository
$ mkdir dotfiles; cd dotfiles
$ git init

## install dotdrop as a submodule
$ git submodule add https://github.com/deadc0de6/dotdrop.git
$ pip3 install --user -r dotdrop/requirements.txt
$ ./dotdrop/bootstrap.sh

## use dotdrop
$ ./dotdrop.sh --help
```

### 基础使用
dotdrop使用`config.yaml`作为默认配置文件, 不过鉴于YAML的劣势, 将来应该会顺应潮流升级为TOML格式.

dotdrop的基本理念把dotfiles放到git repo中进行统一管理, 在`config.yaml`文件中指定dotfiles的git repo内的路径和系统安装路径, 其中`src`, 就是git repo中的路径(可以在`config.yaml`修改), 然后`dst`是dotfiles的安装路径. 运行dotdrop时, 就会读取git repo中的dotfiles, 然后做一些修改, 之后安装到系统某个路径. 安装路径应该依据软件来选择, 例如vim配置文件需要放到`~/.vimrc`.

```yaml
config:
  backup: true
  banner: true
  create: true
  dotpath: dotfiles
dotfiles:
  d_polybar:
    dst: ~/.config/polybar
    src: config/polybar
  f_vimrc:
    dst: ~/.vimrc
    src: vimrc
  f_xinitrc:
    dst: ~/.xinitrc
    src: xinitrc
profiles:
  home:
    dotfiles:
    - f_vimrc
    - f_xinitrc
    - d_polybar
```

另一个概念是profile, 每一个profile中包含了多个dotfiles, 基本来说是以host来区分, 即某个host上使用的bash和vim, 那这个profile就需要包含bash和vim这两个dotfiles. 当然也可以不以hostname来区分, 如果多个host可共用一套配置, 那就可以用别的命名, 比如`vps`等.
```shell
$ dotdrop profiles #列出配置文件中定义的profiles, 比如profileA profileB
$ dotdrop install -p profileA # 安装指定的profile
$ dotdrop compare -p profileA # 对比profileA和已安装的dotfiles
$ dotdrop files -p profileA # 列出profileA中的dotfiles
```
它也支持直接导入dotfile, 可以使用`dotdrop import`命令, 但我不使用.

还有一些概念包括variables和actions.
variables就是一些变量, 在`config.yaml`中设置之后, 可以在各个dotfile中使用, 会被profile中的局部同名variable覆盖.
actions就是一段shell程序, 会被Python调用执行, 比如可以安装某些依赖. actions可以设置执行的时机(安装dotfiles之前或之后). 可以设置某些actions为`default_actions`, 这样每安装一个dotfile就会执行一次(用于log). 不过更常用的是在某个dotfile中使用某个action, 比如安装`.vimrc`时, 执行action安装Vim插件.

### 配置文件

dotdrop的`config.yaml`文件有多个层级(官方称为block), 每个层级都有自己的配置项.
- config, 一些全局配置项, 比如git repo中的dotfiles路径, 在安装时是否创建backup文件等
- dotfiles, 核心, 定义dotfiles, 设置安装后的文件权限等
- profiles, 可以include其他profile, 可以设定局部variables, 设定要执行的actions
- actions, 定义action, 执行一段程序. 通常是单行命令, 也可以调用某个shell脚本文件.
- variables, 定义variable
- dynvariables, 也是定义variable, 不过它的值来源于一段shell程序的输出结果, 方便我们动态判断运行环境, 比如确定git的版本.
- uservariables, 也是定义variable, 不过会在要求用户输入来确定值. 方便处理一些敏感信息
- transformations, 官方给的用例有两种, 一种是处理需要压缩和解压缩的文件, 另一种是用pgp加密和解密文件. 这样就可以把`.ssh`之类的敏感文件加密后管理起来.

具体使用参考官方文档.

### 高级用法
高级用法依赖于`jinja2`, 可以在dotfiles中使用template, 由dotdrop进行替换, template中可以使用variable, 也可以使用一些条件逻辑. 常用使用方式如下. 一看就会.

#### example1
判断profile的名字, 如果是wsl, 那就把`~/.local/bin`加入到`PATH`中
```jinja2
{%@@ if profile == "wsl" @@%}
export  PATH="{{@@ HOME @@}}/.local/bin:${PATH}"
{%@@ endif @@%}
```
#### example2
在`config.yaml`中有两个variable: `USE_PROXY` 和 `PROXY`, 首先判断 `USE_PROXY` 的值, 如果为`YES`, 那就添加两个alias, 可以设置proxy server, proxy server的IP也是通过template解析出来的. 一般是首先全局定义`USE_PROXY`为`NO`, 然后在某个profile中覆盖为`YES`.
```jinja2
{%@@ if USE_PROXY == "YES" @@%}
alias setproxy="export http_proxy=http://{{@@ PROXY @@}}; export https_proxy=https://{{@@ PROXY @@}}"
alias noproxy="unset http_proxy; unset https_proxy"
{%@@ endif @@%}
```
#### example3
判断git的版本, 如果大于2.13, 就在`.gitconfig`中添加一项配置. 这样的话在一些git版本为1.8的旧机器上, 这个额外的配置就不会产生.
```jinja2
{%@@ if GIT_VERSION >= "2.13" @@%}
# The contents of this file are included only for GitLab.com URLs
[includeIf "hasconfig:remote.*.url:git@**gitlab**:**/**"]
        # Edit this line to point to your alternative configuration file
        path = ~/.gitconfig-gitlab
{%@@ endif @@%}
```
其中`GIT_VERSION`是一个dynvariable, 在`config.yaml`中定义:
```yaml
dynvariables:
  GIT_VERSION: git --version | cut -f 3 -d " "
```