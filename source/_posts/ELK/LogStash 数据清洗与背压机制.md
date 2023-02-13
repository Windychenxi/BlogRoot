---
title: ELK 专题三 （LogStash 数据清洗与被压机制）
top_img: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/big-data-g34dcd7c5d_640.jpg
cover: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/big-data-g34dcd7c5d_640.jpg
tag:
 - LogStash
 - ELK
categories: 
 - ELK
---

# 前言

LogStash 一般用来数据清洗，即数据过滤。可以定制从日志中要过滤出的字段。

# 推荐阅读

- [ELK专题一 IK 分词器源码升级改造实现热更新机制](https://windychenxi.github.io/2023/02/12/ELK/IK%E5%88%86%E8%AF%8D%E5%99%A8%E6%BA%90%E7%A0%81%E5%8D%87%E7%BA%A7%E6%94%B9%E9%80%A0%E5%AE%9E%E7%8E%B0%E7%83%AD%E6%9B%B4%E6%96%B0%E6%9C%BA%E5%88%B6/)
- [ELK专题二 FileBeat 日志收集](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86/)
- [ELK专题三 LogStash 数据清洗与被压机制](https://windychenxi.github.io/2023/02/12/ELK/LogStash%20%E6%95%B0%E6%8D%AE%E6%B8%85%E6%B4%97%E4%B8%8E%E8%83%8C%E5%8E%8B%E6%9C%BA%E5%88%B6/)
- [ELK专题四 FileBeat + LogStash 整合](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20+%20LogStash%20%E6%95%B4%E5%90%88/)
- [ELK专题五 Google 浏览器插件 ElasticSeach-head 安装](https://windychenxi.github.io/2023/02/12/ELK/Google%E6%B5%8F%E8%A7%88%E5%99%A8%E6%8F%92%E4%BB%B6ElasticSearch-head%E5%AE%89%E8%A3%85/)
- [ELK专题六 ELK + FileBeat 整合](https://windychenxi.github.io/2023/02/12/ELK/ELK%20+%20FileBeat%20%E6%95%B4%E5%90%88/)
- [ELK专题七 ElasticSearch 优化](https://windychenxi.github.io/2023/02/12/ELK/ElasticSearch%20%E4%BC%98%E5%8C%96/)

# 简介

Logstash是一个开源的数据采集引擎。它可以动态地将不同来源的数据统一采集，并按照指定的数据格式进行处理后，将数据加载到其他的目的地。最开始，Logstash主要是针对日志采集，但后来Logstash开发了大量丰富的插件，所以，它可以做更多的海量数据的采集。

它可以处理各种类型的日志数据，例如：Apache的web log、Java的log4j日志数据，或者是系统、网络、防火墙的日志等等。它也可以很容易的和Elastic Stack的Beats组件整合，也可以很方便的和关系型数据库、NoSQL数据库、MQ等整合。

![wps1](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/wps1.jpeg)

# 经典架构

![截图 (1)](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/%E6%88%AA%E5%9B%BE%20(1).png)

# 对比 FileBeat

- logstash是JVM跑的，资源消耗比较大

- FileBeat是基于golang编写的，功能较少但资源消耗也比较小，更轻量级

- logstash 和filebeat都具有日志收集功能，Filebeat更轻量，占用资源更少

- logstash 具有filter功能，能过滤分析日志

一般结构都是filebeat采集日志，然后发送到消息队列，redis，MQ中然后logstash去获取，利用filter功能过滤分析，然后存储到elasticsearch中。

FileBeat和Logstash配合，实现背压机制。

![wps3](C:\Users\Administrator\Downloads\wps3.jpeg)

# Logstash 安装

## 下载 Logstash

```text
https://www.elastic.co/cn/downloads/past-releases/logstash-7-6-1
```

## 解压 Logstash

```bash
unzip logstash-7.6.1 -d /usr/local/es/
```

## 运行测试

```bash
cd /usr/local/es/logstash-7.6.1/
bin/logstash -e 'input { stdin { } } output { stdout {} }'
```

> -e 选项表示，直接把配置放在命令中，这样可以有效快速进行测试

待 LogStash 启动完毕

```verilog
Sending Logstash logs to /home/ELK/logStash/logstash-7.6.1/logs which is now configured via log4j2.properties
[2023-02-11T18:32:21,419][INFO ][logstash.setting.writabledirectory] Creating directory {:setting=>"path.queue", :path=>"/home/ELK/logStash/logstash-7.6.1/data/queue"}
[2023-02-11T18:32:21,686][INFO ][logstash.setting.writabledirectory] Creating directory {:setting=>"path.dead_letter_queue", :path=>"/home/ELK/logStash/logstash-7.6.1/data/dead_letter_queue"}
[2023-02-11T18:32:21,943][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
[2023-02-11T18:32:21,954][INFO ][logstash.runner          ] Starting Logstash {"logstash.version"=>"7.6.1"}
[2023-02-11T18:32:21,974][INFO ][logstash.agent           ] No persistent UUID file found. Generating new UUID {:uuid=>"ccfe7a60-a9ce-44b4-b4fc-503d8745082d", :path=>"/home/ELK/logStash/logstash-7.6.1/data/uuid"}
[2023-02-11T18:32:23,078][INFO ][org.reflections.Reflections] Reflections took 42 ms to scan 1 urls, producing 20 keys and 40 values
[2023-02-11T18:32:23,630][WARN ][org.logstash.instrument.metrics.gauge.LazyDelegatingGauge][main] A gauge metric of an unknown type (org.jruby.RubyArray) has been create for key: cluster_uuids. This may result in invalid serialization.  It is recommended to log an issue to the responsible developer/development team.
[2023-02-11T18:32:23,649][INFO ][logstash.javapipeline    ][main] Starting pipeline {:pipeline_id=>"main", "pipeline.workers"=>2, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>50, "pipeline.max_inflight"=>250, "pipeline.sources"=>["config string"], :thread=>"#<Thread:0x63c93421 run>"}
[2023-02-11T18:32:24,342][INFO ][logstash.javapipeline    ][main] Pipeline started {"pipeline.id"=>"main"}
The stdin plugin is now waiting for input:
[2023-02-11T18:32:24,407][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
[2023-02-11T18:32:24,555][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
```

然后，随便在控制台中输入内容，等待LogStash的输出

```bash
Hello LogStash
```

LogStash 输出

```bash
{
    "@timestamp" => 2023-02-12T02:37:58.166Z,
      "@version" => "1",
          "host" => "localhost.localdomain",
       "message" => "Hello LogStash"
}
```

