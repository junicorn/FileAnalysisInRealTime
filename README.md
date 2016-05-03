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
