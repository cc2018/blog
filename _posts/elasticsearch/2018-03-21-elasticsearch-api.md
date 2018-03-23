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

Elasticsearch 提供REST API。包括Cat APIs、Cluster APIs、Indices APIs、Document APIs、Search APIs 等。

### Cat APIs

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

### Indices APIs

索引 Api 可以用来管理各个索引、索引设置、索引别名（可映射多个索引）、mappings和索引模板等。

#### 管理索引

**创建索引**
创建索引api 允许同时初始化 index。

```
# put 协议，创建索引带设置，每个索引 3 个主分片，每个主分片 2 个副本
PUT twitter
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3,
            "number_of_replicas" : 2
        }
    }
}
# 或简约点
PUT twitter
{
    "settings" : {
        "number_of_shards" : 3,
        "number_of_replicas" : 2
    }
}

# 这些设置有的是create时指定的，有的是可以后面再动态修改的
# 静态设置 只能是创建时设置
index.number_of_shards
index.shard.check_on_startup
index.codec 压缩方式
index.routing_partition_size

# 动态设置 可以后期通过更新接口去设置
index.number_of_replicas 副本数
index.auto_expand_replicas
index.refresh_interval 多久更新刷新，刷新后就可搜索到最新的改变，默认为1s
index.max_result_window 搜索结果窗口
index.max_inner_result_window
index.max_rescore_window
index.max_docvalue_fields_search
index.max_script_fields 一次查询的最大 script_fields，默认为32
index.max_ngram_diff
index.max_shingle_diff
index.blocks.read_only  只读
index.blocks.read_only_allow_delete
index.blocks.read  
index.blocks.write
index.blocks.metadata
index.max_refresh_listeners
index.highlight.max_analyzed_offset
index.max_terms_count 默认65536

# 创建索引时带上 mapping
PUT twitter
{
    "settings" : {
        "number_of_shards" : 1
    },
    "mappings" : {
        "type1" : {
            "properties" : {
                "field1" : { "type" : "text" }
            }
        }
    }
}

# 创建索引时带上别名
PUT twitter
{
    "aliases" : {
        "alias_1" : {},
        "alias_2" : {
            "filter" : {
                "term" : {"user" : "kimchy" }
            },
            "routing" : "kimchy"
        }
    }
}

```

**删除索引**

用 delete 协议删除，后跟 index 或通配符即可删除，别名不能用来删除

```
DELETE /twitter
DELETE /_all
DELETE /*
```

为了禁用上面特别危险的 delete all 操作，可以设置 `action.destructive_requires_name=true`

**获取索引信息**

使用 GET Index 可以获取一个或多各 index 信息

```
GET /twitter
GET /_all
GET /*
```

**判断索引是否存在**

```
# 404 表示不存在，200表示存在
HEAD /twitter

```

**打开或关闭索引**

```
# 关闭索引，屏蔽对索引的读或写操作
POST /my_index/_close
POST /my_index/_open
```

**压缩索引**

将存在的一个索引迁移到一个新的更少分片的索引，但必须按倍数压缩

```
# 先将索引设为只读，并将分片都分配到同一个节点
PUT /my_source_index/_settings
{
  "settings": {
    "index.routing.allocation.require._name": "shrink_node_name", #重新分配分片
    "index.blocks.write": true #不允许写
  }
}

# 压缩索引到更少的分片数
POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 1,
    "index.number_of_shards": 1,
    "index.codec": "best_compression"
  },
  "aliases": {
    "my_search_indices": {}
  }
}
```

**拆分索引**

将存在的一个索引迁移到一个新的更多分片的索引，但必须按倍数拆分
```
# 设置为只读
PUT /my_source_index/_settings
{
  "settings": {
    "index.blocks.write": true
  }
}

POST my_source_index/_split/my_target_index
{
  "settings": {
    "index.number_of_shards": 5
  },
  "aliases": {
    "my_search_indices": {}
  }
}
```

**Rollover Index**

满足条件时，自动创建新索引，因为有别名logs_write，别名会自动定向到新的索引
```
PUT /logs-000001
{
  "aliases": {
    "logs_write": {}
  }
}

# Add > 1000 documents to logs-000001，create logs-000002
# 条件是或的关系
POST /logs_write/_rollover
{
  "conditions": {
    "max_age":   "7d",
    "max_docs":  1000,
    "max_size":  "5gb"
  }
}
```

#### 管理 mapping

**Put Mapping**

PUT mapping API 可以对存在的索引添加 index，或者对存在的字段(fields)改变搜索设置。

```
# 创建索引
PUT twitter
{}


# 创建mapping，对_doc添加 email 属性
PUT twitter/_mapping/_doc
{
  "properties": {
    "email": {
      "type": "keyword"
    }
  }
}

# 多索引添加user_name
PUT /twitter-1,twitter-2/_mapping/_doc
{
  "properties": {
    "user_name": {
      "type": "text"
    }
  }
}
```

如果要更新mapping结构的话，可以使用post，比如对动态模板更新。

**通常不能对已存在的字段(field)进行更新**

**Get Mapping**

获取 Mapping

```
# 获取 twitter 索引下 _doc type的 mapping
GET /twitter/_mapping/_doc
GET /_mapping/_doc
# 获取所有索引的 mapping
GET /_all/_mapping/_doc
GET /_all/_mapping
GET /_mapping
# 获取字段 title 的 mapping，可以同时获取多字段
GET publications/_mapping/_doc/field/title
```

#### 创建别名

可以给索引创建别名，甚至可以多个索引对应一个别名。别名可以应用于搜索、路由（别名会自动展开对应多个索引）等。
```
# 创建索引
POST /_aliases
{
    "actions" : [
        { "add" : { "index" : "test1", "alias" : "alias1" } }
    ]
}
# 删除索引
POST /_aliases
{
    "actions" : [
        { "remove" : { "index" : "test1", "alias" : "alias1" } }
    ]
}
```

#### 索引设置

**更新索引设置**

事实更新索引层面的设置

```
# 更新副本数
PUT /twitter/_settings
{
    "index" : {
        "number_of_replicas" : 2
    }
}
# 使用null更新到默认值
PUT /twitter/_settings
{
    "index" : {
        "refresh_interval" : null
    }
}

# 系列操作更新索引的分析器
POST /twitter/_close
PUT /twitter/_settings
{
  "analysis" : {
    "analyzer":{
      "content":{
        "type":"custom",
        "tokenizer":"whitespace"
      }
    }
  }
}
POST /twitter/_open
```

**获取索引设置**

```
GET /twitter/_settings
GET /log_2013_-*/_settings/index.number_*
```

**索引模板**

当创建索引时，可以使用索引模板，包括设置和mappings。索引模板只有在索引创建时才生效，对存在的索引没有任何作用。

```
# 创建一个template_1，此模板会应用到 以te开头或bar开头的模板创建时
PUT _template/template_1
{
  "index_patterns": ["te*", "bar*"],
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
    "type1": {
      "_source": {
        "enabled": false
      },
      "properties": {
        "host_name": {
          "type": "keyword"
        },
        "created_at": {
          "type": "date",
          "format": "EEE MMM dd HH:mm:ss Z YYYY"
        }
      }
    }
  },
  "version": 123
}

# 删除模板
DELETE /_template/template_1

# 获取模板
GET /_template/template_1
GET /_template/temp*
GET /_template/template_1,template_2

# 获取所有模板
GET /_template
```

**Analyze**
```
# 获取 es 对词拆分结果
GET _analyze
{
  "analyzer" : "standard",
  "text" : "this is a test"
}

GET _analyze
{
  "tokenizer" : "keyword",
  "filter" : ["lowercase"],
  "char_filter" : ["html_strip"],
  "text" : "this is a <b>test</b>"
}
```

#### Monitoring

获取索引状态数据

```
# 返回所有索引的状态
GET /_stats
# 返回多索引的状态，默认返回所有的数据
GET /index1,index2/_stats
# url中设置要返回的数据，字段有
# docs|store|indexing|get|search|segments|completion
# fielddata|flush|merge|request_cache|refresh|warmer|translog|fields
GET /my_index/_stats/indexing?types=type1,type2
GET twitter/_stats?filter_path=**.commit&level=shards
```

获取索引分片信息

```
GET /_segments
GET /test1,test2/_segments
GET /test/_segments?verbose=true
```

获取索引恢复状态

```
GET index1,index2/_recovery?human
GET _recovery?human&detailed=true
```

获取索引分片存储

主副分片存储在那个节点上
```
# 返回test索引的分片信息
GET /test/_shard_stores

# 返回test1，test2索引的分片信息
GET /test1,test2/_shard_stores

# 返回所有索引的分片信息
GET /_shard_stores
```

#### 状态管理

**清除缓存**

```
POST /twitter/_cache/clear
POST /kimchy,elasticsearch/_cache/clear
# 清除所有索引的缓存
POST /_cache/clear
```

**Flush**

清除缓存数据到物理存储上
```
POST twitter/_flush
POST kimchy,elasticsearch/_flush
POST _flush
```

**Synced Flush**

```
POST twitter/_flush/synced
POST kimchy,elasticsearch/_flush/synced
POST _flush/synced
```

**刷新(Refresh)**

使最新的更新能被搜索到
```
POST /twitter/_refresh
POST /kimchy,elasticsearch/_refresh
POST /_refresh
```

**Force Merge**
```
POST /twitter/_forcemerge
POST /kimchy,elasticsearch/_forcemerge
POST /_forcemerge
```

### Document APIs

文档 CRUD APIs

**单文档APIs**

**多文档APIs**
