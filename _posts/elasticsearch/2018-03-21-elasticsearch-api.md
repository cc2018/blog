---
author: CC-2018
layout: post
title: elasticsearch-api
date: 2018-03-21 14:40:01 +0800
categories: elasticsearch
tag: elasticsearch
---

* content
{:toc}

Elasticsearch 提供REST API。包括cat APIs、Cluster APIs、Indices APIs、Document APIs、Search APIs 等。

### cat APIs

Elasticsearch 提供的很多 REST API 都是 json 格式的，不是很方便查询。cat APIs 模仿 linux 的 cat 命令设计，输出以表格的形式提供，方便直观。访问：`http:://localhost:9200/_cat` 请求可以列出所有的api：

```
GET /_cat

=^.^=
/_cat/allocation
/_cat/shards
/_cat/shards/{index}
/_cat/master
/_cat/nodes
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
/_cat/plugins
/_cat/fielddata
/_cat/fielddata/{fields}
```

每一个cat Api 都可接受一个 v 参数来显示表头。

先看健康检查：

```
GET /_cat/health?v

epoch   time    cluster status node.total node.data shards pri relo init
1408[..] 12[..] el[..]  1         1         114 114    0    0     114
unassign pending_tasks
```

再看看 cat API 里面的 节点统计 ：

```
GET /_cat/nodes?v

host         ip            heap.percent ram.percent load node.role master name
zacharys-air 192.168.1.131           45          72 1.85 d         *      Zach
```

如果不清楚字段的意思，可以使用help参数，查看每个字段意思，如：

```
GET /_cat/nodes?help

id               | id,nodeId               | unique node id
pid              | p                       | process id
host             | h                       | host name
ip               | i                       | ip address
port             | po                      | bound transport port
version          | v                       | es version
build            | b                       | es build hash
jdk              | j                       | jdk version
disk.avail       | d,disk,diskAvail        | available disk space
heap.percent     | hp,heapPercent          | used heap ratio
heap.max         | hm,heapMax              | max configured heap
ram.percent      | rp,ramPercent           | used machine memory ratio
ram.max          | rm,ramMax               | total machine memory
load             | l                       | most recent load avg
uptime           | u                       | node uptime
node.role        | r,role,dc,nodeRole      | d:data node, c:client node
master           | m                       | m:master-eligible, *:current master
...
```

第一列显示完整的名称，第二列显示缩写，第三列提供了关于这个参数的简介。可以用 ?h 参数来明确指定显示这些指标：
```
GET /_cat/nodes?v&h=ip,port,heapPercent,heapMax

ip            port heapPercent heapMax
192.168.1.131 9300          53 990.7mb
```

另外如果是在linux 命令行下，可以很方便的结合管道使用 cat APIs。如：

```
% curl 'localhost:9200/_cat/indices?bytes=b' | sort -rnk8 | grep -v marvel

yellow test_names         5 1 3476004 0 376324705 376324705
yellow wavelet            5 1    5979 0  54815185  54815185
yellow kibana-int         5 1       2 0     17791     17791
yellow t                  5 1       7 0     15280     15280
yellow website            5 1      12 0     12631     12631
yellow agg_analysis       5 1       5 0      5804      5804
yellow v2                 5 1       2 0      5410      5410
yellow v1                 5 1       2 0      5367      5367
yellow bank               1 1      16 0      4303      4303
yellow v                  5 1       1 0      2954      2954
yellow p                  5 1       2 0      2939      2939
yellow b0001_072320141238 5 1       1 0      2923      2923
yellow ipaddr             5 1       1 0      2917      2917
yellow v2a                5 1       1 0      2895      2895
yellow movies             5 1       1 0      2738      2738
yellow cars               5 1       0 0      1249      1249
yellow wavelet2           5 1       0 0       615       615

```

### Cluster APIs

#### 集群状态

**集群健康**

集群有3种健康状态

```
green：所有的主分片和副本分片都已分配。你的集群是 100% 可用的。
yellow：所有的主分片已经分片了，但至少还有一个副本是缺失的。
red：至少一个主分片（以及它的全部副本）都在缺失中。
```

而当集群状态有问题时，可以参考此文档排查，[集群状态](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_cluster_health.html)

```
# 获取集群健康状态，提供参数：level、wait_for_status、wait_for_nodes、wait_for_no_relocating_shards
# wait_for_no_initializing_shards、wait_for_active_shards、local、timeout 等
GET _cluster/health
# 从索引层面上查看集群状态
GET _cluster/health?level=indices
# 打印出分片的状态
GET /_cluster/health?level=shards
# 获取具体索引的健康状态
GET /_cluster/health/test1,test2
# 同步等待状态变成 yellow
GET /_cluster/health?wait_for_status=yellow&timeout=50s
```

**集群状态**

cluster state API 可以获取更全面的集群状态信息。`GET /_cluster/state` 它会返回集群名称，节点，metadata，索引信息等。

可以对结果加参数过滤。过滤格式为：`GET /_cluster/state/{metrics}/{indices}`，metrics 有如下value：

```
version：集群状态版本
master_node：选举的 master 节点
nodes：节点信息
routing_table：路由
metadata：元数据
blocks：blocks

GET /_cluster/state/metadata,routing_table/foo,bar
GET /_cluster/state/_all/foo,bar
GET /_cluster/state/blocks
```
另外还有一些常见的 Cluster APIs
```
# 更新集群设置信息，persistent：集群重启后也可以生效，transient：短暂生效，集群重启失效
PUT /_cluster/settings
{
    "persistent" : {
        "indices.recovery.max_bytes_per_sec" : "50mb"
    }
}
或
PUT /_cluster/settings?flat_settings=true
{
    "transient" : {
        "indices.recovery.max_bytes_per_sec" : "20mb"
    }
}
# 获取集群数字统计信息，包括分片数、文档数、节点数等等。
GET /_cluster/stats?human&pretty
# 获取集群的一些还未执行的任务，如创建索引，更新mapping，分片或失败分片等，通常会返回为空
GET /_cluster/pending_tasks
# 分片移动，将分片从一个节点移动到另一个节点
POST /_cluster/reroute
{
    "commands" : [
        {
            "move" : {
                "index" : "test", "shard" : 0,
                "from_node" : "node1", "to_node" : "node2"
            }
        },
        {
          "allocate_replica" : {
                "index" : "test", "shard" : 1,
                "node" : "node3"
          }
        }
    ]
}
```

#### 节点状态

大部分的 cluster APIs，都可以指定node执行（比如获取一个节点的状态）

```
# Local
GET /_nodes/_local
# Address
GET /_nodes/10.0.0.3,10.0.0.4
GET /_nodes/10.0.0.*
# Names，可以看到该节点的详细信息，包括节点配置，端口，操作系统，插件等。
GET /_nodes/node_name_goes_here
GET /_nodes/node_name_goes_*
# Attributes (set something like node.attr.rack: 2 in the config)
GET /_nodes/rack:2
GET /_nodes/ra*:2
GET /_nodes/ra*:2*
```

**节点信息统计**
```
GET /_nodes/stats
GET /_nodes/nodeId1,nodeId2/stats
# 只查看索引
GET /_nodes/stats/indices
# 只查看操作系统信息和进程信息
GET /_nodes/stats/os,process
# 只返回某个节点的进程信息
GET /_nodes/10.0.0.1/stats/process

GET _nodes/usage
GET _nodes/nodeId1,nodeId2/usage
```

**管理任务API**

`The Task Management API` 是比较新的功能，api也一直在修改，并不对旧接口兼容。

```
GET _tasks
GET _tasks?nodes=nodeId1,nodeId2
GET _tasks?nodes=nodeId1,nodeId2&actions=cluster:*
```

**热门线程**

获取各个节点的线程信息

```
GET /_nodes/hot_threads
GET /_nodes/{nodesIds}/hot_threads
```

**分配分片理由**

用来解释分片没有被分配的理由，或者分配到某个节点的理由。

```
GET /_cluster/allocation/explain
{
  "index": "myindex",
  "shard": 0,
  "primary": true
}
或 primary 表示是不是要解释副分片
GET /_cluster/allocation/explain
{
  "index": "myindex",
  "shard": 0,
  "primary": false,
  "current_node": "nodeA"                         
}
```
