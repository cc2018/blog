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
