Title: logstash、elasticsearch和kibana3使用简介
Slug:log_analysis_witch_logstash_elasticserach_kibana3
Date: 2013-12-12
Category: 后台开发
Tags: log analysis, logstash, elasticsearch, kibana3
Author: 伊恩


##使用场景
日志集中收集，及实时统计分析

##系统架构
![系统架构](/images/zengyi/log_architecture.png)

Shipper: 分布式部署在各应用服务器，收集并转发日志。

Broker：将日志集中收集、排队并转发。

Indexer：收集和转发数据至ElasticSearch。在ElasticSearch建立索引并存储日志数据。

Web interface:基于nginx的Kibana3 http访问接口，提供UI搜索ElasticSearch的数据。

##系统配置
以nginx访问日志为例，配置如下：

1）Shipper.conf

```
input {
    file {
        type => "nginx"
        path => [ "/nginx/path/logs/access.log" ]
        start_position => "beginning"
        sincedb_path => "/some/place/sincedb_nginx"
      }
}

filter {
    grok {
        match => [ "message", "%{IP:client} (%{USER:indent}|-) (%{USER:auth}|-) \[%{HTTPDATE:local_time}\] \"%{WORD:method} (?<request_url>%{URIPATH})(?<request_params>%{URIPARAM}) HTTP/%{NUMBER:protocol}\" %{NUMBER:status} %{NUMBER:bytes_sent} %{QS:http_referer} %{QS:user_agent}" ]
    }

    date {
        locale => "en"
        match => [ "local_time", "dd/MMM/YYYY:HH:mm:ss Z" ]
        timezone => "Asia/Shanghai"
    }
}

output {
  redis {
    host => "192.168.1.130"
    port => 6379
    data_type => "list"
    key => "nginx"
  }
}
```

2) Indexer.conf
```
input {
    redis {
        host => "192.168.1.130"
        port => 6379
        # these settings should match the output of the agent
        data_type => "list"
        key => "nginx"
        codec => json
    }
}

output {
    elasticsearch_http {
        host => "192.168.1.130"
        index => "%{type}-%{+YYYY.MM.dd}"
        index_type =>"%{type}"
        flush_size => 1000
    }
}
```

##Kibana查询介绍

Kibana查询语法遵循[Lucence的语法规范](https://lucene.apache.org/core/3_5_0/queryparsersyntax.html)

常用的有以下几种

1）逻辑查询

```
操作符：AND（&&）， OR（||）， NOT（！）
优先级：！ > && > ||
默认情况下是或操作，如：field：（hello world）匹配field值为hello或world的事件
```
2）范围查询
```
支持类型：date，数值，字符串
闭区间：[min to max]  等价于 >=min && <= max
开区间：{min to max} 等价于 >min && <max
半开闭区间: [min to max} {min to max]

NOTE: 对于date和数值类型，min或max可以使用*
```
3）子查询
```
使用()，如: field：（hello world）,(hello world)为一子查询
```
4）通配符
```
？：匹配一个字符
*: 匹配0个或多个字符

NOTE：通配符会导致使用大量内存，从而降低响应速度，要慎用
```
5）保留字符转义
```
保留字符有：+ - && || ! ( ) { } [ ] ^ " ~ * ? : \ /
如果搜索条件中含有保留字符，使用\转义
```


