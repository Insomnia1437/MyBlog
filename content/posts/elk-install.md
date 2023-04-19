---
title: Elasticsearch+Kibana+Beats+Logstash安装
categories:
  - Software
  - elk
tags:
  - Kibana
  - Elasticsearch
  - Chinese
abbrlink: 5410d8cb
date: 2020-01-28 11:15:08
---

```python
# @Time    : 2019-01-28
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```


本博客资料来源：[Elastic官网](https://www.elastic.co/)

Elasticsearch+Logstash+Kibana，合称ELK，Elasticsearch用来完成搜索，Logstash用于数据采集，Kibana用于数据可视化。
数据采集工作早期由Logstash完成，现在则使用Beats，Logstash更多用于数据汇聚处理。
<!-- more -->

## Elastic Stack学习笔记

### 概述

**Phronesis!!!**

实践是最好的理解，先安装然后慢慢深入了解。

- Elasticsearch
- Kibana
- Beats
- Logstash (可选安装)

最新版本为7.5，可以在此查看不同版本对不同的系统的支持：[版本以及系统支持](https://www.elastic.co/support/matrix)。

安装Java的过程省略，JVM的版本支持：[可用Java版本](https://www.elastic.co/support/matrix#matrix_jvm)。

我使用CentOS 7以及openJDK 1.8 作为测试，测试过程所有软件都安装在一台服务器上。

> 关于不同Linux发行版本启动系统服务的命令，使用`ps -p 1`命令查看你的 SysV，如果是`init`，请使用
>
> ```sh
> sudo -i service elasticsearch start
> sudo -i service elasticsearch stop
> ```
>
> 如果是`systemd`，请使用
>
> ```shell
> sudo systemctl start elasticsearch.service
> sudo systemctl stop elasticsearch.service
> ```



#### Elasticsearch 安装

```shell
$ curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.5.2-x86_64.rpm

$ sudo rpm -i elasticsearch-7.5.2-x86_64.rpm
warning: elasticsearch-7.5.2-x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID d88e42b4: NOKEY
Creating elasticsearch group... OK
Creating elasticsearch user... OK
### NOT starting on installation, please execute the following statements to configure elasticsearch service to start automatically using systemd
 sudo systemctl daemon-reload
 sudo systemctl enable elasticsearch.service
### You can start elasticsearch service by executing
 sudo systemctl start elasticsearch.service
Created elasticsearch keystore in /etc/elasticsearch

$ sudo systemctl start elasticsearch

```

如果服务启动成功，在浏览器中输入`http://localhost:9200/`或者执行` curl http://127.0.0.1:9200`，能看到如下Json返回值则表示成功。
```json
{
  "name" : "wang.linac.kek.jp",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "BSIeBQ36Trqdyj4ve6O3xQ",
  "version" : {
    "number" : "7.5.2",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "8bec50e1e0ad29dad5653712cf3bb580cd1afcdf",
    "build_date" : "2020-01-15T12:11:52.313576Z",
    "build_snapshot" : false,
    "lucene_version" : "8.3.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

#### 安装Kibana

```shell
$ curl -L -O https://artifacts.elastic.co/downloads/kibana/kibana-7.5.2-x86_64.rpm
$ sudo rpm --install kibana-7.5.2-x86_64.rpm
$ sudo service start elasticsearch
```
如需关闭，kill 其pid即可。

在浏览器中访问`http://127.0.0.1:5601`就能看到Kibana。

#### 安装Beats

Beats可以直接传输数据到Elasticsearch或者通过Logstash。不同的Beats支持不同的数据，此处我们安装Metricbeat，使用其`system` module去收集系统级数据，如CPU使用量，内存，文件系统，磁盘IO，网络等。

| Elastic Beasts | 捕获数据                |
| -------------- | ----------------------- |
| Auditbeat      | audit data              |
| Filebeat       | log files               |
| Functionbeat   | cloud data              |
| Heartbeat      | availability monitoring |
| Journalbeat    | systemd journals        |
| Metricbeat     | metrics                 |
| Packetbeat     | network traffic         |
| Winlogbeat     | windows event logs      |

下载安装

```shell
$ curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.5.2-x86_64.rpm
$ sudo rpm -vi metricbeat-7.5.2-x86_64.rpm
```

启动`system` module之前请确保Elasticsearch和Kibana处于运行状态。执行命令

```shell
$ sudo metricbeat modules enable system
Module system is already enabled
$ sudo metricbeat setup -e
...省略一大堆
2020-01-28T13:05:32.204+0900    INFO    instance/beat.go:780    Kibana dashboards successfully loaded.
Loaded dashboards
$ sudo systemctl start metricbeat
```

然后打开浏览器，进入` http://localhost:5601/app/kibana#/dashboard/Metricbeat-system-overview-ecs` 或者点击左侧的`dashboard`，找到system overview，点击`Host Overview`就能看到系统数据的图表显示了。

#### 安装Logstash

```sh
curl -L -O https://artifacts.elastic.co/downloads/logstash/logstash-7.5.2.rpm
sudo rpm -i logstash-7.5.2.rpm
```

配置Logstash监听beats或者直接读取日志，并添加filter，之后输出到Elasticsearch。