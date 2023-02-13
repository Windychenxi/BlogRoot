---
title: ELK 专题七 （ElasticSearch 优化）
top_img: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/tree-g97833b6a8_1280.jpg
cover: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/tree-g97833b6a8_1280.jpg
tag: 
 - ElasticSearch
 - ELK
categories: 
 - ElasticSearch
---

# 前言

对 ElasticSearch 进行合理的优化，提高 ElasticSearch 集群的性能。

# 推荐阅读

- [ELK专题一 IK 分词器源码升级改造实现热更新机制](https://windychenxi.github.io/2023/02/12/ELK/IK%E5%88%86%E8%AF%8D%E5%99%A8%E6%BA%90%E7%A0%81%E5%8D%87%E7%BA%A7%E6%94%B9%E9%80%A0%E5%AE%9E%E7%8E%B0%E7%83%AD%E6%9B%B4%E6%96%B0%E6%9C%BA%E5%88%B6/)
- [ELK专题二 FileBeat 日志收集](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86/)
- [ELK专题三 LogStash 数据清洗与被压机制](https://windychenxi.github.io/2023/02/12/ELK/LogStash%20%E6%95%B0%E6%8D%AE%E6%B8%85%E6%B4%97%E4%B8%8E%E8%83%8C%E5%8E%8B%E6%9C%BA%E5%88%B6/)
- [ELK专题四 FileBeat + LogStash 整合](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20+%20LogStash%20%E6%95%B4%E5%90%88/)
- [ELK专题五 Google 浏览器插件 ElasticSeach-head 安装](https://windychenxi.github.io/2023/02/12/ELK/Google%E6%B5%8F%E8%A7%88%E5%99%A8%E6%8F%92%E4%BB%B6ElasticSearch-head%E5%AE%89%E8%A3%85/)
- [ELK专题六 ELK + FileBeat 整合](https://windychenxi.github.io/2023/02/12/ELK/ELK%20+%20FileBeat%20%E6%95%B4%E5%90%88/)
- [ELK专题七 ElasticSearch 优化](https://windychenxi.github.io/2023/02/12/ELK/ElasticSearch%20%E4%BC%98%E5%8C%96/)

# 分片策略

## 合理设置分片数

ElasticSearch 在 7.0 之前，ElasticSearch 默认 5 个主分片、1 个备份分片；在7.0 之后，默认一个主分片、1 个备份分片。

分片和副本的设计为 ES 提供了支持分布式和故障转移的特性，但并不意味着分片和副本是可以无限分配的。而且索引的分片完成分配后，由于索引的路由机制，我们是不能重新修改分片数的。

可能有人会说，我不知道这个索引将来会变得很大，并且过后我也不能更改索引的大小，所以为了保险起见，还是给它设置 1000 个分片吧。但是需要知道的是，一个分片并不是没有代价的。

需要了解如下几个问题：

1. 每个搜索请求都需要命中索引中的每一个分片，如果每一个分片都处于不同的节点还好，但是如果多个分片都需要在同一个节点上竞争使用相同的资源就有些糟糕了。
2. 用于计算相关度的词项统计信息是基于分片的。如果有许多分片，每一个都只有很少的数据，会导致很低的相关度。

## 推迟分片分配

对于节点瞬时中断的问题，默认情况，集群会等一分钟来查看节点是否会重新加入。如果这个节点在此期间重新加入，重新加入的节点会保持现有的分片数据，不会出发新的分片分配。这样就可以减少 ES 在自动再平衡可用分片时所带来的极大开销。

通过修改参数 `delayed_timeout`，可以延长再均衡的时间。可以全局设置，也可以在索引级别进行修改。

```yaml
PUT /_all/_settings
{
    "settings": {
        "index.unassigned.node_left.delayed_timeout": "5m"
    }
}
```

## 批量提交数据

ES 提供 Bulk API 支持批量操作，当我们有大量的写任务时，可以使用 Bulk 来进行批量写入。

通用的策略如下：

Bulk 默认设置批量提交的数据量不能超过 100M，数据条数一般是根据文档的大小和服务器性能而定的。但是单次批处理的数据大小应从 5MB~15MB 逐渐增加。当性能没有提升时，把这个数据量作为最大值。

## 优化存储设备

ES 是一种密度使用磁盘的应用，在段合并时会频繁操作磁盘，所以对磁盘要求较高，可以使用 SSD。当磁盘速度提升之后，集群的整体性能会大幅提高。

## 减少 Refresh 次数

Lucene 在新增数据时，采用了延迟写入的策略。默认情况下索引的 refresh_interval 为 1秒。

Lucene 将待写入的数据先写到内存中，超过 1 秒（默认）时，就会触发一次 Refresh。然后 Refresh 会把内存中的数据刷新到操作系统的文件缓存系统中。

如果我们对搜索的时效性要求不高，可以将 Refresh 周期延长。例如 30 秒，这样还可以有效地减少段刷新次数。但这同时也意味着需要消耗更多的 Heap 内存。

![image-20230212224007228](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/image-20230212224007228.png)

> **注意**
>
> 1. `Memory` 阶段，不能被 ElasticSearch 检索
>
> 2. `OS Cache` 阶段，可以被 ElasticSearch 检索
>
> 3. **内存数据会不会丢失？**
>
>    不会。
>
>    （1） 在`Memory`阶段，存在一主一备。
>
>    （2） 同时在数据写入 `Memory`时，还会写入 `Translog`中，如果存在数据丢失，可以从 `Translog`进行恢复。

## 加大 Flush 设置

Flush 的主要目的是把文件缓存系统中的段持久化到硬盘。当 Translog 的数据量达到 512MB 或者 30 分钟时，会触发一次 Flush。

`index.translog.flush_threshold_size` 参数的默认值是 512MB， 我们可以进行修改。增加参数值意味着文件缓存系统中可能需要存储更多的数据。所以我们需要为操作系统的文件缓存系统留下足够的空间。

## 减少副本的数量

ES 为了保证集群的可用性，提供了 Replicas（副本）支持。然而每个副本也会执行分析、索引及可能得合并过程。所以 Replicas 的数量会严重影响写索引的效率。

当写索引时，需要把写入的数据都同步到副本节点，副本节点越多，写索引的效率就越慢。

如果我们需要大批量进行写入操作，可以先禁止 Replica 的复制。设置 `index.number_of_replicas: 0`关闭副本。在写入完成后，Replica 修改回正常的状态。

## 路由选择

当我们查询文档的时候，Elasticsearch 如何知道一个文档应该存放到哪个分片中呢？它其实是

通过下面这个公式来计算出来：

`shard = hash(routing) % number_of_primary_shardsrouting` 默认值是文档的 id，也可以采用自定义值，比如用户 id。

- **不带 routing 查询**

在查询的时候因为不知道要查询的数据具体在哪个分片上，所以整个过程分为 2 个步骤：

1、分发：请求到达协调节点后，协调节点将查询请求分发到每个分片上。

2、聚合: 协调节点搜集到每个分片上查询结果，在将查询的结果进行排序，之后给用户返回结果。

- **带 routing 查询**

查询的时候，可以直接根据 routing 信息定位到某个分配查询，不需要查询所有的分配，经过协调节点排序。向上面自定义的用户查询，如果 routing 设置为 userid 的话，就可以直接查询出数据来，效率提升很多。

# ES核心配置参数

ES 核心配置文件`elasticsearch.yml`重要参数

| 参数名                             | 参数值        | 说明                                                         |
| ---------------------------------- | ------------- | ------------------------------------------------------------ |
| cluster.name                       | elasticsearch | 配置 ES 的集群名称，默认值是 ES，建议改成与所存数据相关的名称，ES 会自动发现在同一网段下的集群名称相同的节点 |
| node.name                          | node-1        | 集群中的节点名，在同一个集群中不能重复。节点的名称一旦设置，就不能再改变了。当然，也可以设 置 成 服 务 器 的 主 机 名 称 ， 例 如 node.name:${HOSTNAME}。 |
| node.master                        | true          | 指定该节点是否有资格被选举成为 Master 节点，默认是 True，如果被设置为 True，则只是有资格成为Master 节点，具体能否成为 Master 节点，需要通过选举产生。 |
| node.data                          | true          | 指定该节点是否存储索引数据，默认为 True。数据的增、删、改、查都是在 Data 节点完成的。 |
| index.number_of_shards             | 1             | 设置都索引分片个数，默认是 1 片。也可以在创建索引时设置该值，具体设置为多大都值要根据数据量的大小来定。如果数据量不大，则设置成 1 时效率最高 |
| index.number_of_replicas           | 1             | 设置默认的索引副本个数，默认为 1 个。副本数越多，集群的可用性越好，但是写索引时需要同步的数据越多。 |
| transport.tcp.compress             | true          | 设置在节点间传输数据时是否压缩，默认为 False，不压缩         |
| discovery.zen.minimum_master_nodes | 1             | 设置在选举 Master 节点时需要参与的最少的候选主节点数，默认为 1。如果使用默认值，则当网络不稳定时有可能会出现脑裂。合理的数值为 (master_eligible_nodes/2)+1 ，其中master_eligible_nodes 表示集群中的候选主节点数 |
| discovery.zen.ping.timeout         | 3s            | 设置在集群中自动发现其他节点时 Ping 连接的超时时间，默认为 3 秒。在较差的网络环境下需要设置得大一点，防止因误判该节点的存活状态而导致分片的转移 |



# 问题及解决方案

## 为什么要使用 ES?

系统中的数据，随着业务的发展，时间的推移，将会非常多，而业务中往往采用模糊查询进行数据的搜索，而模糊查询会导致查询引擎放弃索引，导致系统查询数据时都是全表扫描，在百万级别的数据库中，

查询效率是非常低下的，而我们使用 ES 做一个全文索引，将经常查询的系统功能的某些字段，比如说电商系统的商品表中商品名，描述、价格还有 id 这些字段我们放入 ES 索引库里，可以提高查询速度。



## master 选举流程

1. Elasticsearch 的选主是 ZenDiscovery 模块负责的，主要包含 Ping（节点之间通过这个 RPC 来发现彼此）和 Unicast（单播模块包含一个主机列表以控制哪些节点需要 ping 通）这两部分

2. 对所有可以成为 master 的节点（node.master: true）根据 nodeId 字典排序，每次选举每个节点都把自己所知道节点排一次序，然后选出第一个（第 0 位）节点，暂且认为它是 master 节点。

3. 如果对某个节点的投票数达到一定的值（可以成为 master 节点数 n/2+1）并且该节点自己也选举自己，那这个节点就是 master。否则重新选举一直到满足上述条件。

4. master 节点的职责主要包括集群、节点和索引的管理，不负责文档级别的管理；



## 集群脑裂问题

### 成因

1. **网络问题**：集群间的网络延迟导致一些节点访问不到 master，认为 master 挂掉了从而选举出新的master，并对 master 上的分片和副本标红，分配新的主分片

2. **节点负载**：主节点的角色既为 master 又为 data，访问量较大时可能会导致 ES 停止响应造成大面积延迟，此时其他节点得不到主节点的响应认为主节点挂掉了，会重新选取主节点。

3. **内存回收**：data 节点上的 ES 进程占用的内存较大，引发 JVM 的大规模内存回收，造成 ES 进程失去响应。



### 解决方案

1. **减少误判：**discovery.zen.ping_timeout 节点状态的响应时间，默认为 3s，可以适当调大，如果 master在该响应时间的范围内没有做出响应应答，判断该节点已经挂掉了。

调大参数（如 6s，discovery.zen.ping_timeout:6），可适当减少误判。

2. **选举触发**: `discovery.zen.minimum_master_nodes:1` 

   该参数是用于控制选举行为发生的最小集群主节点数量。当备选主节点的个数大于等于该参数的值，且备选主节点中有该参数个节点认为主节点挂了，进行选举。官方建议为（n/2）+1，n 为主节点个数(即有资格成为主节点的节点个数）

3. **角色分离**：即 master 节点与 data 节点分离，限制角色

（1） 主节点配置为：node.master: true node.data: false

（2） 从节点配置为：node.master: false node.data: true

## 更新删除文档

1. 删除和更新也都是写操作，但是 Elasticsearch 中的文档是不可变的，因此不能被删除或者改动以展示其变更；

2. 磁盘上的每个段都有一个相应的.del 文件。当删除请求发送后，文档并没有真的被删除，而是在.del文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在.del 文件中被标记为删除的文档将不会被写入新段。

3. 在新的文档被创建时，Elasticsearch 会为该文档指定一个版本号，当执行更新时，旧版本的文档在.del文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结果中被过滤掉.

## 倒排索引

​    倒排索引是搜索引擎的核心。搜索引擎的主要目标是在查找发生搜索条件的文档时提供快速搜索。ES中的倒排索引其实就是 lucene 的倒排索引，区别于传统的正向索引，倒排索引会在存储数据时将关键词和 数据进行关联，保存到倒排表中，然后查询时，将查询内容进行分词后在倒排表中进行查询，最后匹配数据即可。

## 文档写入原理

​    ![0](https://note.youdao.com/yws/public/resource/06b6f48ef4e56cb429410a1f35176f14/xmlnote/WEBRESOURCE842c63d365ea11f99fec0334262f1bce/4891)

1. 选择任意一个DataNode发送请求，例如：node2。此时，node2就成为一个coordinating node（协调节点）

2. 计算得到文档要写入的分片

   `shard = hash(routing) % number_of_primary_shards`

   routing 是一个可变值，默认是文档的 _id

3. coordinating node会进行路由，将请求转发给对应的primary shard所在的DataNode（假设primary shard在node1、replica shard在node2）

4. node1节点上的Primary Shard处理请求，写入数据到索引库中，并将数据同步到Replica shard

5. Primary Shard和Replica Shard都保存好了文档，返回client

## 检索原理

​    ![0](https://note.youdao.com/yws/public/resource/06b6f48ef4e56cb429410a1f35176f14/xmlnote/WEBRESOURCEd9d8a50b3e09e33d5f4367f98211d401/4890)

1.  client发起查询请求，某个DataNode接收到请求，该DataNode就会成为协调节点（Coordinating Node）。

2.  协调节点（Coordinating Node）将查询请求广播到每一个数据节点，这些数据节点的分片会处理该查询请求。

3. 每个分片进行数据查询，将符合条件的数据放在一个优先队列中，并将这些数据的文档ID、节点信息、分片信息返回给协调节点。

4. 协调节点将所有的结果进行汇总，并进行全局排序。

5. 协调节点向包含这些文档ID的分片发送get请求，对应的分片将文档数据返回给协调节点，最后协调节点将数据返回给客户端。

## 准实时索引实现

1. 溢写到文件系统缓存

   当数据写入到ES分片时，会首先写入到内存中，然后通过内存的buffer生成一个segment，并刷到**文件系统缓存**中，数据可以被检索（注意不是直接刷到磁盘）。ES中默认1秒，refresh一次。

2. 写translog保障容错

   在写入到内存中的同时，也会记录translog日志，在refresh期间出现异常，会根据translog来进行数据恢复，等到文件系统缓存中的segment数据都刷到磁盘中，清空translog文件

3. flush到磁盘

   ES默认每隔30分钟会将文件系统缓存的数据刷入到磁盘

4. segment合并

   Segment太多时，ES定期会将多个segment合并成为大的segment，减少索引查询时IO开销，此阶段ES会真正的物理删除（之前执行过的delete的数据）

​    ![0](https://note.youdao.com/yws/public/resource/06b6f48ef4e56cb429410a1f35176f14/xmlnote/WEBRESOURCE43a725c2701afe9abf580718839878c1/4892)