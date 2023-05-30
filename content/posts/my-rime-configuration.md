---
title: "My Rime configuration"
date:  2023-04-18T22:15:20+09:00
# weight: 1
# aliases: ["/first"]
tags: ["Software", "Rime", "Chinese", "Japanese"]
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

## 在Rime之前

> 人生中的某个阶段执迷于效率软件在某种意义上来看还是要好于不尝试效率软件的.

对于效率软件, 我一向是秉持够用即可的心态, 但最近也陆续尝试了不少或老或新的`新`玩意. 其中一个就是Rime. 原本无论Windows还是macOS, 我一直使用系统自带的输入法, 一方面是平常大多使用英文输入, 对于中文词库或输入联想等功能无甚要求.
但往往变化孕育于更高维度, 对于输入法来讲, 就是键盘配列. 由于Windows的中文输入法无法选择键盘配列 (对比下日语输入法就可选日语配列或英语配列, 因此不是哪怕一丁点的技术问题), 导致日语配列键盘下输入中文会导致符号错位, 相当恼人.
于是乎, 为了解决这个麻烦, 接触了Rime, 但最终发现Rime依然使用了操作系统回报的键盘配列, 对这个问题无能为力. 但过程中也成功自我欺骗, 使自己相信自己有更多输入法方面的需求, 倒也不算坏事 (继续自我合理化😊). 大概也算是生活中诸多"错位动机引致不差结果"的一范例.

---

BTW, Windows下可以在注册表中修改输中文入法使用日语键盘配列的方式如下
```
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Keyboard Layouts\00000804]

Change "Layout File"="KBDUS.DLL"
To     "Layout File"="KBD106.dll"
```

## Rime

Rime是输入法框架, 提供了从键盘按下字符到分词到翻译到筛选到输出一系列组合功能. 主要服务于`CJKV`人群 (用于输入中文的漢字、日文的漢字及假名、韓文的漢字及諺文和越南文的儒字和喃字), 用户自定不同的输入方案, 由Rime切换与使用. 关于Rime的介绍到此为止, 官方[wiki](https://github.com/rime/home/wiki)中包含了更多作者的思考与哲学.

- 【中州韻】 ibus-rime → Linux, `~/.config/ibus/rime/`, `~/.local/share/fcitx5/rime/`
- 【小狼毫】 Weasel → Windows, `%APPDATA%\Rime`
- 【鼠鬚管】 Squirrel → Mac OS X, `~/Library/Rime/`

很早之前就在Manjaro中使用过, 但也仅限于在Linux内使用. 跨平台在我看来是比隐私保护更重要的功能. 而跨平台的用户自管理的配置同步就更加难得可贵.

Rime有点像Zsh, 刚推出的时候都非常灵活且有诸多配置项, 但灵活意味着自定义与门槛, 就像`Oh My Zsh`的推出改变了Zsh的生态, 目前倒是有些项目取名为`Oh My Rime`, 但其实现似乎还难以于`Oh My Zsh`的完善性相提并论. `CJKV`语种人群本身就互有嫌隙, 开源精神在东亚稍显水土不服, 跨国合作非常不易, 而即便是汉语使用者内部, 对Rime作者身份与繁体字使用习惯有微词的也偶尔可见. Rime的发展大概会一直如此不温不火下去.

## Personal Rime

[**My Rime configuration**](https://github.com/Insomnia1437/rime)

只使用全拼, 个人而言, 英文输入与多设备场景更多的情况下, 去记忆不同的双拼或五笔码码显得得不偿失.

对于汉字, 只要输入字母序列, 就可以按照不同的分词方式, 然后按照字典去查词. 基本按个人习惯配置一下输入法切换快捷键就可以快速使用. 其他的一些诸如快捷输入日期, 中英混输, 模糊音等配置也就是在努力接近普通的商用中文输入法.

### Japanese input

但日语输入的麻烦在于日语动词变形与鼻音. kani可以是日语的**カニ/かに**🦀, 也可以是**簡易/かんい**, 对于鼻音可以使用分隔符, 一方面也可以在候选项中同时提示(如大多数商用日语输入法所做的那样).

而对于动词活用, 如**買う**需要输入`kau`,  而过去时态的**買った**则需要输入`katta`, 目前没发现有Rime配置能处理日语的动词变形, 趁着还没开始搜索答案, 我也在猜想, 商用日语输入法如ATOK是如何处理的呢. 对每个动词的不同活用形都生成多个码表自然是简单易懂, 但未免费事费力; 如果是使用类似Rime的Lua脚本实现, 那程序的替换逻辑改如何实现呢?

自然, 谜题的揭晓还需要我本人按耐得住性子, 唔...此事应当不难. ╮(￣▽￣")╭

![日语动词活用示例](/images/japanese-verb-conjugation.png)
图片来源于 https://en.wikipedia.org/wiki/Japanese_verb_conjugation#Causative
