---
title: ELK配置
categories:
  - Software
  - elk
tags:
  - Kibana
  - Elasticsearch
  - Chinese
abbrlink: d1e427b6
date: 2020-05-02 23:27:33
---

```python
# @Time    : 2020-05-02
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

> 为了方便配置，（其实是懒得研究`Puppet`，`Chef`，和`Ansible`这些自动化管理工具），把一些`我常用`的配置项在这里列一下。所有内容只针对CentOS 7。ES版本 7.6。

<!-- more -->

## 集群设置

VMware 15 创建四个 CentOS 7虚拟机（我只有16G内存，所以不使用GUI，设置了系统默认运行级别为`multi-user.target`，通过ssh进行操作），IP都使用NAT。

| 节点         | IP            | 说明                     |
| ------------ | ------------- | ------------------------ |
| Host Machine | 192.168.1.1   | Windows 10               |
| node1        | 192.168.1.111 | 生产集群，运行ES，Kibana |
| node2        | 192.168.1.112 | 生产集群，运行ES         |
| node3        | 192.168.1.113 | 生产集群，运行ES         |
| node4        | 192.168.1.114 | 监控集群，运行ES，Kibana |

## ES

安装和预配置，（shasum命令如果不存在，则`sudo yum install perl-Digest-SHA`）

```shell
mkdir -p ~/elk/downloads
cd ~/elk/downloads

wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.2-x86_64.rpm
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.2-x86_64.rpm.sha512
shasum -a 512 -c elasticsearch-7.6.2-x86_64.rpm.sha512 
sudo rpm --install elasticsearch-7.6.2-x86_64.rpm

sudo systemctl start elasticsearch.service
sudo systemctl status elasticsearch.service
sudo systemctl stop elasticsearch.service
```

安装完成之后会自动生成名为`elasticsearch`的用户和组，因为配置文件的目录`/etc/elasticsearch/`权限是`740`，同组的可读，所以可以把自己添加到组里，方便**查看**配置文件，修改还是需要当然`root`。

` sudo usermod -a -G elasticsearch yourusername`

`/etc/elasticsearch/elasticsearch.yml`:

```yaml
# cluster
cluster.name: elasticsearch-dev
cluster.remote.connect: false
# node
node.name: node1
node.master: true
node.voting_only: false
node.data: true
node.ingest: true
node.ml: false
xpack.ml.enabled: false
# path
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
# memory
bootstrap.memory_lock: true
# network
network.host: 0.0.0.0
http.port: 9200
# discovery
discovery.seed_hosts: ["192.168.1.111", "192.168.1.112:9300", "192.168.1.113"]
cluster.initial_master_nodes: ["node1", "node2", "node3"]
# gateway
gateway.expected_nodes: 3
gateway.expected_master_nodes: 3
# gateway.expected_data_nodes: 3
gateway.recover_after_time: 5m
gateway.recover_after_nodes: 3
# xpack monitoring
xpack.monitoring.enabled: true
xpack.monitoring.collection.enabled: true
```

`jvm.options`，默认为1 GB

```
-Xms512m
-Xmx512m
```

一些系统设置：可能也不需要设置

```
# 这个需要设置！！！
sudo systemctl edit elasticsearch
# 添加
[Service]
LimitMEMLOCK=infinity
# 执行
sudo systemctl daemon-reload
-------------------------------------
# 这些因人而异:
sysctl vm.max_map_count
ulimit -a
vim /etc/security/limits.conf
# 文件添加
elasticsearch  -  nofile  65535
# 下面这俩也是配置文件
vim /etc/sysconfig/elasticsearch
vim /usr/lib/systemd/system/elasticsearch.service
```

最终通过检查是否设置成功。

```shell
curl -X GET "192.168.1.113:9200/_nodes?filter_path=**.mlockall&pretty"
curl -X GET "192.168.1.113:9200/_nodes/stats/process?filter_path=**.max_file_descriptors&pretty"
```

## Kibana

安装

```shell
cd ~/elk/downloads
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.6.2-x86_64.rpm
shasum -a 512 kibana-7.6.2-x86_64.rpm 
sudo rpm --install kibana-7.6.2-x86_64.rpm

sudo systemctl start kibana.service
sudo systemctl stop kibana.service
```

`/etc/kibana/kibana.yml`

```yaml
server.port: 5601
server.host: 192.168.1.111
server.name: "Kibana-dev"

elasticsearch.hosts: ["http://192.168.1.111:9200", "http://192.168.1.112:9200", "http://192.168.1.113:9200"]
xpack.monitoring.enabled: true
xpack.monitoring.kibana.collection.enabled: true
```

至此，开启集群就可以正常运行，且集群内可以监控数据。

## Logstash

//TODO

## Beat

### Filebeat

安装

```shell
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.6.2-x86_64.rpm
sudo rpm -vi filebeat-7.6.2-x86_64.rpm
```

这篇文章写的挺好，我自己就不总结了。[FileBeat-Log 相关配置指南](https://www.cyhone.com/articles/usage-of-filebeat-log-config/)

`filebeat.yml`：

> input 和 output都可以设置pipeline，如果都设置会使用input中的pipeline，ES官方建议在input中，因为 `this option usually results in simpler configuration files`
>
> 通过ES的Ingest Node可以使用pipeline，在pipeline内定义了不同的processors。同时也可以直接在filebeat中定义processor，注意这两个地方的processors并不一样。前者的内容更丰富。

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
    #- c:\programdata\elasticsearch\logs\*
    # exclude_lines: ['^DBG']
    # exclude_files: ['.gz$']
  fields: 
    logtype: test2
    
- type: log
  enabled: true
  paths:
    - /var/another_log/*.log
    #- c:\programdata\elasticsearch\logs\*
    # exclude_lines: ['^DBG']
    # exclude_files: ['.gz$']
  fields: 
    logtype: test1
  # 以下几项都是默认值，根据需要修改
  encoding: plain
  ignore_older: "2h"
  close_inactive: "1m"
  close_renamed: false
  close_eof: false
  close_removed: true
  close_timeout: "0"
  harvester_limit: 0
  scan_frequency: "10s"
  # pipeline需要在集群内通过API设置
  pipeline: "my_pipeline"

setup.ilm.enabled: false
#setup.ilm.rollover_alias: "test-%{[agent.version]}"
#setup.ilm.pattern: "%{now/d}-000001"
#setup.ilm.policy_name: "my-policy-%{[agent.version]}"

output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]
  indices:
    - index: "filebeat-%{[agent.version]}-test1-%{+yyyy.MM.dd}"
      when.contains:
        fields.logtype: "test1"
    - index: "filebeat-%{[agent.version]}-test2-%{+yyyy.MM.dd}"
      when.contains:
        fields.logtype: "test2"
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
  
monitoring.enabled: true
```

写完一个pipeline，可以通过以下`_simulate` API进行测试，测试没问题之后写入集群，一例：

```http
POST /_ingest/pipeline/_simulate?pretty
{
  "pipeline": {
    "description": "_description",
    "processors": [
      {
        "csv": {
          "field": "message",
          "target_fields": [
            "NTPtime",
            "NTPts",
            "PTPtime",
            "PTPts",
            "databuffer_time",
            "databuffer_ts",
            "255_1",
            "255_2",
            "shot",
            "pre_trigger_delay_1",
            "pre_trigger_delay_2",
            "mr_bkt_1",
            "mr_bkt_2",
            "dr_bkt",
            "D8",
            "D9",
            "rf_phase_her",
            "rf_phase_ler",
            "c_mode",
            "n_bmode",
            "nn_bmode"
          ],
          "trim": true
        },
        "remove": {
          "field": "message"
        }
      }
    ]
  },
  "docs": [
    {
      "_index": "index",
      "_id": "id",
      "_source": {
        "message": "2020/04/04 00:01:53.369591,3668770913.369590,2020/04/04 00:02:30.363358,3668770950.363357,2020/04/04 00:01:53.370848,3668770913.370847,255,255,25026,0,0,245,230,0,0,50544,1038,36492,42,31,180,60032,1053"
      }
    },
    {
      "_index": "index",
      "_id": "id",
      "_source": {
        "message": "2020/04/04 00:01:53.429934,3668770913.429934,2020/04/04 00:02:30.423227,3668770950.423226,2020/04/04 00:01:53.430860,3668770913.430860,255,255,25029,0,0,686,117,0,0,50544,1038,36492,182,41,30,60032,1053"
      }
    }
  ]
}
```

### Metricbeat

> 因为发现ES和kibana默认的监控也大体够用（metricbeat提供的内容更丰富），以及自己机器内存有限，开俩集群耗电太高（在家办公时刻心疼电费）。所以下面的内容就没测试。

ES 推荐使用单独的集群和单独的`kibana`实例监控生产集群，事实上这也是非常必要的，因为一旦生产集群出现问题时，监控也可能会产生问题，那监控就没有意义了。

如果不想使用单独的集群，那直接配置`xpack.monitoring.collection.enabled`就可以。

以下的配置是用来介绍分开监控的情况，有两个集群：生产集群和监控集群，

在生产集群禁用默认的ES监控：

```http
PUT _cluster/settings
{
  "persistent": {
    "xpack.monitoring.elasticsearch.collection.enabled": false
  }
}
```

安装和配置

```shell
curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.6.2-x86_64.rpm
sudo rpm -vi metricbeat-7.6.2-x86_64.rpm
metricbeat modules enable elasticsearch-xpack
```

`/etc/metricbeat/metricbeat.yml`

```yaml
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
  index.codec: best_compression
setup.dashboards.enabled: true
setup.kibana:
  # 把 dashboard 导入监控集群的kibana实例
  host: "192.168.1.114:5601"
output.elasticsearch:
  # 此处指定为监控集群
  hosts: ["192.168.1.114:9200"]
```

`modules.d/elasticsearch-xpack.yml`

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
    hosts: ["http://192.168.1.111:9200"]
    #username: "user"
    #password: "secret"
    xpack.enabled: true
```

测试

```
curl -XGET 'http://192.168.1.114:9200/metricbeat-*/_search?pretty'
```

