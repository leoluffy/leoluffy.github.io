---
layout: post
category : ELK
title: '基于ELK+Kafka的错误/异常告警服务'
tagline: ""
tags : [elk,kafka]
---
* auto-gen TOC:
{:toc}

## 业务架构图
![](/images/post/alert-structure.png)
## 业务需求
1. 各业务系统统一上报错误/异常日志到日志收集服务集群
2. 监控这些错误日志，做一个统一的告警服务
3. 因为不想造轮子，所以打算使用较为成熟的ELK开源方案，日志收集这部分我们自己做了，ELK主要用于告警服务的日志检索服务

<!--break-->

## ELK版本选择
本来打算安装最新的release版本，因为新版中的x-pack，支持的功能更加丰富：Security, alerting, monitoring, reporting, machine learning & Graph   
但由于我们的kafka的版本是0.9.0，而最新版的logstash-input-kafka组件不兼容kafka的0.9.0版本，所以最终决定和公司运维部使用统一的版本

```
logstash：2.3.4  
elasticsearch：2.4.1      
kibana：4.5.4
```

## 安装ELK
安装比较简单，具体可以查阅官方的安装文档：[https://www.elastic.co/downloads](https://www.elastic.co/downloads)    
安装好elasticsearch，并启动后，执行curl http://localhost:9200/，会得到以下信息

```
{
  "name" : "Aftershock",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "mmKhTViiQXyNzPYL9UXcrw",
  "version" : {
    "number" : "2.4.1",
    "build_hash" : "c67dc32e24162035d18d6fe1e952c4cbcbe79d16",
    "build_timestamp" : "2016-09-27T18:57:55Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.2"
  },
  "tagline" : "You Know, for Search"
}
```

## 安装logstash的kafka插件
由于我们需要从kafka拉取数据，所以需要安装logstash的logstash-input-kafka插件

```
./bin/logstash-plugin install logstash-input-kafka  
```

## logstash配置
```
input {
        kafka{
            zk_connect => "zookeeper.xxx.com:2181" //zookeeper的地址
            group_id => "logstash" //kafka consumer的group_id
            topic_id => "xxx" //kafka的topic
            consumer_threads => 5  
            decorate_events => true
        }
        
        kafka{ //配置多个topic
            zk_connect => "zookeeper.xxx.com:2181" //zookeeper的地址
            group_id => "logstash" //kafka consumer的group_id
            topic_id => "xxx" //kafka的topic
            consumer_threads => 5  
            decorate_events => true
        }
        
}

output {
        elasticsearch { //写入到es
            hosts => ["127.0.0.1:9200"]
            index => "logstash-app-error-%{+YYYY.MM.dd}"
       }
       stdout { codec => rubydebug } //输出到终端
}

```

## 启动logstash

```
//  --debug参数为调试模式，可以看到一些比较详细的错误信息
/install_dir/bin/logstash --debug -f /conf_dir 
```
此时如果能正常读取到kafka的数据，会在终端打印相关数据，也可以通过执行以下方式来查询es里面的数据    

```
curl http://localhost:9200/logstash-app-error-*/_search?pretty
```

## 使用es的watcher插件做告警服务
### 安装watcher

```
bin/plugin install license
bin/plugin install watcher
```
### 我们的消息body内容

```
{
    "logs": {
        "app_id": 123456,
        "child_app": "admin",
        "client_ip": "xxx.xxx.xxx.xxx",
        "detail": "this is error detail",
        "ip": "xxx.xxx.xxx.xxx",
        "level": 2,
        "mtime": 1512961547,
        "related_app_id": 18,
        "request_id": 1,
        "summary": "this is error summary"
    }
}
```
### 监控告警需求
每分钟扫描前一分钟的错误日志，统计某个related_app_id的某个child_app类型的某种summary错误内容的发生总次数，然后组织告警信息，例如：

```
【related_app_id-child_app】发生COUNT次错误告警：
summary

```
### 新增一个watcher配置

```
curl -XPUT 'http://localhost:9200/_watcher/watch/app_error_watch' -d '{
  "trigger" : { "schedule" : { "interval" : "1m" } },
  "input" : {
    "search" : {
      "request" : {
        "indices" : [ "logstash-app-error-*" ],
        "body" : {
          "query" : {
            "bool": {
                "must": [
                  {
                    "range": {
                        "@timestamp": {
                            "gt" : "now-1m"
                        }
                    }
                  }
                ]
            }
          },
          "size": 0,
          "aggregations": { 
            "group_by_related_app_id": { 
              "terms": { 
                "field": "logs.related_app_id" 
              },
              "aggs": {
                "group_by_child_app": {
                  "terms": {
                    "field": "logs.child_app.raw"
                  },
                  "aggs": {
                    "group_by_summary": {
                      "terms": {
                        "field": "logs.summary.raw"
                      }
                    }
                  }
                }
              }
            } 
          } 
        }
      }
    }
  },
  "condition" : {
    "array_compare": {
      "ctx.payload.aggregations.group_by_related_app_id.buckets" : { 
        "path": "doc_count" ,
        "gt": { 
          "value": 0, 
          "quantifier": "some" 
        }
      }
    }
  },
  "actions" : {
    "log_error" : {
      "logging" : {
        "text" : "{{#toJson}}ctx.payload.aggregations{{/toJson}}"
      }
    },
    "notify_pager" : {
      "webhook" : {
        "method" : "POST",
        "host" : "alert.xxx.com",
        "port" : 80,
        "path" : "/alerter",
        "body" : "{{#toJson}}ctx.payload.aggregations{{/toJson}}"
      }
    }
  }
}'
```

### watcher配置详解
主要分为几大块：trigger、input、condition、actions   
trigger：触发器，可以设置扫描日志的频率   
input：数据来源/输入   
condition：设置触发action的条件   
action：设置告警方式   
#### 聚合运算
以下配置类似于sql语句：   
select related_app_id,child_app,summary,count(1) from table group by related_app_id,child_app,summary      
aggregations配置参考资料：   https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html

```
{
    "aggregations": {
        "group_by_related_app_id": {
            "terms": {
                "field": "logs.related_app_id"
            },
            "aggs": {
                "group_by_child_app": {
                    "terms": {
                        "field": "logs.child_app.raw"
                    },
                    "aggs": {
                        "group_by_summary": {
                            "terms": {
                                "field": "logs.summary.raw"
                            }
                        }
                    }
                }
            }
        }
    }
}

```

#### 告警触发条件
以下配置表示只要在related_app_id维度的聚合分组中，只要有某个分组的count大于0，则条件成立

```
{
"condition" : {
    "array_compare": {
      "ctx.payload.aggregations.group_by_related_app_id.buckets" : { 
        "path": "doc_count" ,
        "gt": { 
          "value": 0, 
          "quantifier": "some" 
        }
      }
    }
  }
}
```
#### 为什么要使用logs.summary.raw

在使用log.summary时，es会产生以下报错

```
nested: IllegalStateException[Field data loading is forbidden on [logs.summary]];
```
加上.raw可以解决该问题，详细内容可以查看：   [https://github.com/elastic/elasticsearch/issues/15267](https://github.com/elastic/elasticsearch/issues/15267)

#### 结果集转换成json格式
默认的结果集格式是java的map格式，对于其他语言来说不好解析，这种情况一般的做法是转换成像json这样的中间语言，但是在官方文档好像找不到相关内容，google之后，在github的elasticsearch仓库里看到了转换成json的相关代码和说明：   
[https://github.com/elastic/elasticsearch/pull/19153/files#diff-b99a30a3f37550a69a2508b3a3e120a6R226](https://github.com/elastic/elasticsearch/pull/19153/files#diff-b99a30a3f37550a69a2508b3a3e120a6R226)

### 查看app_error_watch配置

```
curl 'http://localhost:9200/_watcher/watch/app_error_watch'
```

### 删除app_error_watch配置

```
curl -XDELETE 'http://localhost:9200/_watcher/watch/app_error_watch'
```
## 使用supervisord管理ELK服务
### 安装supervisord
[查阅官方安装文档](http://www.supervisord.org/installing.html)
### 配置管理ELK服务

```
[unix_http_server]
file=/tmp/supervisor.sock

[supervisord]
logfile=/data/logs/supervisord.log ; (main log file;default $CWD/supervisord.log)
logfile_maxbytes=50MB        ; (max main logfile bytes b4 rotation;default 50MB)
logfile_backups=10           ; (num of main logfile rotation backups;default 10)
loglevel=info                ; (log level;default info; others: debug,warn,trace)
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
nodaemon=false               ; (start in foreground if true;default false)
minfds=65535                  ; (min. avail startup file descriptors;default 1024)
minprocs=200                 ; (min. avail process descriptors;default 200)

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///tmp/supervisor.sock

[program:logstash]
command = /data/webapp/logstash/bin/logstash -f /data/conf/logstash/
environment=ES_HEAP_SIZE=8g
numprocs = 1
autostart = true
autorestart = true
redirect_stderr = true
stdout_logfile=/data/logs/elk/logstash.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=5
stdout_capture_maxbytes=10MB
stderr_logfile=/data/logs/elk/logstash_error.log
stderr_logfile_maxbytes=50MB
stderr_logfile_backups=5
stderr_capture_maxbytes=10MB

[program:es]
command = /data/webapp/elasticsearch/bin/elasticsearch
environment=ES_HEAP_SIZE=30g,ES_JAVA_OPTS="-Dmapper.allow_dots_in_name=true"
numprocs = 1
autostart = true
autorestart = true
redirect_stderr = true
stdout_logfile=/data/logs/elk/es.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=5
stdout_capture_maxbytes=10MB
stderr_logfile=/data/logs/elk/es_error.log
stderr_logfile_maxbytes=50MB
stderr_logfile_backups=5
stderr_capture_maxbytes=10MB
user=elasticsearch

[program:kb]
command = /data/webapp/kibana/bin/kibana -H  127.0.0.1
numprocs = 1
autostart = true
autorestart = true
redirect_stderr = true
stdout_logfile=/data/logs/elk/kibana.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=5
stdout_capture_maxbytes=10MB
stderr_logfile=/data/logs/elk/kibana_error.log
stderr_logfile_maxbytes=50MB
stderr_logfile_backups=5
stderr_capture_maxbytes=10MB
user=elasticsearch
```
### 启动supervisord

```
service supervisord start
```

### 通过supervisorctl工具查看和管理ELK服务
```
[username@xxx.xxx.xxx.xxx_A ~]# supervisorctl
es                               RUNNING   pid 7215, uptime 2 days, 17:01:17
kb                               RUNNING   pid 7213, uptime 2 days, 17:01:17
logstash                         RUNNING   pid 7954, uptime 2 days, 17:00:02
supervisor> 
```
更多关于supervisord的内容可以查阅官方文档：   
[http://www.supervisord.org/](http://www.supervisord.org/)


字符串超过默认值256不会被索引
https://www.elastic.co/guide/en/elasticsearch/reference/2.4/ignore-above.html#ignore-above

indices-put-mapping
https://www.elastic.co/guide/en/elasticsearch/reference/2.4/indices-put-mapping.html

修改mapping配置

```
curl -XPUT 'localhost:9200/logstash-app-error-*/_mapping/logs?pretty' -H 'Content-Type: application/json' -d'
{
  "properties": {
    "logs": {
      "properties" : {
        "detail" : {
          "type" : "string",
          "norms" : {
            "enabled" : false
          },
          "fielddata" : {
            "format" : "disabled"
          },
          "fields" : {
            "raw" : {
              "type" : "string",
              "index" : "not_analyzed",
              "ignore_above" : 32766
            }
          }
        },
        "summary" : {
          "type" : "string",
          "norms" : {
            "enabled" : false
          },
          "fielddata" : {
            "format" : "disabled"
          },
          "fields" : {
            "raw" : {
              "type" : "string",
              "index" : "not_analyzed",
              "ignore_above" : 32766
            }
          }
        }
      }
    }
  }
}
'
```

### 创建indices-templates模板
https://www.elastic.co/guide/en/elasticsearch/reference/2.4/indices-templates.html

```
curl -XPUT 'localhost:9200/_template/logstash-app-error?pretty' -H 'Content-Type: application/json' -d'
{
    "order": 1,
    "template": "logstash-app-error-*",
    "settings": {
        "index": {
            "refresh_interval": "5s"
        }
    },
    "mappings": {
        "_default_": {
            "dynamic_templates": [
                {
                    "message_field": {
                        "mapping": {
                            "fielddata": {
                                "format": "disabled"
                            },
                            "index": "analyzed",
                            "omit_norms": true,
                            "type": "string"
                        },
                        "match_mapping_type": "string",
                        "match": "message"
                    }
                },
                {
                    "string_fields": {
                        "mapping": {
                            "fielddata": {
                                "format": "disabled"
                            },
                            "index": "analyzed",
                            "omit_norms": true,
                            "type": "string",
                            "fields": {
                                "raw": {
                                    "ignore_above": 32766,
                                    "index": "not_analyzed",
                                    "type": "string"
                                }
                            }
                        },
                        "match_mapping_type": "string",
                        "match": "*"
                    }
                }
            ],
            "_all": {
                "omit_norms": true,
                "enabled": true
            },
            "properties": {
                "@timestamp": {
                    "type": "date"
                },
                "geoip": {
                    "dynamic": true,
                    "properties": {
                        "ip": {
                            "type": "ip"
                        },
                        "latitude": {
                            "type": "float"
                        },
                        "location": {
                            "type": "geo_point"
                        },
                        "longitude": {
                            "type": "float"
                        }
                    }
                },
                "@version": {
                    "index": "not_analyzed",
                    "type": "string"
                }
            }
        }
    },
    "aliases": {}
}
'
```