---
title: "Japanese Romaji"
date:  2023-05-14T11:59:15+09:00
# weight: 1
# aliases: ["/first"]
tags: ["Chinese", "Rime", "Japanese"]
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
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

## 日语罗马字化与计算机输入

> 上文[My Rime configuration](/posts/my-rime-configuration)中提到了日语输入的两个小问题, 第一个是拨音的输入, 第二个是如何处理动词变形. 在尝试解决的过程中发现了现行日语输入的混乱之处...

### ヘポン式ローマ字 and 訓令式ローマ字

对于日语假名的罗马字化, 简言之, 前者是美国传教士`Hepburn`基于英语发音创建的假名与罗马字的对应表, 对于英语使用者来说拼读更符合英语的习惯, 比如`つ`记为`tsu`. 而后者是日本政府公布的标准, `つ`记为`tu`.

但是常用的商业输入法, 如macOS与iOS日语输入法, Microsoft IME, Google 日语输入法, 为了用户的输入方便都使用了二者相结合的方式. 也就是说, 主要基于訓令式, 但一些不冲突的码会支持ヘポン式, 比如(`ja ju je jo`).

### 特殊的ん

传统`Hepburn`记法中, ん可以写成`m`或`n`, 比如群馬(ぐんま): `Gumma`. 但新式`Hepburn`记法中则和訓令式相同:

对于记法, 二者统一, 在元音和`y`之前写为`n'`, 其他情况记为`n`, 例:

- 簡易(かんい): kan'i
- カニ(かに): kani

但对于计算机输入, 输入单引号显得繁琐. 一是日语键盘配列中需要键入`Shift+7`来输出`'`. 二是手机端输入单引号需要切换面板.

因此虽然罗马字记法中使用`n`来输入`ん`, 但为了减少歧义, 更多情况下使用`nn`来输入`ん`.
这样一来, 记法就与编码不同. 对于Rime字典来说, 就需要手动更新. 例如将`kan'i`修改为`kanni`

### Solution

1. 使用假名作为字典编码, 也就是和`mozc`同样的方案. 经过测试, Rime支持假名编码方案, 这样的话可以直接使用`mozc`的字典. 可参见项目 [m13253/rime-nihongo-romaji](https://github.com/m13253/rime-nihongo-romaji) 和 [sgalal/rime-kunyomi](https://github.com/sgalal/rime-kunyomi)

2. 使用罗马字母进行编码, 通过Rime提供的`speller/algebra`转写, 此种方案需要自行编写脚本转换`mozc`的字典, 此项目已包含脚本[lazyfoxchan/rime-jaroomaji](https://github.com/lazyfoxchan/rime-jaroomaji/tree/master/dict_tools)

当前我正在使用方案2. 能满足基本输入要求, 能处理拨音, 也能处理动词变形, 其处理方案与`mozc`一致, 将变形写入字典, 如下所示:

```
まがった	ma ga xtu ta	60129
まがって	ma ga xtu te	60129
まがっと	ma ga xtu to	60129
曲がった	ma ga xtu ta	61962
曲がって	ma ga xtu te	61962
曲がっと	ma ga xtu to	61962
```

但此方案自动提示功能不可用, 待时间充足与精力恢复会逐渐准备方案1.

### zh zj zk zl

对于Google日语输入法(或开源版本的`mozc`, 字典源文件参见[GitHub mozc](https://github.com/google/mozc/blob/master/src/data/preedit/romanji-hiragana.tsv#LL11C5-L11C5)), 有以下一组mapping, 用于输入方向箭头, 显然`hjkl`代表的方向指向是源于Vim传统. 何人加入此项配置的不详.

```
zh	←
zj	↓
zk	↑
zl	→
```

但蹊跷的是, 在Apple日语输入法中, 同样有此feature. 而Windows日语IME却不支持. 由此猜测, 莫非Apple是复用/借鉴了Google日语输入法的词库? 而微软日语输入法则是"独立自主研发"? (www

简单搜索后并无答案, 但鉴于Apple进入日本市场的时间之早, 很难相信Apple会复用Google的词库, 许是Google复用/借鉴Apple日语输入法也说不准, 这其中想必有一段趣事. 只可惜实验室的那台`NeXTstation`已无法启动, 不然倒可以验证下90年代的macOS前身OS是否支持此feature.