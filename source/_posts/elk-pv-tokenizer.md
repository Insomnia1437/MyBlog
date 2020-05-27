---
title: 配置基于Elasticsearch的EPICS PV分析器
date: 2020-05-27 17:45:41
categories:
  - Software
  - elk
  - EPICS
tags:
  - Elasticsearch
  - Chinese
---

```python
# @Time    : 2020-05-27
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```
> 把EPICS PV存入ES后，为了方便搜索，需要自定义analyzer。

<!-- more -->


# ES 文本分析

基于[https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html)完成本文。

ES默认使用`standard analyzer`对一段文本进行操作，如分词（根据空格拆分），单词还原（`foxes`还原为`fox`），去除停用词（移除`the`,`a`等冠词）等。除默认分析器还有一系列内置分析器（`built-in analyzer`），用户也可以自定义分析器。

## Analyzer概述

### 文本分析器结构

`character filters, tokenizers, and token filters`，顾名思义，第一个是去除那些不想要的character，第二个是根据分隔符得到一组token，第三个是设置token的过滤和处理，下面详细讲。

### 使用时机

1. 索引一个`text`的field时进行分析，称为 `index analyzer`
2. 调用search API进行检索时对要搜索的文本进行分析`search analyzer`

大多数情况，使用同一个分析器。

### 如何设置分析器

对于`index analyzer`，可以直接设置`index`的默认分词器，也可以在`mapping`里设置`field`的分词器。

对于`search analyzer`，可以在查询语句里指定分析器，也可以设置`index` 和 `field`的搜索分析器，这其中有优先级，按顺序：

1. 查询语句中指定的分析器
2. `field`中指定的搜索分析器
3. `index`中指定的搜索分析器
4. `field`中指定的索引分析器
5. 默认的`standard analyzer`

```http
PUT my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "whitespace",
        "search_analyzer": "simple"
      }
    }
  }
}

PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "default": {
          "type": "simple"
        },
        "default_search": {
          "type": "whitespace"
        }
      }
    }
  }
}
GET my_index/_search
{
  "query": {
    "match": {
      "message": {
        "query": "Quick foxes",
        "analyzer": "stop"
      }
    }
  }
}
```

### Stemming

把单词还原到其词根，`be, are, and am`， `mouse and mice`， `foot and feet`有很多细节，不展开了。

### 如何测试分析器

```http
POST _analyze
{
  "analyzer": "whitespace",
  "text":     "The quick brown fox."
}
```

### 如何配置ES自带的分析器

```http
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "std_english": { 
          "type":      "standard",
          "stopwords": "_english_"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "my_text": {
        "type":     "text",
        "analyzer": "standard", 
        "fields": {
          "english": {
            "type":     "text",
            "analyzer": "std_english" 
          }
        }
      }
    }
  }
}

POST my_index/_analyze
{
  "field": "my_text", 
  "text": "The old brown cow"
}

POST my_index/_analyze
{
  "field": "my_text.english", 
  "text": "The old brown cow"
}
```

输出结过一个去除了`the`，一个保留。

### 如何自定义分析器

```http
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": { 
          "type": "custom",
          "char_filter": [
            "emoticons"
          ],
          "tokenizer": "punctuation",
          "filter": [
            "lowercase",
            "english_stop"
          ]
        }
      },
      "tokenizer": {
        "punctuation": { 
          "type": "pattern",
          "pattern": "[ .,!?]"
        }
      },
      "char_filter": {
        "emoticons": { 
          "type": "mapping",
          "mappings": [
            ":) => _happy_",
            ":( => _sad_"
          ]
        }
      },
      "filter": {
        "english_stop": { 
          "type": "stop",
          "stopwords": "_english_"
        }
      }
    }
  }
}

POST my_index/_analyze
{
  "analyzer": "my_custom_analyzer",
  "text": "I'm a :) person, and you?"
}
```

自定义内容：

- 字符过滤器，把颜文字替换掉
- 分词器：根据标点符号分词
- 过滤器：大写字母变小写，然后去除英语停用词

输出结果： `[ i'm, _happy_, person, you ]`

看完这个其实就可以自定义了，毕竟EPICS PV一般都长这样：

> `LIiEV:evg0-EvtClk:FracSynFreq-SP`

似乎设置`:`作为分词器，然后大写转小写就搞定了。

## 内置分析器

- Standard Analyzer
  - 根据一个叫`Unicode Text Segmentation algorithm`的算法分隔单词，转小写，支持移除停用词。

- Simple Analyzer
  - 不可配置，遇到不是字母的字符就分隔，转小写

- Whitespace Analyzer
  - 不可配置，遇到空格就分隔。

- Stop Analyzer
  - 比Simple Analyzer 多了移除停用词
- Keyword Analyzer
  - 什么也不干，把整段文字当作一个keyword，不可配置。就和`keyword` field一样。
- Pattern Analyzer
  - 基于正则表达式，默认用的Java `\W+`，匹配任何non-word的字符，也就是`[^a-zA-Z0-9_]`，用别的正则官方说可能导致速度很慢。支持转小写，设置停用词，设置停用词文件路径。
- Language Analyzer
  - 支持不同的语言，懒得看细节了。
- Fingerprint Analyzer
  - 支持小写，去重，排序，移除停用词，以及`concatenated into a single token`（没懂）。

## Character filter

- HTML Strip Char Filter
  - 处理HTML的，把`<p>I&apos;m so <b>happy</b>!</p>`  --->> `I'm, so, happy`
- Mapping Character Filter
  - 看上面自定义分析器的例子
- Pattern Replace Character Filter
  - 用正则表达式进行替换，可能很慢

## Tokenizer

分词器分为三大类：面向单词的，面向单词内的，面向结构型文本的。

### Word Oriented Tokenizers

- Standard Tokenizer
  - 用`Unicode Text Segmentation algorithm`去分隔字符，官方说它是最佳之选。
- Letter Tokenizer
  - 遇到不是字母的就分隔
- Lowercase Tokenizer
  - 比letter tokenizer 多了转小写
- Whtespace Tokenizer
  - 遇到空格就分隔
- UAX URL Email Tokenizer
  - 把URL和Email地址当成一个token
- Classic Tokenizer
  - 针对英语语法的分词器
- Thai Tokenizer
  - 泰语分词（那么多语言，ES居然专门把泰语分词器列出来，why）

### Partial Word Tokenizers

- N-Gram Tokenizer
  - 默认使用2-gram，`quick` --->> `[qu, ui, ic, ck]`
- Edge N-gram Tokenizer
  - 在单词头设置anchor，如果5-gram的话，`quick` --->> `[q, qu, qui, quic, quick]`

### Structured Text Tokenizers

- Keyword Tokenizer
  - 啥也不干
- Pattern Tokenizer
  - 正则
- Simple Pattern Tokenizer
  - 支持简单的正则，更快，返回匹配的项
- Char Group Tokenizer
  - 设置一组字符进行分隔，比正则快
- Simple Pattern Split Tokenizer
  - 遇到正则匹配的项就分隔，和上面那个的区别跑一下下面的例子就明白了
- Path_hierarchy Tokenizer
  - 根据路径分隔，`/one/two/three` --->> `[ /one, /one/two, /one/two/three ]`

```http
POST _analyze
{
  "tokenizer": {
    "type": "simple_pattern_split",
    "pattern": "[0123456789]{3}"
  },
  "text": "fd-786-335-514-x"
}

POST _analyze
{
  "tokenizer": {
    "type": "simple_pattern",
    "pattern": "[0123456789]{3}"
  },
  "text": "fd-786-335-514-x"
}
```



## Token filter

太多了，常用的有：

- ASCII folding token filter
  - 把一些拉丁字母转成ASCII字符
- Lowercase token filter
  - 转小写
- Length token filter
  - 移除长度超过设定值的，可以设置最小或者最大。
- Truncate token filter
  - 超过设定长度的就截短
- Stop token filter
  - 移除停用词

## EPICS PV

直接上代码，添加了`epics_pv_analyzer`，并在`PVNAME`这个field上使用，在mapping里设置了几个常见的EPICS PV field的映射。把`PVTYPE`设置为`keyword`就可以做聚合了，看看哪种类型的PV最多（好像没什么用）。

```http
PUT linac-epics-pv
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "epics_pv_analyzer": {
          "type": "custom",
          "tokenizer": "pv_tokenizer",
          "filter": [
            "lowercase"
          ]
        }
      },
      "tokenizer": {
        "pv_tokenizer": {
          "type": "char_group",
          "tokenize_on_chars": [
            ":",
            "-",
            "_",
            " "
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "PVNAME": {
        "type": "text",
        "analyzer": "epics_pv_analyzer",
        "search_analyzer": "epics_pv_analyzer"
      },
      "PVTYPE": {
        "type": "keyword",
        "ignore_above": 256
      },
      "DTYP": {
        "type": "text"
      },
      "NELM": {
        "type": "integer"
      },
      "PINI": {
        "type": "keyword"
      },
      "FLNK": {
        "type": "text",
        "analyzer": "epics_pv_analyzer",
        "search_analyzer": "epics_pv_analyzer"
      },
      "FTVL": {
        "type": "keyword"
      }
    }
  }
}
# 测试分析器
POST linac-epics-pv/_analyze
{
  "analyzer": "epics_pv_analyzer",
  "text": "LIiBM:SP_61_F5_1:Y1PK:ZRE:1S:INP"
}
# 修改mapping
PUT linac-epics-pv/_mapping
{
  "properties": {
    "PVNAME": {
      "type": "text",
      "analyzer": "epics_pv_analyzer",
      "search_analyzer": "epics_pv_analyzer"
    },
    "PVTYPE": {
      "type": "keyword",
      "ignore_above": 256
    },
    "DTYP": {
      "type": "text",
    },
    "NELM": {
      "type": "integer",
    },
    "PINI": {
      "type": "keyword",
    },
    "FLNK": {
      "type": "text",
      "analyzer": "epics_pv_analyzer",
      "search_analyzer": "epics_pv_analyzer"
    },
    "FTVL": {
      "type": "keyword",
    },
  }
}
# 搜索
GET linac-epics-pv/_search
{
  "size": 20, 
  "query": {
    "match": {
      "PVNAME": {
        "query": "liibm zre inp",
        "operator": "and"
      } 
    }
  }
}
```

