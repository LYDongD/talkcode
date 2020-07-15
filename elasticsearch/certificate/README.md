## ES死磕

#### cross cluster search 跨集群搜索

> 文档

[Modules -> cross-cluster search](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/modules-cross-cluster-search.html)

> 问题

* 节点2启动kibana后es进程退出
    * 重新启动es失败，无异常日志
    * 查看系统错误日志：dmesg -T
        * 发现有java进程发生OOM，说明es进程因为oom被系统杀死

> 实现

1. 通过cluster update setting api 设置集群2关联的远程集群

要求重启有效，因此设置为persistent

```
PUT /_cluster/settings
{
  "persistent": {
    "cluster": {
      "remote" : {
        "original" : {
          "seeds" : ["172.18.155.128:9300"] //远程集群的transport端口
        }
      }
    }
  }
}

#查看配置确认
GET /_cluster/settings

```

2. 在集群2执行跨集群搜索

同时搜索两个集群的两个不同索引

```
GET /original:hamlet,hamlet-pirate/_search
{
  "query": {
    "match": {
      "speaker": "BERNARDO" 
    }
  } 
}

```

#### 文档操作

[document api](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/docs.html)

> 问题

添加文档，报错：

```
{
  "error": {
    "root_cause": [
      {
        "type": "cluster_block_exception",
        "reason": "index [hamlet-raw] blocked by: [FORBIDDEN/12/index read-only / allow delete (api)];"
      }
    ],
    "type": "cluster_block_exception",
    "reason": "index [hamlet-raw] blocked by: [FORBIDDEN/12/index read-only / allow delete (api)];"
  },
  "status": 403
}

```

日志：

```
[2020-07-13T19:47:34,483][INFO ][o.e.c.r.a.DiskThresholdMonitor] [node-1] low disk watermark [85%] exceeded on [zIHuCzhLS6acQfAKdW_PSQ][node-1][/home/elastic/data2/nodes/0] free: 2.4gb[12.7%], replicas will not be assigned to this node

```

原因：系统磁盘容量不足，可用内存低于es的期望值，则将索引转化为只读模式

解决方案：

```
PUT /_settings
{
  "index.blocks.read_only_allow_delete" : null
}

``` 

> 操作

1 创建索引：hamlet-raw

```
PUT hamlet-raw
{
  "settings": {
    "number_of_shards": 1, 
    "number_of_replicas": 3
  }
}

```

2 创建文档

```
#指定文档id
POST /hamlet-raw/_doc/1
{
  "line" : "To be, or not to be: that is the question"
}

#自动生成id
POST /hamlet-raw/_doc
{
  "text_entry":"Whether tis nobler in the mind to suffer",
  "line_number":"3.1.66"
}
```

3 更新或添加新字段

```
#partial doc 模式
POST /hamlet-raw/_update/1
{
  "doc" : {
    "line_number" : "3.1.64"  
  }
}

#script 模式
POST /hamlet-raw/_update/LQZHSHMB7W4lefW00iRy 
{
  "script" : "ctx._source.line_number='3.1.5'"
}

#批量操作多个文档
POST _bulk
{"update" : {"_id" : 1, "_index" : "hamlet-raw"}}
{"script" : "ctx._source.speaker='hamlet'"}
{"update" : {"_id" : "LQZHSHMB7W4lefW00iRy","_index" : "hamlet-raw"}}
{"script" : "ctx._source.speaker='hamlet'"}

```

4 查找更新

```

#修改所有文档
POST /hamlet-raw/_update_by_query
{
  "script" : {
    "lang" : "painless",
    "source": "ctx._source.speaker2='hamlet'"
  }
}

```

5 替换字段名

```
#创建ingest pipeline，指定rename processor
POST /hamlet-raw/_update_by_query?pipeline=hamlet-raw
{
  "query" : {
    "term" : {
      "_id" : 1
    }
  }
}

#应用在指定索引
POST /hamlet-raw/_update_by_query?pipeline=hamlet-raw

```

6 保存脚本并应用与查找更新操作

```

#创建脚本，增加字段并按条件赋值
POST _scripts/set_is_hamlet
{
  "script" : {
    "lang": "painless",
    "source": "if (ctx._source.speaker == 'HAMLET') {ctx._source.is_hamlet = true} else {ctx._source.is_hamlet = false}"
  }
}

#应用脚本
POST hamlet/_update_by_query
{
  "script" : {
    "id" : "set_is_hamlet"
  }
}

```
