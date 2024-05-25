---
title: "epics access security"
date:  2024-05-08T10:52:18+02:00
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
## 关键词

- ASL: Access Security Level.
- ASG: Access Security Group
- UAG: User Access Group
- HAG: Host Access Group

每个record field都有ASL, 在dbd文件中定义, 可以是`ASL0`或者`ASL1`. 通常`VAL` field为`ASL0`, 其它为`ASL1`. `ASL1`的优先级高于`ASL0`

每个record都有一个ASG field, 默认为`default`.

## as file语法

- as file中定义一些用户组, 主机组, 然后一些规则组.
- record在它自己的ASG field中指定要使用的规则组.
- 规则组内包含了若干RULE.
- 每个RULE中可以设置用户组和主机组是否可读/可写.
- 每个RULE可以设置ASL
- 规则组可以依据某个pv的值来生效. 通过INP来指定pv, CALC来判断pv值.

```epics
UAG(<name>) { <user> [, <user> ...] }
...
HAG(<name>) { <host> [, <host> ...] }
...
ASG(<name>) {
  [INP<index>(<pvname>) ...]
  RULE(<level>,NONE |READ|WRITE [,NOTRAPWRITE | TRAPWRITE] ) {
    [UAG(<name> [,<name> ...])]
    [HAG(<name> [,<name> ...])]
    CALC(<calculation>)
  }
...
}
...
```

## 启用
as file 中可以使用macro.
```shell
asSetFilename("/full/path/to/accessSecurityFile")
asSetSubstitutions("var1=sub1,var2=sub2,...")
ascheck -S "xxx=yyy,..." < "filename"
```

可以在运行中重载as, 但不推荐. 具体方法为使用subroutine record, 指定`asSubInit`和`asSubProcess`函数.

## 示例
使用`softIoc st.cmd`运行ioc, 无法通过caput修改`demo:limit`.
```shell
$ caput demo:limit 5
Old : demo:limit                     10
Error from put operation: Write access denied
```
### `st.cmd`
```shell
asSetFilename("$(PWD)/secure.acf")
asSetSubstitutions("S=demo")
dbLoadRecords("solution.db", "S=demo")

# var("asCheckClientIP",1)
iocInit
```
### `slotion.db`
```epics
record(calc, "$(S):ramp")
{
  field(CALC, "(A<B)?(A+C):0")
  field(INPA, "$(S):ramp")
  field(SCAN, "1 second")
  field(INPB, "$(S):limit")
  field(INPC, "$(S):step")
}

record(ai, "$(S):limit")
{
  field(INP, "10")
  # ACC SECURITY
  field(ASG, "EXPERT")
}

record(ai, "$(S):step")
{
  field(INP, "1")
}

# ACC SECURITY
record(bo, "$(S):accessState") {
  field(DESC, "Used by ASG to open access")
  field(ZNAM, "Expert-Only")
  field(ONAM, "Anybody")
}
```

### `secure.acf`
```acf
# list of users
UAG(normUsers) {training}
UAG(expertUsers) {expert}

HAG(normHosts) {training, training.epics, training-VirtualBox}

ASG(DEFAULT) {
    RULE(1,READ)
    RULE(1,WRITE) {
        UAG(normUsers)
        HAG(normHosts)
    }
}

ASG(EXPERT) {
    INPA($(S):accessState)

    RULE(1,READ)

    RULE(1,WRITE) {
        UAG(expertUsers)
        HAG(normHosts)
    }

    RULE(1,WRITE) {
        UAG(normUsers)
        HAG(normHosts)
        CALC("A=1")
    }
}

```
## IP
从 `R7.0.3.1`开始, 可以使用`asCheckClientIP` variable来检查`HAG`中指定的host的真实IP. 这样可以防止有人伪装自己的hostname来绕开AS. 如下所示, HAG中的host变为`HAG(normHosts) {unresolved:training,unresolved:training.epics,127.0.1.1}`. 无法解析的hostname会带有`unresolved`字样.

```shell
epics> asdbdump
UAG(expertUsers) {expert}
UAG(normUsers) {training}
HAG(normHosts) {unresolved:training,unresolved:training.epics,127.0.1.1}
ASG(DEFAULT) {
	RULE(1,READ,NOTRAPWRITE)
	RULE(1,WRITE,NOTRAPWRITE) {
		UAG(normUsers)
		HAG(normHosts)
	}
	MEMBERLIST
		<null> Record:demo:accessState
		<null> Record:demo:step
		<null> Record:demo:ramp
}
ASG(EXPERT) {
	INPA(demo:accessState) INVALID value=0.000000
	RULE(1,READ,NOTRAPWRITE)
	RULE(1,WRITE,NOTRAPWRITE) {
		UAG(expertUsers)
		HAG(normHosts)
	}
	RULE(1,WRITE,NOTRAPWRITE) {
		UAG(normUsers)
		HAG(normHosts)
		CALC("A=1") result=FALSE
	}
	MEMBERLIST
		EXPERT Record:demo:limit
}
epics>
```

## tips

```acf
ASG(private) {
    RULE(1, NONE)
}
ASG(DEFAULT) {
   RULE(1,READ)
   RULE(1,WRITE,TRAPWRITE)
}
```