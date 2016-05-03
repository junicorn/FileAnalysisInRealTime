FileAnalysisInRealTime
----------------------

**Logstash+ElasticSearch+Kibana+Redis日志分析和监控工具**

**注意：关于安装文档，网络上有很多，可以参考，不可以全信，而且三件套各自的版本很多，差别也不一样，需要版本匹配上才能使用。推荐直接使用官网的这一套：elkdownloads。**
logstash 1.4.2 + elasticsearch 1.4.2 + kibana 3.1.2

安装elasticsearch
------------------

下载elasticsearch 1.4.2

```
tar -xf elasticsearch-1.4.2.tar.gz
mv elasticsearch-1.4.2 /usr/local/
ln -s /usr/local/elasticsearch-1.4.2 /usr/local/elasticsearch
```

 - 测试elasticsearch

```
[root@localhost service]# curl -X GET http://localhost:9200/
{
  "status" : 200,
  "name" : "Fury",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "1.4.2",
    "build_hash" : "927caff6f05403e936c20bf4529f144f0c89fd8c",
    "build_timestamp" : "2014-12-16T14:11:12Z",
    "build_snapshot" : false,
    "lucene_version" : "4.10.2"
  },
  "tagline" : "You Know, for Search"
}
```

 - 安装到自启动项

下载解压到/usr/local/elasticsearch/bin文件夹下

```
/usr/local/elasticsearch/bin/service/elasticsearch install
```

 

安装logstash
----------

 - 下载logstash 1.4.2

```
tar -xf logstash-1.4.2
mv logstash-1.4.2 /usr/local/
ln -s /usr/local/logstash-1.4.2 /usr/local/logstash
```

 - 测试logstash

```
bin/logstash -e 'input { stdin { } } output { stdout {} }'
```

 - 配置logstash

```
创建配置文件目录：
mkdir -p /usr/local/logstash/etc

vim /usr/local/logstash/etc/hello_search.conf
```

输入下面：

```
input {
  stdin {
    type => "human"
  }
}

output {
  stdout {
    codec => rubydebug
  }

  elasticsearch {
        host => "192.168.33.10"
        port => 9200
  }
}
```

启动：

```
/usr/local/logstash/bin/logstash -f /usr/local/logstash/etc/hello_search.conf
```

安装kibana
--------

注：logstash 1.4.2中也自带了kabana，但是你如果使用自带的kibana安装完之后会发现有提示“Upgrade Required Your version of Elasticsearch is too old. Kibana requires Elasticsearch 0.90.9 or above.”。

下载kibana 3.1.2
注：现在kibanna可以自带了web服务，bin/kibana就可以直接启动了，建议不用nginx进行配合启动了

logstash,elasticsearch,kibana 进行nginx的日志分析
------------------------------------------

首先，架构方面，nginx是有日志文件的，它的每个请求的状态等都有日志文件进行记录。其次，需要有个队列，redis的list结构正好可以作为队列使用。然后分析使用elasticsearch就可以进行分析和查询了。

一个分布式的，日志收集和分析系统。logstash有agent和indexer两个角色。对于agent角色，放在单独的web机器上面，然后这个agent不断地读取nginx的日志文件，每当它读到新的日志信息以后，就将日志传送到网络上的一台redis队列上。对于队列上的这些未处理的日志，有不同的几台logstash indexer进行接收和分析。分析之后存储到elasticsearch进行搜索分析。再由统一的kibana进行日志web界面的展示。

 - 准备工作

安装了redis,开启在6379端口
安装了elasticsearch, 开启在9200端口
安装了kibana, 开启了监控web
logstash安装在/usr/local/logstash
nginx开启了日志，目录为：/usr/share/nginx/logs/test.access.log
设置nginx日志格式
在nginx.conf 中设置日志格式：logstash

```
log_format logstash '$http_host $server_addr $remote_addr [$time_local] "$request" '
                    '$request_body $status $body_bytes_sent "$http_referer" "$http_user_agent" '
                    '$request_time $upstream_response_time';
```

在vhost/test.conf中设置access日志：

```
access_log  /usr/share/nginx/logs/test.access.log  logstash;
```

开启logstash agent
注：这里也可以不用logstash，直接使用rsyslog

创建logstash agent 配置文件

```
vim /usr/local/logstash/etc/logstash_agent.conf
```

代码如下：

```
input {
        file {
                type => "nginx_access"
                path => ["/usr/share/nginx/logs/test.access.log"]
        }
}
output {
        redis {
                host => "localhost"
                data_type => "list"
                key => "logstash:redis"
        }
}
```

启动logstash agent

```
/usr/local/logstash/bin/logstash -f /usr/local/logstash/etc/logstash_agent.conf
```

这个时候，它就会把test.access.log中的数据传送到redis中，相当于tail -f。

开启logstash indexer
创建 logstash indexer 配置文件

vim /usr/local/logstash/etc/logstash_indexer.conf
代码如下：

```
input {
        redis {
                host => "localhost"
                data_type => "list"
                key => "logstash:redis"
                type => "redis-input"
        }
}
filter {
    grok {
        match => [
            "message", "%{WORD:http_host} %{URIHOST:api_domain} %{IP:inner_ip} %{IP:lvs_ip} \[%{HTTPDATE:timestamp}\] \"%{WORD:http_verb} %{URIPATH:baseurl}(?:\?%{NOTSPACE:request}|) HTTP/%{NUMBER:http_version}\" (?:-|%{NOTSPACE:request}) %{NUMBER:http_status_code} (?:%{NUMBER:bytes_read}|-) %{QS:referrer} %{QS:agent} %{NUMBER:time_duration:float} (?:%{NUMBER:time_backend_response:float}|-)"
        ]
    }
    kv {
        prefix => "request."
        field_split => "&"
        source => "request"
    }
    urldecode {
        all_fields => true
    }
    date {
        type => "log-date"
        match => ["timestamp" , "dd/MMM/YYYY:HH:mm:ss Z"]
    }
}
output {
        elasticsearch {
                embedded => false
                protocol => "http"
                host => "localhost"
                port => "9200"
                index => "access-%{+YYYY.MM.dd}"
        }
}
```

**这份配置是将nginx_access结构化以后塞入elasticsearch中。**

对这个配置进行下说明：

*grok中的match正好匹配和不论是GET，还是POST的请求。
kv是将request中的A=B&C=D的key，value扩展开来，并且利用es的无schema的特性，保证了如果你增加了一个参数，可以立即生效
urldecode是为了保证参数中有中文的话进行urldecode
date是为了让es中保存的文档的时间为日志的时间，否则是插入es的时间
好了，现在的结构就完成了，你可以访问一次test.dev之后就在kibana的控制台看到这个访问的日志了。而且还是结构化好的了，非常方便查找。*

 - 使用kibana进行查看

依次开启es，logstash，kibana之后，可以使用es的head插件确认下es中有access-xx.xx.xx索引的数据，然后打开kibana的页面，第一次进入的时候会让你选择mapping，索引名字填写access-*，则kibana自动会创建mapping
