---
title: "Personal Knowledge Management"
date:  2023-05-30T09:36:59+09:00
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

## Logseq与管理

本文大致讲讲当前我的笔记软件与笔记管理方案. 本地使用Logseq记录笔记与日记, Git进行版本管理, 保存后自动push到个人服务器上的Gitea, 个人服务器上同时运行duplicati, 每日将repository强加密后增量备份存在OneDrive中. 目前挺满意.

### Logseq

对于笔记软件, 有太多形而上与形而下的讨论, 我无力参与太多. 但为何从Notion转到Logseq倒可以讲讲.

1. 一是Notion的性能问题. 在page层次较深与page中有较多block时, 能明显感受到中文输入条件下切换页面的卡顿. 或许是对CJKV的支持不够, 或许是数据库索引方面的问题, 但无论如何还是令人困扰.

2. 二是云存储的隐私问题. Notion中的图片会存放在AWS上, 不适合存放一些偏私密的内容.


3. 三是Notion的笔记粒度问题. 以页面为基础笔记单元在整理树形结构笔记时(如自然按章节间断的课程笔记)很合适, 但面对非树形结构性内容(如日记与code snippet)则显得力不从心. 在写笔记前必须思考将之归入何大类下的何小类显得繁琐, 而往往此种分类还会面临日后调整的需要.


为何是Logseq? 试用了一段时间, 它让我捡回了中断数年的写日记习惯. 于是就一直用着了.  而Logseq中将笔记的粒度控制在block, 在某些场景下是比粒度在page更好的选择. 至于它的whiteboard, flashcard功能, 我选择关闭. 以及block的嵌入引用等华丽功能也甚少使用.

### Logseq与Git

Logseq支持使用Git进行版本管理. 但Git在Windows下自然需要自己安装.

然后添加hooks
文件`pre-commit`会在commit之前执行, 先pull.
```
#!/bin/sh
#
#
# Pull before committing
# Credential handling options:
#  - hardcode credentials in URL
#  - use ssh with key auth
#  - https://git-scm.com/docs/git-credential-store
#  - git credential helper on windows

# Redirect output to stderr, uncomment for more output for debugging
# exec 1>&2

output=$(git pull --no-rebase)

# Handle non error output as otherwise it gets shown with any exit code by logseq
if [ "$output" = "Already up to date." ]; then
    # no output
    :
else
    # probably error print it to screen
    echo "${output}"
fi

git add -A
```

文件`post-commit`会在本地commit之后执行, push to remote
```
#!/bin/sh

git push origin main
```

### Gitea

相当多人使用GitHub的私有仓库作为Logseq的备份仓库, 但还是个人服务器更隐蔽一些.

对于个人Git服务器, 我选择使用Gitea. Gitea的安装与配置见官方文档.

Linux/macOS/Windows下都需要配置`.ssh/config`文件

下例为GitHub的配置, 如果自建则可能需要修改HostName, Port与IdentityFile项.
```
Host github.com
	User git
#	Port 22
	HostName github.com
	PreferredAuthentications publickey
	IdentitiesOnly yes
	IdentityFile ~/.ssh/id_ed25519_github
	ForwardX11 no
```

### Duplicati

Duplicati的安装与配置见官方文档.

由于是在个人服务器上运行, 因此duplicati必须配置密码. 之后就可以在网页中设置需要备份的目录, OneDrive的连接密码与目标目录.
