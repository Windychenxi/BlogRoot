---
title: ELK 专题六 （ELK + FileBeat 整合）
top_img: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/big-data-gbeadaf748_1920.jpg
cover: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/big-data-gbeadaf748_1920.jpg
tag: 
 - ELK
 - LogStash
 - ElasticSearch
 - ELK
 - Grok 插件
 - Mutate 插件
categories: 
 - ELK
---

# 前言

ELK + FileBeat 整合，实现由 FileBeat 监控日志变化，并发送给 LogStash。由 LogStash 按照指定的规则进行数据清洗，在发送至 ElasticSearch 存储。

# 推荐阅读

- [ELK专题一 IK 分词器源码升级改造实现热更新机制](https://windychenxi.github.io/2023/02/12/ELK/IK%E5%88%86%E8%AF%8D%E5%99%A8%E6%BA%90%E7%A0%81%E5%8D%87%E7%BA%A7%E6%94%B9%E9%80%A0%E5%AE%9E%E7%8E%B0%E7%83%AD%E6%9B%B4%E6%96%B0%E6%9C%BA%E5%88%B6/)
- [ELK专题二 FileBeat 日志收集](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86/)
- [ELK专题三 LogStash 数据清洗与被压机制](https://windychenxi.github.io/2023/02/12/ELK/LogStash%20%E6%95%B0%E6%8D%AE%E6%B8%85%E6%B4%97%E4%B8%8E%E8%83%8C%E5%8E%8B%E6%9C%BA%E5%88%B6/)
- [ELK专题四 FileBeat + LogStash 整合](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20+%20LogStash%20%E6%95%B4%E5%90%88/)
- [ELK专题五 Google 浏览器插件 ElasticSeach-head 安装](https://windychenxi.github.io/2023/02/12/ELK/Google%E6%B5%8F%E8%A7%88%E5%99%A8%E6%8F%92%E4%BB%B6ElasticSearch-head%E5%AE%89%E8%A3%85/)
- [ELK专题六 ELK + FileBeat 整合](https://windychenxi.github.io/2023/02/12/ELK/ELK%20+%20FileBeat%20%E6%95%B4%E5%90%88/)
- [ELK专题七 ElasticSearch 优化](https://windychenxi.github.io/2023/02/12/ELK/ElasticSearch%20%E4%BC%98%E5%8C%96/)

# LogStash输出到ES

如果我们需要将数据输出值ES而不是控制台的话，我们修改Logstash的output配置。

例如：

```shell
output {
    elasticsearch {
       hosts => [ "localhost:9200" ]
    }
}
```

## 启动ES集群

启动 ES 集群时，请使用非root用户，同时关闭防火墙

```shell
nohup /usr/local/es/elasticsearch-7.6.1/bin/elasticsearch 2>&1 &
```

## 配置 LogStash

重新拷贝一份配置文件

```shell
cd /usr/local/es/logstash-7.6.1
touch config/filebeat-elasticSearch.conf
```

将output修改为Elasticsearch

```shell
vim config/filebeat-elasticSearch.conf
```

```shell
input {
    beats {
      port => 5044
    }
}

output {
    elasticsearch {
    hosts => [ "192.168.10.158:9200","192.168.10.158:9200","192.168.10.158:9200"]
}
stdout {
    codec => rubydebug
    }
}
```

## 启动 LogStash

启动 LogStash 服务

```shell
bin/logstash -f config/filebeat-elasticSearch.conf --config.reload.automatic
```

启动成功

```verilog
Sending Logstash logs to /home/ELK/logStash/logstash-7.6.1/logs which is now configured via log4j2.properties
[2023-02-11T20:56:42,027][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
[2023-02-11T20:56:42,106][INFO ][logstash.runner          ] Starting Logstash {"logstash.version"=>"7.6.1"}
[2023-02-11T20:56:43,224][INFO ][org.reflections.Reflections] Reflections took 24 ms to scan 1 urls, producing 20 keys and 40 values
[2023-02-11T20:56:44,113][INFO ][logstash.outputs.elasticsearch][main] Elasticsearch pool URLs updated {:changes=>{:removed=>[], :added=>[http://192.168.10.30:9200/, http://192.168.10.31:9200/, http://192.168.10.32:9200/]}}
[2023-02-11T20:56:44,244][WARN ][logstash.outputs.elasticsearch][main] Restored connection to ES instance {:url=>"http://192.168.10.30:9200/"}
[2023-02-11T20:56:44,280][INFO ][logstash.outputs.elasticsearch][main] ES Output version determined {:es_version=>7}
[2023-02-11T20:56:44,289][WARN ][logstash.outputs.elasticsearch][main] Detected a 6.x and above cluster: the `type` event field won't be used to determine the document _type {:es_version=>7}
[2023-02-11T20:56:44,332][WARN ][logstash.outputs.elasticsearch][main] Restored connection to ES instance {:url=>"http://192.168.10.31:9200/"}
[2023-02-11T20:56:44,360][WARN ][logstash.outputs.elasticsearch][main] Restored connection to ES instance {:url=>"http://192.168.10.32:9200/"}
[2023-02-11T20:56:44,407][INFO ][logstash.outputs.elasticsearch][main] New Elasticsearch output {:class=>"LogStash::Outputs::ElasticSearch", :hosts=>["//192.168.10.30:9200", "//192.168.10.31:9200", "//192.168.10.32:9200"]}
[2023-02-11T20:56:44,455][INFO ][logstash.outputs.elasticsearch][main] Using default mapping template
[2023-02-11T20:56:44,500][WARN ][org.logstash.instrument.metrics.gauge.LazyDelegatingGauge][main] A gauge metric of an unknown type (org.jruby.specialized.RubyArrayOneObject) has been create for key: cluster_uuids. This may result in invalid serialization.  It is recommended to log an issue to the responsible developer/development team.
[2023-02-11T20:56:44,505][INFO ][logstash.javapipeline    ][main] Starting pipeline {:pipeline_id=>"main", "pipeline.workers"=>2, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>50, "pipeline.max_inflight"=>250, "pipeline.sources"=>["/home/ELK/logStash/logstash-7.6.1/config/filebeat-elasticSearch.conf"], :thread=>"#<Thread:0x74cf420a run>"}
[2023-02-11T20:56:44,551][INFO ][logstash.outputs.elasticsearch][main] Attempting to install template {:manage_template=>{"index_patterns"=>"logstash-*", "version"=>60001, "settings"=>{"index.refresh_interval"=>"5s", "number_of_shards"=>1, "index.lifecycle.name"=>"logstash-policy", "index.lifecycle.rollover_alias"=>"logstash"}, "mappings"=>{"dynamic_templates"=>[{"message_field"=>{"path_match"=>"message", "match_mapping_type"=>"string", "mapping"=>{"type"=>"text", "norms"=>false}}}, {"string_fields"=>{"match"=>"*", "match_mapping_type"=>"string", "mapping"=>{"type"=>"text", "norms"=>false, "fields"=>{"keyword"=>{"type"=>"keyword", "ignore_above"=>256}}}}}], "properties"=>{"@timestamp"=>{"type"=>"date"}, "@version"=>{"type"=>"keyword"}, "geoip"=>{"dynamic"=>true, "properties"=>{"ip"=>{"type"=>"ip"}, "location"=>{"type"=>"geo_point"}, "latitude"=>{"type"=>"half_float"}, "longitude"=>{"type"=>"half_float"}}}}}}}
[2023-02-11T20:56:44,578][INFO ][logstash.outputs.elasticsearch][main] Installing elasticsearch template to _template/logstash
[2023-02-11T20:56:45,212][INFO ][logstash.inputs.beats    ][main] Beats inputs: Starting input listener {:address=>"0.0.0.0:5044"}
[2023-02-11T20:56:45,227][INFO ][logstash.javapipeline    ][main] Pipeline started {"pipeline.id"=>"main"}
[2023-02-11T20:56:45,274][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
[2023-02-11T20:56:45,337][INFO ][org.logstash.beats.Server][main] Starting server on port: 5044
[2023-02-11T20:56:45,508][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
```

## 查看索引

在浏览器中，打开 ElasticSearch-head 插件，输入 ElasticSearch 集群地址。连接后，可以看到如下信息

![image-20230212191043995](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/image-20230212191043995.png)

点击<**数据浏览**>，查看 logstash-2023.02.12-000001 索引信息

![image-20230212191201068](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/image-20230212191201068.png)



从输出返回结果，我们可以看到，日志确实已经保存到了Elasticsearch中，而且我们看到消息数据是封装在名为**message**中的，其他的数据也封装在一个个的字段中。我们其实更想要把消息解析成一个个的字段。例如：IP字段、时间、请求方式、请求URL、响应结果，这样。



# LogStash 过滤器

从日志文件中收集到的数据包含了很多有效信息，比如IP、时间等，

在Logstash中可以配置过滤器Filter对采集到的数据进行过滤处理，在Logstash中，有大量的插件供我们使用。

参考官网：

https://www.elastic.co/guide/en/logstash/7.6/filter-plugins.html



查看 LogStash 已经安装的插件

```shell
cd /usr/local/es/logstash-7.6.1/
bin/logstash-plugin list
```

此处重点讲解 Grok 插件。

# Grok 插件

Grok 是通过模式匹配的方式来识别日志中的数据。可以把 Grok 插件简单理解为升级版本的正则表达式。它拥有更多的模式，默认 LogStash 拥有120 个模式。如果这些模式不满足我们解析日志的需求，可以直接使用正则表达式来进行匹配。

官网：https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns

grok模式的语法是：

```yml
%{SYNTAX:SEMANTIC}
```

> **SYNTAX**指的是Grok模式名称 ；**SEMANTIC**是给模式匹配到的文本字段名。例如：
>
> `%{NUMBER:duration} %{IP:client}  `
>
> `duration` 表示：匹配一个数字，`client`表示匹配一个`IP`地址。             

默认在Grok中，所有匹配到的的数据类型都是字符串，如果要转换成int类型（目前只支持int和float），可以这样：

```yaml
%{NUMBER:duration:int} %{IP:client}
```

以下是常用的Grok模式

| NUMBER | 匹配数字（包含：小数） |
| ------ | ---------------------- |
| INT    | 匹配整形数字           |
| POSINT | 匹配正整数             |
| WORD   | 匹配单词               |
| DATA   | 匹配所有字符           |
| IP     | 匹配IP地址             |
| PATH   | 匹配路径               |

## 用法

![截图 (3)](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/%E6%88%AA%E5%9B%BE%20(3).png)

```yaml
filter {
    grok {
      match => { "message" => "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}" }
    }
}
```

匹配日志中的IP、日期

```verilog
90.224.57.84 - - [15/Feb/2023:00:27:19 +0800] "POST /report HTTP/1.1" 404 21 "www.baidu.com" "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.104 Safari/537.36 Core/1.53.4549.400 QQBrowser/9.7.12900"
```

##  Kibana 测试 Grok

在 Kibana 上测试 Grok 语法，可以看到 Grok 已经从日志中获取到了date及IP信息

![image-20230212211527995](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/image-20230212211527995.png)

## 配置 LogStash

```shell
vim config/filebeat-filter-console.conf
```

配置如下类容

```yaml
input {
	beats {
		port => 5044
	}
}

filter {
	grok {
		match => { 
			"message" => "%{IP:ip} - - \[%{HTTPDATE:date}\]" 
		}
	} 
}

output {
	stdout {
		codec => rubydebug
	}
}
```

重启 LogStash

```shell
bin/logstash -f config/filebeat-filter-console.conf --config.reload.automatic
```

## 过滤结果

我们看到，经过Grok过滤器插件处理之后，我们已经获取到了ip和date两个字段。接下来，我们就可以继续解析其他的字段。

![image-20230212205558456](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/image-20230212205558456.png)

## 解析所有字段

将日志解析成以下字段

| 字段名    | 说明                 |
| --------- | -------------------- |
| client IP | 浏览器端IP           |
| timestamp | 请求的时间戳         |
| method    | 请求方式（GET/POST） |
| uri       | 请求的链接地址       |
| status    | 服务器端响应状态     |
| length    | 响应的数据长度       |
| reference | 从哪个URL跳转而来    |
| browser   | 浏览器               |

为了方便进行Grok开发，此处使用Kibana来进行调试，我们使用IP就可以把前面的IP字段匹配出来，使用HTTPDATE可以将后面的日期匹配出来。

为了方便测试，我们可以使用Kibana来进行Grok开发：

![截图 (4)](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/%E6%88%AA%E5%9B%BE%20(4).png)

![截图 (5)](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/%E6%88%AA%E5%9B%BE%20(5).png)

可以在Kibana中先把Grok的表达式写好，然后再放入到Logstash的配置文件中，这样可以大大提升调试的效率。

```yaml
filter {
	grok {
		match => { 
			"message" => "%{IP:ip} - - \[%{HTTPDATE:date}\] \"%{WORD:method} %{PATH:uri} %{DATA}\" %{INT:status} %{INT:length} \"%{DATA:reference}\" \"%{DATA:browser}\"" 
		}
	} 
}
```

在 Kibaba 中测试结果如下

![image-20230212211306263](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/image-20230212211306263.png)

## 输入 ElasticSearch

到目前为止，我们已经通过了Grok Filter可以将日志消息解析成一个一个的字段，那现在我们需要将这些字段保存到Elasticsearch中。我们看到了Logstash的输出中，有大量的字段，但如果我们只需要保存我们需要的8个，该如何处理呢？而且，如果我们需要将日期的格式进行转换，我们又该如何处理呢？



**过滤出来需要的字段**

要过滤出来我们需要的字段。我们需要使用mutate插件。mutate插件主要是作用在字段上，例如：它可以对字段进行重命名、删除、替换或者修改结构。

# mutate 插件

用法如下

```yaml
mutate {
    enable_metric => "false"
    remove_field => ["message", "log", "tags", "@timestamp", "input", "agent", "host", "ecs", "@version"]
}
```

> **enable_metric**
>
> `false`表示禁用默认输出
>
> **remove_field**
>
> 从默认中移除不需要的字段

# date 插件

要将日期格式进行转换，我们可以使用Date插件来实现。该插件专门用来解析字段中的日期，官方说明文档：https://www.elastic.co/guide/en/logstash/7.6/plugins-filters-date.html



用法如下

```yaml
date {
    match => ["date","dd/MMM/yyyy:HH:mm:ss Z","yyyy-MM-dd HH:mm:ss"]
    target => "date"
    }
}
```

将date字段转换为「年月日 时分秒」格式。默认字段经过date插件处理后，会输出到@timestamp字段，所以，我们可以通过修改target属性来重新定义输出字段。

# 指定ES索引

我们可以通过

```yaml
elasticsearch {
    hosts => ["192.168.10.30:9200" ,"192.168.10.31:9200" ,"192.168.10.32:9200"]
    index => "xxx"
}
```

index来指定索引名称，默认输出的index名称为：logstash-%{+yyyy.MM.dd}。

> **注意**
>
> 1. 要在index中使用时间格式化，filter的输出必须包含 @timestamp字段，否则将无法解析日期。
>
> 2. index名称中，不能出现大写字符



# 融合测试

现在使用 Grok、Mutate、Date 插件融合过滤日志信息，并指定 ElasticSearch 索引。

## 配置 LogStash

在 config/ 目录下，创建 `filebeat-tomcat-weblog.conf` 配置文件

```yaml
input {
    beats {
    port => 5044
    }
}

filter {
    grok {
    match => { 
    "message" => "%{IP:ip} - - \[%{HTTPDATE:date}\] \"%{WORD:method} %{PATH:uri} %{DATA}\" %{INT:status:int} %{INT:length:int} \"%{DATA:reference}\" \"%{DATA:browser}\"" 
    }
}
mutate {
    enable_metric => "false"
    remove_field => ["message", "log", "tags", "input", "agent", "host", "ecs", "@version"]
}
date {
    match => ["date","dd/MMM/yyyy:HH:mm:ss Z","yyyy-MM-dd HH:mm:ss"]
    target => "date"
    }
}

output {
    stdout {
    codec => rubydebug
}
elasticsearch {
    hosts => ["192.168.10.30:9200" ,"192.168.10.31:9200" ,"192.168.10.32:9200"]
    index => "tomcat_web_log_%{+YYYY-MM}"
    }
}
```



## 启动 LogStash

```shell
bin/logstash -f config/filebeat-tomcat-weblog.conf --config.test_and_exit
bin/logstash -f config/filebeat-tomcat-weblog.conf --config.reload.automatic
```

## 收集结果

数据过滤

从 LogStash 控制台上可以看到

```verilog
{
        "length" => 21,
     "reference" => "www.baidu.com",
          "date" => 2023-02-14T16:27:19.000Z,
        "method" => "POST",
           "uri" => "/report",
    "@timestamp" => 2023-02-12T13:45:14.339Z,
       "browser" => "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.104 Safari/537.36 Core/1.53.4549.400 QQBrowser/9.7.12900",
        "status" => 404,
            "ip" => "90.224.57.84"
}
```

已经过滤出了我们所定义的字段。



来到 ElasticSearch-Head 界面，可以查看到 ElasticSearch 已经收集到了我们所定义的字段

![image-20230212215052836](https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/image-20230212215052836.png)