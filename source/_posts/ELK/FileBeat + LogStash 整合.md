---
title: ELK 专题四 （FileBeat + LogStash 整合）
top_img: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/big-data-gc3961aa73_1920.jpg
cover: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/big-data-gc3961aa73_1920.jpg
tag:
 - ELK
 - FileBeat
 - LogStash
categories: 
 - ELK
---

# 前言

使用 FileBeat 进行数据实时监听，一旦发生变化时发送给 LogStash，LogStash 会按照指定的规则过滤输出日志信息。

# 推荐阅读

- [ELK专题一 IK 分词器源码升级改造实现热更新机制](https://windychenxi.github.io/2023/02/12/ELK/IK%E5%88%86%E8%AF%8D%E5%99%A8%E6%BA%90%E7%A0%81%E5%8D%87%E7%BA%A7%E6%94%B9%E9%80%A0%E5%AE%9E%E7%8E%B0%E7%83%AD%E6%9B%B4%E6%96%B0%E6%9C%BA%E5%88%B6/)
- [ELK专题二 FileBeat 日志收集](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86/)
- [ELK专题三 LogStash 数据清洗与被压机制](https://windychenxi.github.io/2023/02/12/ELK/LogStash%20%E6%95%B0%E6%8D%AE%E6%B8%85%E6%B4%97%E4%B8%8E%E8%83%8C%E5%8E%8B%E6%9C%BA%E5%88%B6/)
- [ELK专题四 FileBeat + LogStash 整合](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20+%20LogStash%20%E6%95%B4%E5%90%88/)
- [ELK专题五 Google 浏览器插件 ElasticSeach-head 安装](https://windychenxi.github.io/2023/02/12/ELK/Google%E6%B5%8F%E8%A7%88%E5%99%A8%E6%8F%92%E4%BB%B6ElasticSearch-head%E5%AE%89%E8%A3%85/)
- [ELK专题六 ELK + FileBeat 整合](https://windychenxi.github.io/2023/02/12/ELK/ELK%20+%20FileBeat%20%E6%95%B4%E5%90%88/)
- [ELK专题七 ElasticSearch 优化](https://windychenxi.github.io/2023/02/12/ELK/ElasticSearch%20%E4%BC%98%E5%8C%96/)

# 采集日志信息

# 需求

Tomcat 服务器运行过程中产生很多日志信息，通过LogStash采集并存储日志信息至ElasticSearch中。

Tomcat 日志文件如下

```shell
[root@localhost apache-tomcat-10.0.27]# tree logs/
logs/
├── catalina.2023-02-11.log
├── catalina.out
├── localhost.2023-02-11.log
└── localhost_access_log.2023-02-11.txt

0 directories, 4 files
```

日志文件信息如下所示

```txt
192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] "GET / HTTP/1.1" 200 11437
192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] "GET /tomcat.css HTTP/1.1" 200 5895
192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] "GET /tomcat.svg HTTP/1.1" 200 68761
192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] "GET /bg-upper.png HTTP/1.1" 200 3103
192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] "GET /bg-nav.png HTTP/1.1" 200 1401
192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] "GET /asf-logo-wide.svg HTTP/1.1" 200 27530
192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] "GET /bg-button.png HTTP/1.1" 200 713
192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] "GET /bg-middle.png HTTP/1.1" 200 1918
192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] "GET /favicon.ico HTTP/1.1" 200 21630
```



这个日志其实由一个个的字段拼接而成

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

为了便于后期数据分析，需要将该日志信息解析成指定字段的filed的值并存储，便于后期分析。

# 准备日志

将Tomcat日志文件上传到指定的目录

# 发送日志

使用 FileBeats 将日志发送到 LogStash

之前，我们使用的FileBeat是通过FileBeat的Harvester组件监控日志文件，然后将日志以一定的格式保存到Elasticsearch中，而现在我们需要配置FileBeats将数据发送到Logstash再将数据发送至ElasticSearch。

# 配置 FileBeat

FileBeat相关信息按照如下格式配置即可：

```yaml
#----------------------------- Logstash output ---------------------------------
output.logstash:
  # Boolean flag to enable or disable the output module.
  enabled: true

  # The Logstash hosts
  hosts: ["localhost:5044"]
```

> hosts配置的是Logstash监听的IP地址/机器名以及端口号

进入fileBeat安装路径并准备FileBeat配置文件

```bash
cd /usr/local/es/filebeat-7.6.1-linux-x86_64
touch filebeat-logstash.yml
vim filebeat-logstash.yml
```



因为Tomcat的web log日志都是以IP地址开头的，所以我们需要修改下匹配字段

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/tomcat/log/access*.*
  multiline.pattern: '^\\d+\\.\\d+\\.\\d+\\.\\d+ '
  multiline.negate: true
  multiline.match: after

output.logstash:
   enabled: true
   hosts: ["192.168.21.133:5044"]
```

> 注意该配置中
>
> ```yaml
> multiline.pattern: '^\\d+\\.\\d+\\.\\d+\\.\\d+ '
> multiline.negate: true
> multiline.match: after
> ```
>
> 表示以IP地址开头的行追加到上一行



```yaml
multiline:
    pattern: '^[0-2][0-9]:[0-5][0-9]:[0-5][0-9]'
    negate: true
    match: after
```

上面配置的意思是：不以时间格式开头的行都合并到上一行的末尾（正则写的不好，忽略忽略）

- pattern：正则表达式

- negate：true 或 false；默认是false，匹配pattern的行合并到上一行；true，不匹配pattern的行合并到上一行

- match：after 或 before，合并到上一行的末尾或开头

# 启动 FileBeat

启动FileBeat，并指定使用指定的配置文件

```shell
./filebeat -e -c filebeat-logstash.yml
```

FileBeat将尝试建立与Logstash监听的IP和端口号进行连接。但此时，我们并没有开启并配置Logstash，所以FileBeat是无法连接到Logstash的。



# 配置 LogStash

Logstash的配置文件和FileBeat类似，它也需要有一个input、和output。

基本格式如下：

```shell
# #号表示添加注释
# input表示要接收的数据
input {
}

# file表示对接收到的数据进行过滤处理
filter {

}

# output表示将数据输出到其他位置
output {
}
```

配置从FileBeat接收数据

```shell
cd /usr/local/es/logstash-7.6.1
vim config/filebeat-console.conf
```

```conf
input {
    beats {
      port => 5044
    }
}

output {
    stdout {
      codec => rubydebug
    }
}
```

> **提示**
>
> 复制可能存在乱码，可以将复制内容先放到文本文档，再拷出来

# 验证 LogStash

测试 LogStash 配置是否正确

```shell
bin/logstash -f config/filebeat-console.conf --config.test_and_exit
```

LogStash 测试启动成功

```verilog
Sending Logstash logs to /home/ELK/logStash/logstash-7.6.1/logs which is now configured via log4j2.properties
[2023-02-11T19:35:56,635][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
[2023-02-11T19:35:57,461][INFO ][org.reflections.Reflections] Reflections took 58 ms to scan 1 urls, producing 20 keys and 40 values
Configuration OK
[2023-02-11T19:35:58,004][INFO ][logstash.runner          ] Using config.test_and_exit mode. Config Validation Result: OK. Exiting Logstash
```

# 启动 LogStash

启动 LogStash 服务

```shell
bin/logstash -f config/filebeat-console.conf --config.reload.automatic
```

启动成功

```verilog
Sending Logstash logs to /home/ELK/logStash/logstash-7.6.1/logs which is now configured via log4j2.properties
[2023-02-11T19:49:56,706][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
[2023-02-11T19:49:56,783][INFO ][logstash.runner          ] Starting Logstash {"logstash.version"=>"7.6.1"}
[2023-02-11T19:49:57,880][INFO ][org.reflections.Reflections] Reflections took 25 ms to scan 1 urls, producing 20 keys and 40 values
[2023-02-11T19:49:58,404][WARN ][org.logstash.instrument.metrics.gauge.LazyDelegatingGauge][main] A gauge metric of an unknown type (org.jruby.RubyArray) has been create for key: cluster_uuids. This may result in invalid serialization.  It is recommended to log an issue to the responsible developer/development team.
[2023-02-11T19:49:58,412][INFO ][logstash.javapipeline    ][main] Starting pipeline {:pipeline_id=>"main", "pipeline.workers"=>2, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>50, "pipeline.max_inflight"=>250, "pipeline.sources"=>["/home/ELK/logStash/logstash-7.6.1/config/filebeat-console.conf"], :thread=>"#<Thread:0x3ec7886b run>"}
[2023-02-11T19:49:59,196][INFO ][logstash.inputs.beats    ][main] Beats inputs: Starting input listener {:address=>"0.0.0.0:5044"}
[2023-02-11T19:49:59,217][INFO ][logstash.javapipeline    ][main] Pipeline started {"pipeline.id"=>"main"}
[2023-02-11T19:49:59,302][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
[2023-02-11T19:49:59,361][INFO ][org.logstash.beats.Server][main] Starting server on port: 5044
[2023-02-11T19:49:59,630][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
```

# 日志监听

启动 FileBeat 后，可以看到控制台已经连接 LogStash 成功，同时正在监听日志文件

```verilog
2023-02-11T19:52:00.922-0800    INFO    pipeline/output.go:95   Connecting to backoff(async(tcp://192.168.10.158:5044))
2023-02-11T19:52:00.923-0800    INFO    pipeline/output.go:105  Connection to backoff(async(tcp://192.168.10.158:5044)) established
2023-02-11T19:52:24.920-0800    INFO    [monitoring]    log/log.go:145  Non-zero metrics in the last 30s        {"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":100,"time":{"ms":105}},"total":{"ticks":110,"time":{"ms":122},"value":110},"user":{"ticks":10,"time":{"ms":17}}},"handles":{"limit":{"hard":4096,"soft":1024},"open":9},"info":{"ephemeral_id":"82569f6e-16d4-4a5c-9372-06d83d5a5d04","uptime":{"ms":30073}},"memstats":{"gc_next":8111376,"memory_alloc":6627960,"memory_total":14803392,"rss":31338496},"runtime":{"goroutines":28}},"filebeat":{"events":{"added":2,"done":2},"harvester":{"files":{"4f6bbc19-92bf-4434-a8ce-477369c614c0":{"last_event_published_time":"2023-02-11T19:51:59.921Z","last_event_timestamp":"2023-02-11T19:51:54.920Z","name":"/home/ELK/tomcat/apache-tomcat-10.0.27/logs/localhost_access_log.2023-02-11.txt","read_offset":747,"size":747,"start_time":"2023-02-11T19:51:54.919Z"}},"open_files":1,"running":1,"started":1}},"libbeat":{"config":{"module":{"running":0}},"output":{"events":{"acked":1,"batches":1,"total":1},"read":{"bytes":6},"type":"logstash","write":{"bytes":546}},"pipeline":{"clients":1,"events":{"active":0,"filtered":1,"published":1,"retry":1,"total":2},"queue":{"acked":1}}},"registrar":{"states":{"current":1,"update":2},"writes":{"success":3,"total":3}},"system":{"cpu":{"cores":2},"load":{"1":0.26,"15":0.23,"5":0.4,"norm":{"1":0.13,"15":0.115,"5":0.2}}}}}}
2023-02-11T19:52:54.919-0800    INFO    [monitoring]    log/log.go:145  Non-zero metrics in the last 30s        {"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":100,"time":{"ms":3}},"total":{"ticks":110,"time":{"ms":3},"value":110},"user":{"ticks":10}},"handles":{"limit":{"hard":4096,"soft":1024},"open":9},"info":{"ephemeral_id":"82569f6e-16d4-4a5c-9372-06d83d5a5d04","uptime":{"ms":60074}},"memstats":{"gc_next":8111376,"memory_alloc":6988704,"memory_total":15164136},"runtime":{"goroutines":28}},"filebeat":{"harvester":{"open_files":1,"running":1}},"libbeat":{"config":{"module":{"running":0}},"pipeline":{"clients":1,"events":{"active":0}}},"registrar":{"states":{"current":1}},"system":{"load":{"1":0.15,"15":0.22,"5":0.36,"norm":{"1":0.075,"15":0.11,"5":0.18}}}}}}
2023-02-11T19:53:24.921-0800    INFO    [monitoring]    log/log.go:145  Non-zero metrics in the last 30s        {"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":110,"time":{"ms":5}},"total":{"ticks":120,"time":{"ms":6},"value":120},"user":{"ticks":10,"time":{"ms":1}}},"handles":{"limit":{"hard":4096,"soft":1024},"open":9},"info":{"ephemeral_id":"82569f6e-16d4-4a5c-9372-06d83d5a5d04","uptime":{"ms":90073}},"memstats":{"gc_next":8111376,"memory_alloc":7316496,"memory_total":15491928},"runtime":{"goroutines":28}},"filebeat":{"harvester":{"open_files":1,"running":1}},"libbeat":{"config":{"module":{"running":0}},"pipeline":{"clients":1,"events":{"active":0}}},"registrar":{"states":{"current":1}},"system":{"load":{"1":0.09,"15":0.22,"5":0.33,"norm":{"1":0.045,"15":0.11,"5":0.165}}}}}}

```

# 日志收集

来到 LogStash 的控制台，可以看到 tomcat 的日志已被收集

```shell
/home/ELK/logStash/logstash-7.6.1/vendor/bundle/jruby/2.5.0/gems/awesome_print-1.7.0/lib/awesome_print/formatters/base_formatter.rb:31: warning: constant ::Fixnum is deprecated
{
       "message" => "192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] \"GET / HTTP/1.1\" 200 11437\n192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] \"GET /tomcat.css HTTP/1.1\" 200 5895\n192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] \"GET /tomcat.svg HTTP/1.1\" 200 68761\n192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] \"GET /bg-upper.png HTTP/1.1\" 200 3103\n192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] \"GET /bg-nav.png HTTP/1.1\" 200 1401\n192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] \"GET /asf-logo-wide.svg HTTP/1.1\" 200 27530\n192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] \"GET /bg-button.png HTTP/1.1\" 200 713\n192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] \"GET /bg-middle.png HTTP/1.1\" 200 1918\n192.168.10.1 - - [11/Feb/2023:19:20:31 -0800] \"GET /favicon.ico HTTP/1.1\" 200 21630",
      "@version" => "1",
    "@timestamp" => 2023-02-12T03:51:54.920Z,
           "log" => {
        "offset" => 0,
          "file" => {
            "path" => "/home/ELK/tomcat/apache-tomcat-10.0.27/logs/localhost_access_log.2023-02-11.txt"
        },
         "flags" => [
            [0] "multiline"
        ]
    },
          "tags" => [
        [0] "beats_input_codec_plain_applied"
    ],
         "input" => {
        "type" => "log"
    },
         "agent" => {
        "ephemeral_id" => "82569f6e-16d4-4a5c-9372-06d83d5a5d04",
             "version" => "7.6.1",
                  "id" => "902b638f-fcd1-44b2-8a81-5cfae937cbef",
                "type" => "filebeat",
            "hostname" => "localhost.localdomain"
    },
           "ecs" => {
        "version" => "1.4.0"
    },
          "host" => {
        "name" => "localhost.localdomain"
    }
}
```
