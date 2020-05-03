---
title: ELK配置
date: 2020-05-02 23:27:33
categories:
  - Software
  - elk
tags:
  - Kibana
  - Elasticsearch
  - Chinese
---

```python
# @Time    : 2020-05-02
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

> 为了方便配置，（其实是懒得研究`Puppet`，`Chef`，和`Ansible`这些自动化管理工具），把一些`我常用`的配置项在这里列一下。所有内容只针对CentOS 7。ES版本 7.6。

<!-- more -->

## 集群设置

VMware 15 创建四个 CentOS 7虚拟机，IP都使用NAT。

| 节点         | IP            | 说明                         |
| ------------ | ------------- | ---------------------------- |
| Host Machine | 192.168.1.1   | Windows 10                   |
| node1        | 192.168.1.111 | 生产集群，运行ES，Kibana     |
| node2        | 192.168.1.112 | 生产集群，运行ES，metricbeat |
| node3        | 192.168.1.113 | 生产集群，运行ES，metricbeat |
| node4        | 192.168.1.114 | 监控集群，运行ES，Kibana     |

## ES

安装和预配置

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
# Must config！！！
sudo systemctl edit elasticsearch
# 添加
[Service]
LimitMEMLOCK=infinity
# 执行
sudo systemctl daemon-reload
-------------------------------------
# Not clear:
sysctl vm.max_map_count
ulimit -a
# 其中 -u 应该是4096， -n 应该是65535
vim /etc/security/limits.conf
# 文件添加
elasticsearch  -  nofile  65535
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

## Logstash

//TODO

## Beat

### Metricbeat

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

