---
author: CC-2018
layout: post
title: Elasticsearch
date: 2018-03-20 20:51:01 +0800
categories: elasticsearch
tag: elasticsearch
---

* content
{:toc}

Elasticsearch 是一个实时的分布式搜索分析引擎，以近实时的方式快速存储、搜索和分析大数据。它通常被用作一项基础的技术，来应对复杂程序的搜索特性或需求。

### 基础概念

**近实时性(Near Realtime[NRT])**

Elasticsearch是一个近实时的搜索平台。这意味着从索引（即存储）一个文档，到可搜索状态，只有轻微的延时（通常为1秒）。

**集群(Cluster)**

一个集群是由一个或多个节点(服务器)组成的，这些节点一起联合提供存储、索引、搜索功能。一个集群有唯一的名称去标识，默认为elasticsearch，每个节点只有在 elasticsearch.yml 中使用 cluster.name 配置正确的集群名称，才能加入到相应的es集群。

有一点很重要，在不同的环境下要配置成不同的集群名称，以免线下的节点加入到线上的集群，如可分别将 logging-dev、logging-stage、logging-prod 来分别给开发，展示和生产集群命名。

**节点(Node)**

一个节点是一个单一的服务器，是你的集群的一部分，存储数据，并且参与集群的索引和搜索功能。节点在启动时也会被分配一个唯一的标识名称，这个名称默认是一个随机的UUID(Universally Unique IDentifier)，可以在 elasticsearch.yml 中将 node.name 配置成 `${HOSTNAME}`，以方便识别。通过配置 `cluster.name` 可加入特定的集群。

在一个集群中可以启动任意多的节点，而如果只有一个节点，将默认形成一个单节点集群。

**索引(Index)**

一个索引就是含有某些相似特性的文档的集合，比较类似关系数据库中的数据库。一个索引被一个名称(**必须都是小写**)唯一标识，并且这个名称被用于文档的索引、搜索、更新和删除操作。在一个集群中可以定义任意多的索引。

索引当动词时，表示存储一个文档，如"索引一个文档"表示存储一个文档到索引中。

倒排索引：关系型数据库通过增加一个 索引 比如一个 B树（B-tree）索引 到指定的列上，以便提升数据检索速度。Elasticsearch 和 Lucene 使用了一个叫做 **倒排索引** 的结构来达到相同的。默认的，一个文档中的每一个属性都是 被索引 的（有一个倒排索引）和可搜索的。一个没有倒排索引的属性是不能被搜索到的。

**类型(Type)**

Type 也可称之为Mapping types，有自己的field，元数据等。

一个类型是索引中的一个分类或者说是一个分区，它可以让你在同一索引中存储不同类型的文档，例如，为用户建一个类型，为博客文章建另一个类型，比较类似关系数据库中的表概念。现在已不可能在同一个索引中创建多个类型，并且整个类型的概念将会在未来的版本（7.0.0）中彻底移除。

警告！Type，实际上是_Type在6.0.0版本中已经不赞成使用

为什么移除type：在一个Elasticsearch的索引中，有相同名称字段的不同映射类型在Lucene内部是由同一个字段支持的。换言之，看下面的这个例子，user 类型中的 user_name字段和tweet类型中的user_name字段实际上是被存储在同一个字段中，而且两个user_name字段在这两种映射类型中都有相同的定义（如类型都是 text或者都是date）。这样在索引这个字段时会导致失败。并且最重要的是，在一个索引中存储那些有很少或没有相同字段的实体会导致稀疏数据，并且干扰Lucene有效压缩文档的能力。

移除type的替换方案：

```
1. 每个索引一个文档类型
数据更有可能是密集的，因此可以从Lucene中使用的压缩技术中获益。
在全文搜索中的为评分使用的单词统计更有可能是准确的，因为同一个索引中的所有文档都代表一个单一的实体类型。

2. 使用自定义type field字段：显式类型 type 字段取代隐式 _type 字段。
PUT twitter
{
  "mappings": {
    "user": {
      "properties": {
        "name": { "type": "text" },
        "user_name": { "type": "keyword" },
        "email": { "type": "keyword" }
      }
    },
    "tweet": {
      "properties": {
        "content": { "type": "text" },
        "user_name": { "type": "keyword" },
        "tweeted_at": { "type": "date" }
      }
    }
  }
}

PUT twitter/user/kimchy
{
  "name": "Shay Banon",
  "user_name": "kimchy",
  "email": "shay@kimchy.com"
}

PUT twitter/tweet/1
{
  "user_name": "kimchy",
  "tweeted_at": "2017-10-24T09:00:00Z",
  "content": "Types are going away"
}
```

**文档(Document)**

一个文档是一个可被索引（存储）的数据的基础单元。这个文档用JSON格式表现。在一个index/type中可以存储任意多的文档，一个文档物理上是属于index的，但是文档必须只定一个index里的type。

**分片和复制(Shards & Replicas)**

Elasticsearch 可以存储通过多节点存储 TB 或 PB 级别的数据，即 Elasticsearch 的分片能力。一个 *分片* 是一个底层的工作单元 ，它仅保存了全部数据中的一部分。索引实际上是指向一个或者多个物理 分片 的逻辑命名空间。一个集群包括多个节点，一个节点包括多个分片。当集群规模扩大或者缩小时， Elasticsearch 会自动的在各节点中迁移分片，使得数据仍然均匀分布在集群里。

每个索引可以被切分成多个分片，一个索引可以被复制零次(就是没有复制)或多次。一旦被复制，每个索引将会有一些主分片 Shards (就是那些最原始不是被复制出来的分片)，还有一些复制分片 Replicas (就是那些通过复制主分片得到的分片)。主分片和复制分片的数量可以在索引被创建时指定（post 中带 settings 参数）。索引被创建后，你可以随时动态修改复制分片的数量，但是不能修改主分片的数量（默认是5个主分片，1个复制分片）。

```
# 创建索引，并设置分片和副本数
PUT /new_index
{
    "settings": {
        "number_of_shards" :   1,
        "number_of_replicas" : 1
    }
}
```

每个Elasticsearch分片是一个Lucene索引。在一个Lucene索引中有一个文档数量的最大值。截至LUCENE-5843，这个限制是2,147,483,519 (= Integer.MAX_VALUE - 128)个文档。可以使用_cat/shards API监控分片大小。

**映射(Mapping)**

Mapping的过程，是定义文档包含的字段怎么存储，怎么被索引，比如：

```
那些 string 字段被当做全文检索字段
那些字段是数字、日期、或地理类型
字段格式：比如日期格式，或时间显示格式
使用动态模板去控制新增字段的设置
```

Mapping Type

Mapping Type，可以认为是一种自定义的类型，它包括 `Meta-fields` 和 `Fields or properties`

```
# doc 则为一个 mapping type
PUT my_index
{
  "mappings": {
    "doc": {
      "_default_": {
        "_all": {
          "enabled": false
        }
      },
      "properties": {
        "title":    { "type": "text"  },
        "name":     { "type": "text"  },
        "age":      { "type": "integer" },  
        "created":  {
          "type":   "date",
          "format": "strict_date_optional_time||epoch_millis"
        }
      }
    }
  }
}
```

元数据常查询、聚合分析、使用脚本和排序等，通常有以下元数据：

```
_index：文档所属的index
_type：文档所属的mapping type
_id：文档的id
_all：这个字段索引了所有其他字段的值；
_field_names：存储着文档中所有值为非空的字段信息
_meta：特定于应用程序的元数据。
_parent：用于创建两个映射的父子之间的关系；
_routing：自定义路由值，可以路由某个文档到具体的分片(shard)
_source：文档原始数据
_uid：由_type和_id组成
_timestamp：存储着当前文档的时间戳信息
```

`_default_ mapping`为创建新mapping type时的基类，在6.0.0已废弃



[参考文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/mapping.html)

### 安装

最新版的Elasticsearch为6.2.3版本，要求jdk 8以上，推荐jdk 1.8.0_131。

linux下Elasticsearch的安装比较简单：

```
# 下载
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.2.3.tar.gz
# 解压
tar -xvf elasticsearch-6.2.3.tar.gz
# 切换到bin目录：
cd elasticsearch-6.2.3/bin
# 启动
./elasticsearch
# 或者加参数启动
./elasticsearch -Ecluster.name=my_cluster_name -Enode.name=my_node_name
```

**重要配置**

在 config/elasticsearch.yml 中有以下重要配置需要修改

```
cluster.name：集群名称，决定加入到那个es集群
node.name：节点名称，为一标识集群内的各节点

# 路径，elasticsearch升级时可能会覆盖安装目录下的 data，logs，plugins等，最好是指定到其他路径
# 数据可以保存到多个不同的目录， 如果将每个目录分别挂载不同的硬盘，这可是一个简单且高效实现一个软磁盘阵列（ RAID 0 ）的办法。
# 但一个分片只会放到一个路径下
path.data: 可指定多个其他路径/path/to/data1,/path/to/data2
path.logs：log路径
path.conf: 配置文件路径
path.plugins: 安装插件路径

http.port: 8800 更改REST API 端口
transport.tcp.port: 9400 集群内部 tcp 通信端口
transport.tcp.compress: true 对TCP通信数据进行压缩

# 遇到网络故障的群集有可能将群集分成两个独立的群集（两个master脑裂）
# 设置没有足够 master 候选节点的时候，就不要选举 master
discovery.zen.minimum_master_nodes：(master 候选节点个数 / 2) + 1
node.master：决定此节点是否可做，默认为true
node.data：决定此节点是否可做数据节点存储数据，默认为true

# 使用单播代替组播，以防止节点无意中加入集群
discovery.zen.ping.multicast.enabled:false 禁用组播自动发现
# 配置尝试连接的节点代表，当联系到单播列表中的成员时，它就会得到整个集群所有节点的状态，然后它会联系 master 节点，并加入集群。
discovery.zen.ping.unicast.hosts:["host1", "host2:port"]

# 集群恢复
# 禁止 Elasticsearch 在存在至少 8 个节点（数据节点或者 master 节点）之前进行数据恢复
gateway.recover_after_nodes: 8
gateway.expected_nodes: 10
gateway.recover_after_time: 5m

# 锁定内存，保证jvm不swap
bootstrap.mlockall：true
bootstrap.memory_lock：true

# 创建索引时的默认分片数，也可以通过REST API 创建索引时再指定 settings
index.number_of_shards: 1
index.number_of_replicas: 1
```

以上配置可以通过 REST API 去动态修改配置，如：

```
PUT /_cluster/settings
{
    "persistent" : {
        "discovery.zen.minimum_master_nodes" : 2
    }
}
```

这样就启动了一个单节点的Elasticsearch集群，可以通过 REST API 完成以下功能：

```
检查集群，节点和索引的健康状况，状态和统计数据
管理集群，节点和索引的数据和原数据
执行CRUD(增删改查)操作，依靠索引进行搜索
执行高级搜索操作，比如分页，排序，过滤，脚本化，聚集等等
```
