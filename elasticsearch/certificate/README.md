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
