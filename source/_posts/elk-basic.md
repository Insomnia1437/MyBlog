---
title: ElasticSearch 基本概念
date: 2020-04-23 14:42:38
categories:
  - Software
  - elk
tags:
  - Kibana
  - Elasticsearch
  - Chinese
---

```python
# @Time    : 2020-04-23
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

> 前言：之前在LINAC安装了ES，采集了控制网络内EPICS beacon anomaly和Archive Appliance的数据，但是使用了一个非常老旧的桌面计算机，也没有修改什么参数，导致机器非常卡顿（文末会分析原因）。最近购入了一个新的服务器，准备migrate到新的机器上，再加上疫情在家隔离终于有时间去研究下ES的一些细节概念了。

<!-- more -->

**由于软件版本的更新，很多feature被移除（比如文档的`_type` metadata，`_default_` mapping），因此本文的概念基于Elasticsearch 7 以上版本**

原文在此： [Elasticsearch Reference 7.6](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/index.html)



## 基础概念

- cluster：集群， 一个cluster由多个node组成。

- node：节点，一个node就是一个ES实例，一台机器可以开启多个node，大多数情况就是一台机器或者一个独立的虚拟机环境。

- index：索引，既可以当动词也可以当名词用，一系列文档归于一个index内。

- shard：分片，是底层Lucene分配的基础单元，一个index有一个或者多个shard，shard会被ES动态分配在不同的node上，每个shard最大存放`Integer.MAX_VALUE - 128`个文档

  - primary shard：主分片，响应搜索服务，创建index时设定，除非重建索引否则数量不可变。
  - replica shard：副本分片，备份用主分片挂了，副本就会升级成主分片，数量随时可变


## 安装与配置

### 安装之前

首先确定自己的Java版本，操作系统，查看官方的[support matrix](https://www.elastic.co/support/matrix)。如果不在这个martix里，比如想在Raspberry Pi上安装，可能需要做一些修改。

### 配置

#### 总览

大部分配置都可以通过RESTful API进行动态修改。

一例：

```
curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
    "persistent" : {
        "indices.recovery.max_bytes_per_sec" : "50mb"
    }
}
'
```

剩下的一些初始化配置在`config`目录下，配置文件是`YAML`格式。

- `elasticsearch.yml` 配置Elasticsearch
- `jvm.options` 配置Elasticsearch JVM
- `log4j2.properties` 配置Elasticsearch logging

配置也可以通过环境变量导入。

#### 基础配置项

- `path.data`和`path.logs`：如果使用压缩包安装，为了防止升级时把这`data`和`logs`覆盖导致数据丢失，最好不放在ES文件夹内，`config`文件夹亦然。

- `cluster.name` 集群的名字，默认是`elasticsearch`

- `node.name` 节点的名字，默认是本机`hostname`

- `network.host`  节点IP

- `discovery.seed_hosts` 在ES启动过程中，有一个discovery的过程去寻找别的集群节点，别的节点的IP地址或者hostname（ES会自己用DNS lookup转换成IP地址）就在这个配置项中设置。同时可以指定端口，默认是9300。

- `cluster.initial_master_nodes` 设定初始的master node，集群启动时候有一个`cluster bootstrapping`的过程，会通过选举产生有资格成为master node的节点，如果没有配置这个bootstrapping会自动完成，但可能会由于节点discovery的速度过慢形成了多个集群，因此需要指定几个有选举资格的master node。（discovery和bootstrapping的细节我没有仔细研究，可能有理解错误。ES 7.0以前的版本使用`discovery.zen.minimum_master_nodes`这个配置项防止脑裂，现在已被忽略。

  ```yaml
  discovery.seed_hosts:
     - 192.168.1.10:9300
     - 192.168.1.11 
     - seeds.mydomain.com 
     - [0:0:0:0:0:ffff:c0a8:10c]:9301 
  cluster.initial_master_nodes: 
     - master-node-a
     - master-node-b
     - master-node-c
  ```

  

- `http.port` http端口 默认9200，用于RESTful API的通信
- `bootstrap.memory_lock` 允许JVM锁住内存，禁止swap。

#### JVM配置

查询JVM配置

```http
curl -XGET 'localhost:9200/_nodes/jvm?pretty'
```



##### 堆内存

使用`ES_JAVA_OPTS`环境变量设置

可以指定JVM版本，比如`8-9:-Xmx2g`代表如果JVM版本是8和9，设置最大堆内存为2 GB。默认值为1 GB。

通常需要设置**最大对内存和最小堆内存相同**，防止程序在运行时改变堆内存大小。由于JVM的内存指针压缩（compressed oops），最大值不要超过32 GB，官方推荐安全值26 GB，也有办法查看自己机器的不开启指针压缩的最大堆内存值。

> -Xms8g
> -Xmx8g

##### JVM缓存配置

ES会在`/tmp`目录下创建一些temp file，而有些Linux发行版可能会定期删除这些文件导致ES出现问题。如果使用包管理工具安装ES并且通过`systemd`运行，这种情况不会发生，如果使用`.tar.gz`下载就需要通过`$ES_TMPDIR`这个环境变量指定，同时这个目录也要设置permission，只允许ES的运行者访问。

#### 安全配置

通过`bin/elasticsearch-keystore create`创建`keystore`。目前似乎不是非常完善。

#### 日志配置

ES使用的是[Log4j 2](https://logging.apache.org/log4j/2.x/)，有非常多的配置项，以后需要使用再去研究。

#### xpack配置

xpack有许多功能都可以根据需要进行配置，比如Index lifecycle management (ilm)，Machine learning，Monitoring，Security，SQL，Watcher。稍微详细介绍。

#### 重要配置

ES默认用户在开发模式运行，允许很多重要的设置使用默认值，只给warning，但是一旦修改了网络设置，如`network.host`，ES会认为用户在生产运行模式，会把这些warning升级为exception，阻止ES启动。

##### 增加文件句柄数量

根据ES安装方式不同，配置文件也不同

- `.zip`和.`tar.gz`： 临时配置使用`ulimit`，永久配置在`/etc/security/limits.conf`。在Linux上需要使用`ulimit`来配置最大允许打卡的文件数。例如root 执行：`ulimit -n 65535`

- 包管理工具：RPM在`/etc/sysconfig/elasticsearch`，Debian在`/etc/default/elasticsearch`

- 包管理工具切使用了`systemd`：`sudo systemctl edit elasticsearch`，改成

  ```sh
  [Service]
  LimitMEMLOCK=infinity
  ```

  然后`sudo systemctl daemon-reload`

最后用下面的语句去查询。（这个配置在Windows上不用管，返回值是-1）

```http
GET _nodes/stats/process?filter_path=**.max_file_descriptors
```

##### 禁用swapping

也有不同的方式，懒得一一罗列，可以通过下面这个语句查看是否禁用

```http
GET _nodes?filter_path=**.mlockall
```

##### 虚拟内存

ES使用`mmapfs`目录存indices（？？？不懂），在Linux上要增加mmap count。RPM 和 Debian安装的已自动设置。

```sh
sudo sysctl -w vm.max_map_count=262144
# 也可以在文件里改，永久生效
sudo vim /etc/sysctl.conf
# 检验是否生效
sysctl vm.max_map_count
```

##### 线程数

使用`systemd`运行的忽略此项。

```sh
sudo ulimit -u 4096
# 或者设置 nproc = 4096
sudo vi /etc/security/limits.conf
```

#### bootstrap check

如果只部署了一个node，且不设置`transport`或设置discovery 类型为`single-node`，bootstrap check就会被跳过，这可能导致一些问题，所以最好强制进行check，可以在JVM设置里`es.enforce.bootstrap.checks= true`来强制开启。

check会检查上面所列的一些重要配置项，除此之外，还会检查：

- 最大文件大小

- JVM是client JVM还是server JVM
- GC
- system call filters
- OnError和OnOutOfMemoryError
- early-access
- G1GC
- All permission
- discovery配置检测
  - `discovery.seed_hosts`
  - `discovery.seed_providers`
  - `cluster.initial_master_nodes`

## 模块

ES有不同的模块负责不同的功能，每个模块的配置分为两种，一种必须在配置文件中指定（或者在环境变量），另一种可以通过API动态改变。

### Node

首先介绍node，开启一个ES的时候其实就是启动了一个node，一系列node的集合就是一个集群。每个node使用`HTTP`和`Transport`两个模块进行信息交换，前者用于服务REST客户端，后者是用于节点之间的交互。

节点类型有四种类型，在`elasticsearch.yml`文件中设置。可以既是`master-eligible`也是`data`节点。如果所有类型都设置为false，那节点就成为一个类似于proxy的coordinating node，负责转发请求和结果聚合。

#### Master-eligible node

配置项：`node.master`，使一个节点拥有成为参与投票选举master节点的资格。设置`node.voting_only`可以让这个节点只有选举权没有被选举权。

为了保证master节点有足够的资源，如果资源富裕，可以禁用别的类型。

```yaml
node.master: true 
node.data: false 
node.ingest: false 
cluster.remote.connect: false
```

#### Data node

配置项：`node.data`，数据节点保存数据，可以进行CRUD，搜索和聚合。

#### Ingest node

配置项：`node.ingest`，使用pipeline，用于数据预处理（移除某个field，或者重命名），也可以实现一些复杂的逻辑，条件语句，正则表达式，异常处理。 （感觉和logstash功能重叠了）

#### Machine learning node

用于机器学习，略。

### Discovery and cluster formation

这个模块负责发现节点，组成集群，选举master，改变集群状态。

存在一个`seed hosts providers`的概念，有两种providers：a *settings*-based and a *file*-based seed hosts provider。通过配置`discovery.seed_providers`起作用。

选取master节点是基于`quorum`（法定人数，即大于多少个节点的认可来保证只有一个master），在一批`master-eligible`节点中进行选举，具体办法可以通过`voting configuration`配置。这个过程在`cluster bootstrapping`过程中完成。

添加和移除节点时候要多加小心，不要一次性移除太多节点，否则可能会导致没有足够多的`master-eligible`节点，导致集群无法正常工作。

[一些技术细节可以参考知乎的这篇文章](https://zhuanlan.zhihu.com/p/34858035)

### Shard allocation and cluster-level routing

master节点的一个主要任务是决定分片分配以及在rebalance时如何移动分片。

ES默认会监测节点的磁盘可用空间来选择分片分配方案，如下例，

- 低警戒线：磁盘剩余空间不足100G，则停止向这个节点分配副本分片。默认85%。
- 高警戒线：磁盘剩余空间不足50G，则把此节点的分片移动到别的节点。默认90%。
- 洪水水位线：剩余不足10G，则将所有index标记为只读。默认95%。
- 检测间隔为1分钟

几个设置值要么都是百分比，要么都是字节值，不能混用

```http
curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "100gb",
    "cluster.routing.allocation.disk.watermark.high": "50gb",
    "cluster.routing.allocation.disk.watermark.flood_stage": "10gb",
    "cluster.info.update.interval": "1m"
  }
}
'
```

### Gateway

控制在集群重启时的一些策略。

集群重启时，如果只有部分节点重启，集群发现缺失了节点，就会启动rebalance，在节点之间复制数据，这会消耗大量I/O和带宽。比较重要的配置项有`gateway.recover_after_time` 和 `gateway.recover_after_*`。

### HTTP

提供基于HTTP协议的ES APIs，建议客户端使用HTTP长连接以提高性能。默认端口9200

### Transport

用于节点之间的内部通信。默认端口9300，可以设置多个profile指向不同的网卡地址和端口。

### Network setting

ES默认只会绑定localhost，经常使用的网络设置项：

- `network.host`
- `discovery.seed_hosts`
- `http.port`
- `transport.port`

还有一些高级设置，比如需要proxy才能访问ES时的配置，可以自己去看官方文档。

### Indices

一些和indices相关的配置

#### 静态设置项

- `index.number_of_shards` 主分片的数量，默认为1。最大为1024
- `index.shard.check_on_startup` 打卡分片之前是否进行check，默认为false
- `index.codec` 默认使用`LZ4`压缩，可以设置为`best_compression`来提高压缩率
- `index.routing_partition_size` 用于 routing
- `index.load_fixed_bitset_filters_eagerly` Indicates whether cached filters are pre-loaded for nested queries. Possible values are true (default) and false.

#### 动态设置项

- `index.number_of_replicas` 副本分片数量，默认为1。
- `index.auto_expand_replicas` 自动扩充副本数量，设置为`0-5`，默认为false。只会考虑设置的部分分片分配的规则。
- `index.search.idle.after`  默认30秒之后一个shard才能响应搜索
- `index.refresh_interval` 默认为1秒，设置为-1可禁用。

#### Circuit breaker

ES有很多种断路器，都是指定一个内存使用的limit，防止OOM。

#### Field data

主要用于缓存搜索或者聚合时使用的field，设置cache `indices.fielddata.cache.size`

#### Node query cache

filter 查询时的cache，设置`indices.queries.cache.size`

#### Indexing buffer

顾名思义。`indices.memory.index_buffer_size`

#### Shard request cache

分片级别的cache。查询：

`curl -X GET "localhost:9200/_stats/request_cache?human&pretty"`

`curl -X GET "localhost:9200/_nodes/stats/indices/request_cache?human&pretty"`

#### Index recovery

以下两种情况带来的索引重建时候的配置

- 分片丢失，索引重建
- 集群rebalance或者更改了分片分配设置项，索引重建

设置`indices.recovery.max_bytes_per_sec`：Limits total inbound and outbound recovery traffic for each node，默认为 40 MB

### Thread pool

线程池的设置项。略。

## 集群操作

### 重启

重启集群可以直接关闭整个集群，也可以一次只关闭一个node，从而不影响服务。

##### 关闭shard分配

关闭一个节点时，集群会等待`index.unassigned.node_left.delayed_timeout` （默认一分钟），然后开始把shard复制到别的节点上，而这个过程会花费很多I/O。因此，应该首先禁用此项。

```http
curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.enable": "primaries"
  }
}
'
```

##### 停止索引文档，刷新缓冲区

```http
curl -X POST "localhost:9200/_flush/synced?pretty"
```

synced flush会在ES 8.0中被移除，以后直接使用 `_flush`，二者效果相同。

##### 关闭节点

根据运行的方式：

- `sudo systemctl stop elasticsearch.service`
- `sudo -i service elasticsearch stop`
- `kill $(cat pid)`

##### 重启节点

如果有master nodes，先重启它们，然后等待它们互相发现，组成集群。可以通过下面的API查看集群状态

```http
curl -X GET "localhost:9200/_cat/health?pretty"
curl -X GET "localhost:9200/_cat/nodes?pretty"
```

##### 等待所有节点加入集群，集群状态为yellow

等节点恢复自己的主分片，此时副本分片还未完成分配。

##### 重新开启shard分配

```http
curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
'
```

此时集群开始根据节点数量和设置的副本分片数来分配副本分片到不同的node。

### 升级

官方建议，先升级非master-eligible节点，通过`GET /_nodes/_all,master:false`找到它们，然后升级master-eligible节点。

所有节点必须使用相同的版本。

##### 升级之前

- 检查是否使用了一些deprecated features
- 检查更新日志中一些突破性的改变，是否需要更改配置文件
- 在一个独立的环境中测试需要升级的版本
- **使用snapshot对数据进行备份！**

##### 升级 （关闭节点之前的过程和重启节点一样）

unset `cluster.initial_master_nodes`，升级后的node不需要再进行`bootstrapping check`。

- Debian 或 RPM 包：使用rpm或dpkg安装新版本
- 压缩包安装：解压到一个**新目录**（否则data可能被删除！），同时为了升级方便，之前应该已经把`config`，`data`和`logs`目录放在一个单独的文件夹。此时可以设置`ES_PATH_CONF`环境变量指定`config`目录的位置，设置`path.data`和`path.logs`为`data`和`logs`目录。
  如果当时没有把目录分开，那就要把旧的`config`，`data`和`logs`目录copy到刚解压的新目录下。

之后升级插件，开启shard分配，等待node recover。

## 映射

映射（mapping）就是把定义ES如何索引一个文档的方式。包括

- 把哪些string field定义为full text field。（用于词法分析）
- 包含数字，日期，地理信息的field
- 日期的格式
- 设置自定义规则进行动态field添加

根据不同的需求可以设置不同的映射类型，比如要搜索可以把`string`定义为`text`类型，如果只是排序或者聚合，可以定义为`keyword`类型。也可以设置不同的词法分析器。

每个映射类型都有自己的`parameters`

### 基本操作

按例子中的顺序：创建映射，添加新映射field，查看现有的映射，查看某一个field的映射（如果index太大的话很有用）

> 不能更新映射，如果想更新，可以新建一个index，然后用`reindex`API操作，或者使用`alias`创建一个新的field name。

```http
curl -X PUT "localhost:9200/my-index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "age":    { "type": "integer" },  
      "email":  { "type": "keyword"  }, 
      "name":   { "type": "text"  }     
    }
  }
}
'
curl -X PUT "localhost:9200/my-index/_mapping?pretty" -H 'Content-Type: application/json' -d'
{
  "properties": {
    "employee-id": {
      "type": "keyword",
      "index": false
    }
  }
}
'
curl -X GET "localhost:9200/my-index/_mapping?pretty"
curl -X GET "localhost:9200/my-index/_mapping/field/employee-id?pretty"
```

#### 映射爆炸

动态映射可能导致一个index的fields太多，可以设置limit，比如`index.mapping.total_fields.limit`，默认1000个fields。也可以设置映射的最大深度，默认20。

### Mapping types

ES 7.0开始移除了mapping types，API也发生了变化。东西太多，略去。（反正我也没用过以前版本的ES)

### Dynamic mapping

可以在index文档的时候动态映射。默认情况，当ES发现一个新field时，会自动添加映射。把`dynamic`设置成`false`可以禁用动态映射，设置成`strict`抛出异常。

ES 根据JSON 的类型来生成自己的映射类型，可以动态映射的field包括：

- `null` 会被忽略
- `boolean`
- `float`
- `long`
- `object`
- 数组的类型根据非空的第一个元素类型确定
- JSON 中的string可以映射为`date` `double` `long` `text` `keyword`

#### Dynamic templates

可以设置一个模板指定动态映射。

下例：把映射`integer`而非`long`，把`string`映射为`text`和`keyword`（使用了 `multi-fields`）

其中`match_mapping_type`是其中一种match conditions，别的还有`match`, `match_pattern`, `unmatch`, `path_match`, `path_unmatch`。`match`和`unmatch`使用简单的pattern匹配和排除一些field name，`match_pattern`可以使用Java正则，`path_match`则可以匹配具体的路径：`some_object.*.some_field`

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "dynamic_templates": [
      {
        "integers": {
          "match_mapping_type": "long",
          "mapping": {
            "type": "integer"
          }
        }
      },
      {
        "strings": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "text",
            "fields": {
              "raw": {
                "type":  "keyword",
                "ignore_above": 256
              }
            }
          }
        }
      }
    ]
  }
}
'
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "my_integer": 5, 
  "my_string": "Some string" 
}
'
```

也可以用`{name}` 和 `{dynamic_type}`这俩placeholder去替换，如下例：`english` field的映射参数：`analyzer`的值被替换为field的name，也就是`english`，`count`的`type`是`long`。

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "dynamic_templates": [
      {
        "named_analyzers": {
          "match_mapping_type": "string",
          "match": "*",
          "mapping": {
            "type": "text",
            "analyzer": "{name}"
          }
        }
      },
      {
        "no_doc_values": {
          "match_mapping_type":"*",
          "mapping": {
            "type": "{dynamic_type}",
            "doc_values": false
          }
        }
      }
    ]
  }
}
'
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "english": "Some English text", 
  "count":   5 
}
'
```

### Fields datatypes

#### Core datatypes

##### string

- `text` 如邮件正文，产品描述信息。
- `keyword` 如IDs，邮箱，主机名，状态码，邮编，或标签。有时候一个field既可以是数字类型也可以是`keyword`类型，比如产品ID，这时候如果不会使用`range`查询，就尽量使用`keyword`，因为更快。也可以设置`multi-field`，同时映射两种。

##### numeric

- `long`  有符号64位
- `integer` 有符号32位
- `short` 有符号16位 (-32768, 32767)
- `byte` 有符号8位
- `double` 双精度64位浮点数 IEEE754标准
- `float` 32位浮点数
- `half_float` 16位浮点数
- `scaled_float`  A floating point number that is backed by a `long`, scaled by a fixed `double` scaling factor. 比如CPU利用率是12.7%，设置`scaling_factor`为100，虽然查询出来是浮点数，但ES会先把它取整然后用整型存来节省空间，所以有个balance的过程，这个值越大，ES存的数精确度越高，但是占用的空间也越多。

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "number_of_bytes": {
        "type": "integer"
      },
      "time_in_seconds": {
        "type": "float"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      }
    }
  }
}
'
```

##### date

ES支持三种date：

- strings containing formatted dates, e.g. `"2015-01-01"` or `"2015/01/01 12:10:30"`.
- a long number representing *milliseconds-since-the-epoch*. 
- an integer representing *seconds-since-the-epoch*.

如果不指定format，默认为`strict_date_optional_time||epoch_millis`，也可以指定多个日期格式。

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "date": {
        "type":   "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      }
    }
  }
}
'
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{ "date": "2015-01-01" }
'
curl -X PUT "localhost:9200/my_index/_doc/2?pretty" -H 'Content-Type: application/json' -d'
{ "date": "2015-01-01T12:10:30Z" }
'
curl -X PUT "localhost:9200/my_index/_doc/3?pretty" -H 'Content-Type: application/json' -d'
{ "date": 1420070400001 }
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "sort": { "date": "asc"} 
}
'

```

##### date_nanos

纳秒级的时间，范围从1970到2262年。但是聚合还是只能基于millisecond级别。

##### boolean

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "is_published": {
        "type": "boolean"
      }
    }
  }
}
'
curl -X POST "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "is_published": "true" 
}
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "term": {
      "is_published": true 
    }
  }
}
'
```

##### binary

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "blob": {
        "type": "binary"
      }
    }
  }
}
'
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "name": "Some binary blob",
  "blob": "U29tZSBiaW5hcnkgYmxvYg==" 
}
'
```

##### range

支持类型：

- `integer_range`
- `float_range`
- `long_range`
- `double_range`
- `date_range`
- `ip_range`

```http
curl -X PUT "localhost:9200/range_index?pretty" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "number_of_shards": 2
  },
  "mappings": {
    "properties": {
      "expected_attendees": {
        "type": "integer_range"
      },
      "time_frame": {
        "type": "date_range", 
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      }
    }
  }
}
'
curl -X PUT "localhost:9200/range_index/_doc/1?refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "expected_attendees" : { 
    "gte" : 10,
    "lte" : 20
  },
  "time_frame" : { 
    "gte" : "2015-10-31 12:00:00", 
    "lte" : "2015-11-01"
  }
}
'
```

#### Complex datatypes

##### object

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": { 
      "region": {
        "type": "keyword"
      },
      "manager": { 
        "properties": {
          "age":  { "type": "integer" },
          "name": { 
            "properties": {
              "first": { "type": "text" },
              "last":  { "type": "text" }
            }
          }
        }
      }
    }
  }
}
'
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{ 
  "region": "US",
  "manager": { 
    "age":     30,
    "name": { 
      "first": "John",
      "last":  "Smith"
    }
  }
}
'
```

##### nested

`nested`是特殊的`object`类型，可以被`nested`query独立的检索。如下例，如果不用`nested`，搜索`Alic Smith`也会返回匹配结果，但那是错误的结果。

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "user": {
        "type": "nested" 
      }
    }
  }
}
'
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "group" : "fans",
  "user" : [
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "Smith" }} 
          ]
        }
      }
    }
  }
}
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "White" }} 
          ]
        }
      },
      "inner_hits": { 
        "highlight": {
          "fields": {
            "user.first": {}
          }
        }
      }
    }
  }
}
'
```

#### Geo datatypes

//略

##### geo_point

##### geo_shape

#### Specialised datatypes

##### ip

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "ip_addr": {
        "type": "ip"
      }
    }
  }
}
'
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "ip_addr": "192.168.1.1"
}
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "term": {
      "ip_addr": "192.168.0.0/16"
    }
  }
}
'
```

##### completion

`completion` to provide auto-complete suggestions

//略

##### token_count

这里使用了multi-fields中的`fields`，对一个`name` field添加了两个映射，一个是`text`，一个是`token_count`。

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "name": { 
        "type": "text",
        "fields": {
          "length": { 
            "type":     "token_count",
            "analyzer": "standard"
          }
        }
      }
    }
  }
}
'
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{ "name": "John Smith" }
'
curl -X PUT "localhost:9200/my_index/_doc/2?pretty" -H 'Content-Type: application/json' -d'
{ "name": "Rachel Alice Williams" }
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "term": {
      "name.length": 3 
    }
  }
}
'
```

##### murmur3

用于计算hash值，需要在每一个node上都安装插件，且安装后必须restart

`sudo bin/elasticsearch-plugin install mapper-murmur3`

`cardinality`聚合返回去重的值，如果直接对`my_field`使用结果一样，但是hash速度更快。不鼓励对数字类型使用，且如果大部分string field都不重复，也不建议使用。

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "my_field": {
        "type": "keyword",
        "fields": {
          "hash": {
            "type": "murmur3"
          }
        }
      }
    }
  }
}
'
# Example documents
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "my_field": "This is a document"
}
'
curl -X PUT "localhost:9200/my_index/_doc/2?pretty" -H 'Content-Type: application/json' -d'
{
  "my_field": "This is another document"
}
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "aggs": {
    "my_field_cardinality": {
      "cardinality": {
        "field": "my_field.hash" 
      }
    }
  }
}
'

```

##### percolator

过滤器，太复杂 //TODO

##### join

定义文档的关系，太复杂 //TODO

##### rank feature

##### rank features

##### dense vector

##### sparse vector

##### search-as-you-type

##### alias

##### flattened

##### shape

##### histogram

#### Arrays

ES没有特定的`arrya` datatype，所有field都可以包含0或多个值，但是类型必须相同。

```http
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "message": "some arrays in this document...",
  "tags":  [ "elasticsearch", "wow" ], 
  "lists": [ 
    {
      "name": "prog_list",
      "description": "programming list"
    },
    {
      "name": "cool_list",
      "description": "cool stuff list"
    }
  ]
}
'
curl -X PUT "localhost:9200/my_index/_doc/2?pretty" -H 'Content-Type: application/json' -d'
{
  "message": "no arrays in this document...",
  "tags":  "elasticsearch",
  "lists": {
    "name": "prog_list",
    "description": "programming list"
  }
}
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "tags": "elasticsearch" 
    }
  }
}
'
```

#### Multi-fields

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "city": {
        "type": "text",
        "fields": {
          "raw": { 
            "type":  "keyword"
          }
        }
      }
    }
  }
}
'
curl -X PUT "localhost:9200/my_index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "city": "New York"
}
'
curl -X PUT "localhost:9200/my_index/_doc/2?pretty" -H 'Content-Type: application/json' -d'
{
  "city": "York"
}
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "city": "york" 
    }
  },
  "sort": {
    "city.raw": "asc" 
  },
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.raw" 
      }
    }
  }
}
'
```

### Meta-fields

#### `_index`

文档索引field

#### `_type`

移除

#### `_id`

文档的id

#### `_source`

包含原始的JSON文档

#### `_size`

`_source`field内容的大小，需要安装。

`sudo bin/elasticsearch-plugin install mapper-size`

#### `_field_names`

存所有的非空field，用于exists`查询。

#### `_ignored`

返回被忽略的那些field。被忽略的field使用`ignore_malformed`映射参数指定。

#### `_routing`

通过路由，选择一个特定的分片操作。默认使用id，也可以自定义一个值。

#### `_meta`

自定义的一些meta data。

### Mapping parameters

//TODO

## 搜索

在`regexp`和`query_string`中使用了正则，ES的正则语法：[Regular expression syntax](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/regexp-syntax.html)

### 基本概念

ES使用基于`Query DSL`的语法定义查询。

搜索使用相关性评分，也就是返回值里的`_score`。

#### Query context

判断文档符合查询的程度，以`_score`标识。

#### Filter context

过滤不符合要求的文档，通常用于选择结构性的数据，如日期范围。

例子：`match`语句用来得出文档匹配度，`filter`只过滤不评分。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "title":   "Search"        }},
        { "match": { "content": "Elasticsearch" }}
      ],
      "filter": [ 
        { "term":  { "status": "published" }},
        { "range": { "publish_date": { "gte": "2015-01-01" }}}
      ]
    }
  }
}
'
```

一个典型的搜索语句，查询`bank`这个index里的所有文档，按照`account_number`排序，分页显示，每页10个结果，从第10个结果开始显示。

### Match all 查询

查询所有文档，使用`boost`更改评分。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match_all": { "boost" : 1.2 }
    }
}
'
```

### Term-level 查询

查询结构数据（如日期，IP地址，价格，产品ID等），这个应该是最常用的。

#### exists

查field是否存在，不存在的情况：

- 值为空字符串，如`“”`  `-`
- 数组中包含`null`，如 `[null, "foo"]`
- 在field mapping里自定义的`null-value`

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "exists": {
            "field": "user"
        }
    }
}
'
```

#### fuzzy

返回根据`Levenshtein distance`计算的相似值。

`fuzziness`指定最大的编辑距离。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "fuzzy": {
            "user": {
                "value": "ki",
                "fuzziness": "AUTO",
                "max_expansions": 50,
                "prefix_length": 0,
                "transpositions": true,
                "rewrite": "constant_score"
            }
        }
    }
}
'
```

#### ids

根据文档的`_id`field查询，

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "ids" : {
            "values" : ["1", "4", "100"]
        }
    }
}
'
```

#### prefix

查prefix，可以通过使用`index_prefixes`这个mapping参数让ES保持prefix，从而提高检索速度。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "prefix": {
            "user": {
                "value": "ki"
            }
        }
    }
}
'
```

#### range

返回范围内的文档。

参数：

- `gt`：大于
- `gte`：大于等于
- `lt`
- `lte`
- `format`：日期的格式，默认使用mapping时定义的格式。
- `relation`： Indicates how the range query matches values for `range` fields
  - `INTERSECTS`(Default): Matches documents with a range field value that intersects the query’s range.
  - `CONTAINS`：Matches documents with a range field value that entirely contains the query’s range.
  - `WITHIN`：Matches documents with a range field value entirely within the query’s range.
- `time_zone`
- `boost`: 更改文档评分

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "range": {
      "timestamp": {
        "time_zone": "+01:00",        
        "gte": "2020-01-01T00:00:00", 
        "lte": "now"                  
      }
    }
  }
}
'
```

#### regexp

正则表达式

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "regexp": {
            "user": {
                "value": "k.*y",
                "flags" : "ALL",
                "max_determinized_states": 10000,
                "rewrite": "constant_score"
            }
        }
    }
}
'
```

#### term

完整匹配，不要用于`text`field。对于`text`，应使用`Full text查询`中的`match`查询。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "term": {
            "user": {
                "value": "Kimchy",
                "boost": 1.0
            }
        }
    }
}
'
```

#### terms

多个`term`。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "terms" : {
            "user" : ["kimchy", "elasticsearch"],
            "boost" : 1.0
        }
    }
}
'
```

#### terms_set

返回匹配terms中子集的文档，例子中`required_matches`是index的一个field。

```http
curl -X GET "localhost:9200/job-candidates/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "terms_set": {
            "programming_languages": {
                "terms": ["c++", "java", "php"],
                "minimum_should_match_field": "required_matches"
            }
        }
    }
}
'
```

#### type

搜索文档类型，ES 7.0 DEPRECATED

#### wildcard

匹配通配符，不知道和`regexp`哪个快。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "wildcard": {
            "user": {
                "value": "ki*y",
                "boost": 1.0,
                "rewrite": "constant_score"
            }
        }
    }
}
'
```

### Compound 查询

#### bool 

`query`里可以指定`bool`查询，包括

- `must`：必须匹配这些条件，贡献评分
- `must_not`：必须不匹配，不影响评分
- `should`：满足这些语句中的任意语句，会增加文档相关性得分
- `filter`：必须匹配，不影响评分

```http
curl -X POST "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user" : "kimchy" }
      },
      "filter": {
        "term" : { "tag" : "tech" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tag" : "wow" } },
        { "term" : { "tag" : "elasticsearch" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
'
```

#### boosting 

`positive`里的文档必须保护，`negative`里的文档减评分，减多少用 `negative_boost`指明。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "boosting" : {
            "positive" : {
                "term" : {
                    "text" : "apple"
                }
            },
            "negative" : {
                 "term" : {
                     "text" : "pie tart fruit crumble tree"
                }
            },
            "negative_boost" : 0.5
        }
    }
}
'
```

#### constant_score 

`constant_score` 会给匹配`filter`内语句的文档一个默认`1.0`的评分。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "constant_score" : {
            "filter" : {
                "term" : { "user" : "kimchy"}
            },
            "boost" : 1.2
        }
    }
}
'
```

#### dis_max 

必须匹配一个或多个子句，匹配的多，评分就高。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "dis_max" : {
            "queries" : [
                { "term" : { "title" : "Quick pets" }},
                { "term" : { "body" : "Quick pets" }}
            ],
            "tie_breaker" : 0.7
        }
    }
}
'
```

#### function_score 

可以根据function更改文档的评分，东西太多，懒得看了。下面的`random_score`就是其中一种function。

```http
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "function_score": {
            "query": { "match_all": {} },
            "boost": "5",
            "random_score": {}, 
            "boost_mode":"multiply"
        }
    }
}
'
```

### Full text 查询

查询`analyzed text fields`的，比如邮件正文。查询时用的文本 analyzer 和index时候用的一样。

//TODO 以后再填坑

#### interval 

#### match

#### match_bool_prefix

#### match_phrase

#### match_phrase_prefix

#### multi_match

#### common terms

#### query_string

#### simple_query_string

### Geo 查询

地址位置查询

//TODO

#### geo_bounding_box

#### geo_distance

#### geo_polygon

#### geo_shape

### Shape 查询

//TODO

### Joining 查询

#### nested 

索引应当保护一个`nested` field。

例子：

```http
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
    "mappings" : {
        "properties" : {
            "obj1" : {
                "type" : "nested"
            }
        }
    }
}
'
curl -X GET "localhost:9200/my_index/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query":  {
        "nested" : {
            "path" : "obj1",
            "query" : {
                "bool" : {
                    "must" : [
                    { "match" : {"obj1.name" : "blue"} },
                    { "range" : {"obj1.count" : {"gt" : 5}} }
                    ]
                }
            },
            "score_mode" : "avg"
        }
    }
}
'

```



#### has_child 

使用`join` 映射的文档。

//TODO

#### has_parent

使用`join` 映射的文档。

//TODO

#### Parent ID

使用`join` 映射的文档。

//TODO

### Span 查询

官方说主要用于查询法律文书和专利文件。

个么，略。

### Specialized 查询

不属于其它组的查询。

//TODO

## 聚合

聚合有四种概念：

- `bucket`
- `metric`
- `matrix`
- `pipeline`

聚合的结构：

```json
"aggregations" : {
    "<aggregation_name>" : {
        "<aggregation_type>" : {
            <aggregation_body>
        }
        [,"meta" : {  [<meta_data_body>] } ]?
        [,"aggregations" : { [<sub_aggregation>]+ } ]?
    }
    [,"<aggregation_name_2>" : { ... } ]*
}
```

一个典型的聚合：首先按照`state`分类为不同的`bucket`，然后计算了每个`state`的平均银行余额并按照余额排序。

```http
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
'
```

### Metric 聚合

#### Avg 聚合

```http
curl -X POST "localhost:9200/exams/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "grade_avg" : {
            "avg" : {
                "field" : "grade",
                "missing": 10 
            }
        }
    }
}
'
```

求平均值，`missing`可以指定如果field缺失的默认值。

#### Weighted Avg 聚合

```http
curl -X POST "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "weighted_grade": {
            "weighted_avg": {
                "value": {
                    "field": "grade",
                    "missing": 2
                },
                "weight": {
                    "field": "weight",
                    "missing": 3
                }
            }
        }
    }
}
'

```

计算加权平均，一看就懂。

#### Cardinality 聚合

返回一个field distinct或者unique值的数目，也就是去重

#### Extended Stats 聚合

计算一些统计值，看例子：

```
curl -X GET "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "grades_stats" : { "extended_stats" : { "field" : "grade" } }
    }
}
'

返回值
{
    ...

    "aggregations": {
        "grades_stats": {
           "count": 2,
           "min": 50.0,
           "max": 100.0,
           "avg": 75.0,
           "sum": 150.0,
           "sum_of_squares": 12500.0,
           "variance": 625.0,
           "std_deviation": 25.0,
           "std_deviation_bounds": {
            "upper": 125.0,
            "lower": 25.0
           }
        }
    }
}

```

//TODO 聚合的东西太多了，用到的却没几个，以后再慢慢添加。

## Sctipting (脚本)

ES可以使用`script`。关于`script`，本人没有需求，所以就不深入了解了，这篇腾讯云上的文章写的不错。[Elasticsearch7.X Scripting脚本使用详解](https://cloud.tencent.com/developer/article/1507715)

## ILM index生命周期管理

用于管理index，有几种策略

- **Rollover**  在index达到一定大小时创建新的index，并用alias指向新的index
- **Shrink**  减少index中的主分片
- **Force merge** 减少segments数（Lucene内部的存储形式）
- **Freeze**  把index设为只读
- **Delete** 删除index

### Index lifecycle

有几种生命周期阶段

- hot 正在更新，可被查询。
- warm 不更新，可被查询
- cold 不更新，几乎不被查询 （允许查询速度慢一些）
- delete  

### ILM Phase actions

不同的生命周期有不同的actions

- Hot 
  - Set Priority：设置index的优先级
  - Unfollow：把一个CCR （cross-cluster replication）的index转为普通的index
  - Rollover：最常用，下面介绍
- Warm
  - Set Priority
  - Unfollow
  - Read-Only：设为只读
  - Allocate：设置哪个node允许保存索引分片，以及改变副本分片数量
  - Shrink：设置index为只读，并且减少主分片数量
  - Force Merge：合并segments
- Cold
  - Set Priority
  - Unfollow
  - Allocate
  - Freeze：Freeze 索引，最小化内存占用
- Delete
  - Wait For Snapshot
  - Delete

### Rollover

#### 创建生命周期策略

主要还是通过kibana进行操作，API的话：

```http
curl -X PUT "localhost:9200/_ilm/policy/datastream_policy?pretty" -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "hot": {                      
        "actions": {
          "rollover": {
            "max_size": "50GB",     
            "max_age": "30d"
          }
        }
      },
      "delete": {
        "min_age": "90d",           
        "actions": {
          "delete": {}              
        }
      }
    }
  }
}
'
```

#### 创建索引模板

例子：

```http
curl -X PUT "localhost:9200/_template/datastream_template?pretty" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["datastream-*"],                 
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "datastream_policy",      
    "index.lifecycle.rollover_alias": "datastream"    
  }
}
'
```

#### 索引引导

引导一个初始index，并且指定索引模板中设置好的alias

```http
curl -X PUT "localhost:9200/datastream-000001?pretty" -H 'Content-Type: application/json' -d'
{
  "aliases": {
    "datastream": {
      "is_write_index": true
    }
  }
}
'
```

#### 验证

查看ILM状态

```http
curl -X GET "localhost:9200/datastream-*/_ilm/explain?pretty"
```

## 集群管理

### 集群状态：

- Green：所有primary shard和replica都工作正常
- Yellow：所有primary shard工作正常，但是有replica没有分配就绪（如果cluster里只有一个node，那副本分片和主分片就在一台机器上，这时候集群就是yellow
- Red：有主分片挂了

### Monitor

官方建议使用一个独立的集群监控集群状态。在ES 6.5之后应该使用Metricbeat。而非之前的`exporters`

#### Monitoring settings

默认xpack监控被启用，但是数据收集被禁用。

xpack监控：`xpack.monitoring.enabled`，默认为`true`

开启数据采集：`xpack.monitoring.collection.enabled`，默认为`false`

采集间隔：`xpack.monitoring.collection.interval`，默认为`10s`

ES集群采集：`xpack.monitoring.elasticsearch.collection.enabled`，此项仅禁用ES数据采集，别的数据采集依然允许（kibana，logstash，beats，APM Server）

在`kibana.yml`中配置monitoring数据的显示方式，在`logstash.yml`中配置monitoring数据的采集方式。

### Metricbeat

1. 开启数据采集

   ```http
   curl -X GET "localhost:9200/_cluster/settings?pretty"
   curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
   {
     "persistent": {
       "xpack.monitoring.collection.enabled": true
     }
   }
   '
   ```

2. 在每一个node上安装Metricbeat

3. 使用Metricbeat的模块：
   `metricbeat modules enable elasticsearch-xpack`

4. 配置`modules.d/elasticsearch-xpack.yml`

   ```yaml
   - module: elasticsearch
       metricsets:
         - ccr
         - cluster_stats
         - index
         - index_recovery
         - index_summary
         - ml_job
         - node_stats
         - shard
         - enrich
       period: 10s
       hosts: ["http://localhost:9200"]
       #username: "user"
       #password: "secret"
       xpack.enabled: true
   ```

5. 禁用system模块：
   `metricbeat modules disable system`

6. 输出到ES：

   ```yaml
   output.elasticsearch:
     # Array of hosts to connect to.
     hosts: ["http://es-mon-1:9200", "http://es-mon2:9200"] 
   
     # Optional protocol and basic auth credentials.
     #protocol: "https"
     #username: "elastic"
     #password: "changeme"
   ```

7. 启动Metricbeat

8. 禁用`xpack.monitoring.elasticsearch.collection.enabled` （因为功能重复了）

   ```http
   curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
   {
     "persistent": {
       "xpack.monitoring.elasticsearch.collection.enabled": false
     }
   }
   '
   ```

9. 在kibana中查看

## RESRful API

`curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'`

### cat API

cat API 主要用于命令行或者kibana控制台，对于应用程序，还是应该使用JSON API。以下命令可以查看所有的cat API。

```http
curl -X GET "localhost:9200/_cat"
=^.^=
/_cat/allocation
/_cat/shards
/_cat/shards/{index}
/_cat/master
/_cat/nodes
/_cat/tasks
/_cat/indices
/_cat/indices/{index}
/_cat/segments
/_cat/segments/{index}
/_cat/count
/_cat/count/{index}
/_cat/recovery
/_cat/recovery/{index}
/_cat/health
/_cat/pending_tasks
/_cat/aliases
/_cat/aliases/{alias}
/_cat/thread_pool
/_cat/thread_pool/{thread_pools}
/_cat/plugins
/_cat/fielddata
/_cat/fielddata/{fields}
/_cat/nodeattrs
/_cat/repositories
/_cat/snapshots/{repository}
/_cat/templates
```

可以指定help来查看帮助信息，第一列显示完整名称，第二列是缩写，第三列是参数说明

```http
curl -X GET "localhost:9200/_cat/nodes?help"
id                                 | id,nodeId                                   | unique node id
pid                                | p                                           | process id
ip                                 | i                                           | ip address
port                               | po                                          | bound transport port
http_address                       | http                                        | bound http address
version                            | v                                           | es version
flavor                             | f                                           | es distribution flavor
type                               | t                                           | es distribution type
build                              | b                                           | es build hash
jdk                                | j                                           | jdk version
disk.total                         | dt,diskTotal                                | total disk space
disk.used                          | du,diskUsed                                 | used disk space
disk.avail                         | d,da,disk,diskAvail                         | available disk space
disk.used_percent                  | dup,diskUsedPercent                         | used disk space percentage
heap.current                       | hc,heapCurrent                              | used heap
heap.percent                       | hp,heapPercent                              | used heap ratio
heap.max                           | hm,heapMax                                  | max configured heap
ram.current                        | rc,ramCurrent                               | used machine memory
ram.percent                        | rp,ramPercent                               | used machine memory ratio
ram.max                            | rm,ramMax                                   | total machine memory
file_desc.current                  | fdc,fileDescriptorCurrent                   | used file descriptors
file_desc.percent                  | fdp,fileDescriptorPercent                   | used file descriptor ratio
file_desc.max                      | fdm,fileDescriptorMax                       | max file descriptors
cpu                                | cpu                                         | recent cpu usage
load_1m                            | l                                           | 1m load avg
load_5m                            | l                                           | 5m load avg
load_15m                           | l                                           | 15m load avg
uptime                             | u                                           | node uptime
node.role                          | r,role,nodeRole                             | m:master eligible node, d:data node, i:ingest node, -:coordinating node only
master                             | m                                           | *:current master
name                               | n                                           | node name
```

然后通过`h`指定表头。下面是我的测试机器，可以通过rc rp rm看到，机器内存被用满了，堆内存hp只使用了51%，看到这里分析原因可能是lucene segment cache占用过多或者别的程序。然后看segment memory（sm）占用了403.8 MB也并不多，因此可以确定是别的程序占用了过多内容，事实上我在这个节点上还运行了logstash，占用了较多内存。

```http
curl -X GET "localhost:9200/_cat/nodes?v&h=ip,port,dt,du,dup,hc,hp,rc,rp,rm,cpu,l,sc,sm,fm"
ip            port    dt      du   dup    hc hp   rc rp     rm cpu    l   sc      sm    fm
172.19.64.254 9300 1.5tb 756.3gb 48.75 3.8gb 65 15gb 98 15.4gb  15 1.86 1746 403.8mb 9.7mb
```

