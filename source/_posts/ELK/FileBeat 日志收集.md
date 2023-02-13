---
title: ELK 专题二 （FileBeat 日志收集）
top_img: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/analytics-g16ec55044_1280.jpg
cover: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/analytics-g16ec55044_1280.jpg
tag:
 - ELK
 - FileBeat
categories: 
 - ELK
---

# 前言

Beats 是一个开源代码的数据发送器。可以把 Beats 作为一种代理安装在服务器上，这样就可以比较方便地将数据发送到 Elasticsearch 或者 Logstash 中。Elastic Stack 提供了多种类型的 Beats 组件。

# 推荐阅读

- [ELK专题一 IK 分词器源码升级改造实现热更新机制](https://windychenxi.github.io/2023/02/12/ELK/IK%E5%88%86%E8%AF%8D%E5%99%A8%E6%BA%90%E7%A0%81%E5%8D%87%E7%BA%A7%E6%94%B9%E9%80%A0%E5%AE%9E%E7%8E%B0%E7%83%AD%E6%9B%B4%E6%96%B0%E6%9C%BA%E5%88%B6/)
- [ELK专题二 FileBeat 日志收集](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86/)
- [ELK专题三 LogStash 数据清洗与被压机制](https://windychenxi.github.io/2023/02/12/ELK/LogStash%20%E6%95%B0%E6%8D%AE%E6%B8%85%E6%B4%97%E4%B8%8E%E8%83%8C%E5%8E%8B%E6%9C%BA%E5%88%B6/)
- [ELK专题四 FileBeat + LogStash 整合](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20+%20LogStash%20%E6%95%B4%E5%90%88/)
- [ELK专题五 Google 浏览器插件 ElasticSeach-head 安装](https://windychenxi.github.io/2023/02/12/ELK/Google%E6%B5%8F%E8%A7%88%E5%99%A8%E6%8F%92%E4%BB%B6ElasticSearch-head%E5%AE%89%E8%A3%85/)
- [ELK专题六 ELK + FileBeat 整合](https://windychenxi.github.io/2023/02/12/ELK/ELK%20+%20FileBeat%20%E6%95%B4%E5%90%88/)
- [ELK专题七 ElasticSearch 优化](https://windychenxi.github.io/2023/02/12/ELK/ElasticSearch%20%E4%BC%98%E5%8C%96/)



# Beats 类型

| 数据类型         | Beat 类型    |
| ---------------- | ------------ |
| 审计数据         | AuditBeat    |
| 日志文件         | FileBeat     |
| 云数据           | FunctionBeat |
| 可用性数据       | HeartBeat    |
| 系统日志         | JournalBeat  |
| 指标数据         | MetricBeat   |
| 网络流量数据     | PacketBeat   |
| Windows 事件日志 | WinlogBeat   |

![clipboard](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/clipboard.png)

Beats 可以直接将数据发送到 Elasticsearch 或者发送到 LogStash，基于 LogStash 可以进一步对数据进行处理，然后将处理后的数据存入到 Elasticsearch，最后使用 Kibana 进行数据可视化。



# FileBeat简介

FileBeat 专门用于转发和手机日志数据的轻量级采集工具。它可以作为代理安装在服务器上，FileBeat 见识指定路径的日志文件，收集日志数据，并将收集到的日志转发到 Elasticsearch 或者 LogStash。

# FileBeat工作原理

启动FileBeat时，会启动一个或者多个输入（Input），这些Input监控指定的日志数据位置。FileBeat会针对每一个文件启动一个Harvester（收割机）。Harvester读取每一个文件的日志，将新的日志发送到libbeat，libbeat将数据收集到一起，并将数据发送给输出（Output）。

![clipboard (1)](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/clipboard%20(1).png)

# 安装 FIleBeat

安装FileBeat只需要将FileBeat Linux安装包上传到Linux系统，并将压缩包解压到系统就可以了。

FileBeat官方下载地址：

https://www.elastic.co/cn/downloads/past-releases/filebeat-7-6-1

上传FileBeat安装到Linux，并解压。

```shell
tar -xvzf filebeat-7.6.1-linux-x86_64.tar.gz -C /usr/local/es/
```

## 采集日志

使用FileBeat采集MQ日志到ElasticSearch

## 需求分析

在资料中有一个mq_server.log.tar.gz压缩包，里面包含了很多的MQ服务器日志，现在我们为了通过在Elasticsearch中快速查询这些日志，定位问题。我们需要用FileBeats将日志数据上传到Elasticsearch中。



**问题**

首先，我们要指定FileBeat采集哪些MQ日志，因为FileBeats中必须知道采集存放在哪儿的日志，才能进行采集。

 其次，采集到这些数据后，还需要指定FileBeats将采集到的日志输出到Elasticsearch，那么Elasticsearch的地址也必须指定。



# 配置 FileBeats

FileBeats配置文件主要分为两个部分。

- inputs	输入数据
- output   输出数据



## input 配置

在 FileBeats 中，可以读取一个或多个数据源。

```yml
filebeat.inputs:
# "-" 表示可以配置多个
- type: log  # type表示采集的是读取每一行日志文件，还是可以配置stdin，表示从标准输入刘输入
  enabled: true  # 启用该输入
  paths:
    - /var/log/*.log  # 采集日志路径
    #- c:\programdata\elasticsearch\logs\*
```

## output 配置

```yaml
output.elasticsearch:  # 输出到 ElasticSearch
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]  # ElasticSearch 的集群地址
```

默认 FileBeat 会将日志数据放入到名称为 filebeat-%filebeat-%版本号%-yyyy.mm.dd 的索引中。

> FileBeats 中 filebeat.reference.yml 包含了 FileBeats 所有支持的配置选项

## 配置文件

创建配置文件

```shell
cd /usr/local/es/filebeat-7.6.1-linux-x86_64
touch filebeat_mq_log.yml
vim filebeat_mq_log.yml
```

复制一下到配置文件中

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/mq/log/server.log.*

output.elasticsearch:
    hosts: ["192.168.21.130:9200", "192.168.21.131:9200", "192.168.21.132:9200"]
```

## 运行 FileBeat

### 启动 ES 集群

在每个节点上执行以下命令，启动 Elasticsearch 集群

```bash
nohup /usr/local/es/elasticsearch-7.6.1/bin/elasticsearch 2>&1 &
```

### 运行 FileBeat

```bash
./filebeat -c filebeat_mq_log.yml -e
```

### 上传日志

将日志数据上传到 /var/mq/log，并解压

```bash
mkdir -p /var/mq/log
cd /var/mq/log
tar -zxvf mq_server.log.tar.gz
```

### 查询数据

通过 head 插件，我们可以看到 filebeat 采集了日志消息，并写入到 Elasticsearch 集群中。



# FileBeat 如何工作

FileBeat 主要由 input 和 harvesters（收割机）组成。这两个组件协同工作，并将数据发送到指定的输出。

## inputs（输入）

input是负责管理Harvesters和查找所有要读取的文件的组件。如果输入类型是 log，input组件会查找磁盘上与路径描述的所有文件，并为每个文件启动一个Harvester，每个输入都独立地运行

## Harvesters（收割机）

Harvesters负责读取单个文件的内容，它负责打开/关闭文件，并逐行读取每个文件的内容，将读取到的内容发送给输出每个文件都会启动一个Harvester，Harvester运行时，文件将处于打开状态。如果文件在读取时，被移除或者重命名，FileBeat将继续读取该文件。

## 如何保持文件状态

FileBeat保存每个文件的状态，并定时将状态信息保存在磁盘的「注册表」文件中，该状态记录Harvester读取的最后一次偏移量，并确保发送所有的日志数据。如果输出（Elasticsearch或者Logstash）无法访问，FileBeat会记录成功发送的最后一行，并在输出（Elasticsearch或者Logstash）可用时，继续读取文件发送数据。

在运行FileBeat时，每个input的状态信息也会保存在内存中，重新启动FileBeat时，会从「注册表」文件中读取数据来重新构建状态。

在/usr/local/es/filebeat-7.6.1-linux-x86_64/data目录中有一个Registry文件夹，里面有一个data.json，该文件中记录了Harvester读取日志的offset。

![截图](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/%E6%88%AA%E5%9B%BE.png)