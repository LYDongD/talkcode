## elasticsearch-7.2 文档阅读笔记

### 文档API (Document API)

#### 按条件删除API (Delete By Query API)

> 基础使用

删除满足query查询条件的文档

```

#删除单个索引
POST twitter/_delete_by_query
{
  "query": { 
    "match": {
      "message": "some message"
    }
  }
}

#删除多个索引
POST twitter,blog/_delete_by_query
{
  "query": {
    "match_all": {}
  }
}


#删除指定分片组上的索引
POST twitter/_delete_by_query?routing=1
{
  "query": {
    "range" : {
        "age" : {
           "gte" : 10
        }
    }
  }
}

```

也可以采用query string parameter风格（q=xxx), 和search api用法相同

返回值：

```
{
  "took" : 147, //耗时ms
  "timed_out": false, //是否有请求超时
  "deleted": 119, //删除的文档数
  "batches": 1, //删除的批次数，例如scroll_size=1, deleted=3, 则batches = 3 / 1= 3
  "version_conflicts": 0, //发送版本冲突的版本号
  "noops": 0, 
  "retries": {
    "bulk": 0, //bulk delete 的重试次数
    "search": 0 //query search 的重试次数
  }, 
  "throttled_millis": 0, //限流等待的时间
  "requests_per_second": -1.0, //qps， -1不限流
  "throttled_until_millis": 0, //永远为0，使用task api时该值有意义
  "total": 119, //满足删除条件的文档总数
  "failures" : [ ] //请求因为不可恢复错误中断时的错误信息
}

```

**关于version_conflicts:**
删除文档时，es会获取基于版本的一个索引的快照(snapshot), 意味着在获取snapshot之后，
文档更新(version被更新）之前删除时，会引发version conflict。只有版本匹配时才能
完成删除操作。

如果想在发送版本冲突时依然执行删除，可以通过参数：conflicts=proceed 强制执行：

```
POST twitter/_delete_by_query?conflicts=proceed
{
  "query": {
    "match_all": {}
  }
}

```

> 执行过程

_delete_by_query执行过程中，多条请求(scroll search)被**串行**的发送以查找满足条件的文档，当一个批次
(scroll_size)的文档被找到, 将会通过bulk 批量请求删除这些文档。如果查询或删除请求被拒绝，
默认失败处理机制是：重试。重试至多10次，每次的间隔时间指数型衰减。当重试次数达到最大值
后，请求退出，返回失败信息。

注意，查询删除是分批进行的，它并不是原子性的，因此失败的批次不会回滚前面成功的批次，也不会
继续执行剩下的批次。


另外，我们可以指定每次删除批次的大小：scroll_size

```
POST twitter/_delete_by_query?scroll_size=5000
{
  "query": {
    "term": {
      "user": "kimchy"
    }
  }
}

```

如果scroll_size过大，意味着单批次请求时间被拉长，导致整体看起来不够**平滑**

> 其他URL可选参数

* refresh -> 删除请求完成后会刷新**所有**分片
    * 和delete不同，它只刷新接收delete请求的分片
* wait_for_active_shards -> 等待指定数量的活跃副本就绪
    * timeout -> 等待超时时间
* scroll -> 指定scroll search的最大时间
    * 默认5min
* requests_per_second -> qps流量限制
    * -1 表示不限制，其他正数的decimal类型则进行限制，例如1.5或12
    * 假设一批次查询1000条（scroll_size=1000), qps=500，意味着一批次请求至少2s，如果0.5s执行结束，下一批次需要等待1.5s
        * target_time = 1000 / 500 = 2
        * wait_time = target_time - real_execute_time = 2 - 0.5 = 1.5s
    * qps可以在请求执行时重新设置(前提是wait_for_completion=false）
        * POST _delete_by_query/r1A2WoRbTwKZ516z6NEs5A:36619/_rethrottle?requests_per_second=-1 
* wait_for_completion -> 是否等待删除操作完成
    * 设置成false -> 将返回一个taskId （异步）
        * 可以使用task api对task进一步处理
            * 查看task执行情况
                * GET _tasks?detailed=true&actions=*/delete/byquery
                * GET /_tasks/r1A2WoRbTwKZ516z6NEs5A:36619
            * 取消执行中的task
                * POST _tasks/r1A2WoRbTwKZ516z6NEs5A:36619/_cancel
            * 重新调整task的限流规则
                * POST _delete_by_query/r1A2WoRbTwKZ516z6NEs5A:36619/_rethrottle?requests_per_second=-1
        * 如果task执行结束，它将被删除

> 对scroll请求进行分片并行处理(分治）

将scorll search 分解成多个子请求(sub_requests)，这些请求并发执行

* 手动切片

删除时指定切片参数（切片个数和编号）

```

#删除切片1
POST twitter/_delete_by_query
{
  "slice": {
    "id": 0, //切片编号
    "max": 2 //切片数量
  },
  "query": {
    "range": {
      "likes": {
        "lt": 10
      }
    }
  }
}

##删除切片2
POST twitter/_delete_by_query
{
  "slice": {
    "id": 1,
    "max": 2
  },
  "query": {
    "range": {
      "likes": {
        "lt": 10
      }
    }
  }
}

```

注意，每个切片的文档数不一定相同，即存在较大和较小的切片

* 自动切片

通过slices指定切片个数，将自动切片并完成并发请求

```
POST twitter/_delete_by_query?refresh&slices=2
{
  "query": {
    "range": {
      "likes": {
        "lt": 10
      }
    }
  }
}

```
返回切片及删除信息

```
{
  "took" : 13,
  "timed_out" : false,
  "total" : 12,
  "deleted" : 12,
  "batches" : 2,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "slices" : [
    {
      "slice_id" : 0,
      "total" : 6,
      "deleted" : 6,
      "batches" : 1,
      "version_conflicts" : 0,
      "noops" : 0,
      "retries" : {
        "bulk" : 0,
        "search" : 0
      },
      "throttled_millis" : 0,
      "requests_per_second" : -1.0,
      "throttled_until_millis" : 0
    },
    {
      "slice_id" : 1,
      "total" : 6,
      "deleted" : 6,
      "batches" : 1,
      "version_conflicts" : 0,
      "noops" : 0,
      "retries" : {
        "bulk" : 0,
        "search" : 0
      },
      "throttled_millis" : 0,
      "requests_per_second" : -1.0,
      "throttled_until_millis" : 0
    }
  ],
  "failures" : [ ]
}

```

设置slices=auto, 让es决定切片数，默认一个分片一个切片, 如果要删除多个索引，
切片数 = min(index分片，index2分片...)

* 切片数选择多少合适？
  * query 性能: 
    * 切片 < 分片 -> 性能下降
    * 切片 = 分片 -> 性能最佳
    * 切片 > 分片 -> 性能提升不大
  * delete 性能：
    * 理论上切片阅多，删除越快(并行）
  * 综上，切片数应该设置为分片数，这也是auto的设定

#### 文档更新API （Update API)

> 更新操作

update = get + reindex

更新操作本质上是搜索并重新索引，因此它事实上可以实现增删改的逻辑操作；

从实现上因为合并了两个操作，因此可以节省网络开销，避免版本冲突;

另外，update操作，不允许关闭_source选项。

* 基于脚本更新

```
#修改number类型字段
POST test/_update/1
{
  "script" : {
    "source" :"ctx._source.counter+=params.count",
    "lang":"painless", //脚本语言
    "params":{
      "count":4
    }
  }
}

#数组类型字段增加元素
POST test/_update/1
{
  "script" : {
    "source" :"ctx._source.tags.add(params.tag)",
    "lang":"painless",
    "params":{
      "tag":"pink"
    }
  }
}

#数组类型删除指定元素
POST test/_update/1
{
  "script" : {
    "source": "if (ctx._source.tags.contains(params.tag)) {ctx._source.tags.remove(ctx._source.tags.indexOf(params.tag))}",
    "lang" : "painless",
    "params": {
      "tag" : "red"
    }
  }
}

#增加字段
POST test/_update/1
{
    "script" : "ctx._source.new_field = 'value_of_new_field'"
}

#删除字段
POST test/_update/1
{
    "script" : "ctx._source.remove('new_field')"
}

#动态删除文档
POST test/_update/1
{
    "script" : {
        "source": "if (ctx._source.tags.contains(params.tag)) { ctx.op = 'delete' } else { ctx.op = 'none' }",
        "lang": "painless",
        "params" : {
            "tag" : "green"
        }
    }
}

```
脚本中上下文ctx可获取的属性：

* _source -> 要求文档_source必须启用
* _type
* _id
* _version
* _index
* _routing
* _now -> 时间戳

> 部分文档更新

以递归合并的方式进行更新, 如果想完全替换文档应该使用index api

```
#字段name不存在时，该请求会增加一个新的字段
POST test/_update/1
{
    "doc" : {
        "name" : "new_name"
    }
}

#如果字段存在，并且值相同，则不进行更新，返回noop
{
  "_index" : "test",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 5,
  "result" : "noop",
  "_shards" : {
    "total" : 0,
    "successful" : 0,
    "failed" : 0
  }
}

#可以关闭noop，即使key-value相同，也进行更新
POST test/_update/1
{
  "doc" :{
    "name" : "liam2"
  },
  "detect_noop" : "false"
}

```

如果_update api同时指定script和doc, 那么doc将被忽略，通常情况下，建议
使用script

> upsert 操作

如果更新的文档不存在，则执行插入

```
#script方式, 文档不存在则执行upsert,否则执行script
POST test/_update/1
{
    "script" : {
        "source": "ctx._source.counter += params.count",
        "lang": "painless",
        "params" : {
            "count" : 4
        }
    },
    "upsert" : {
        "counter" : 1
    }
}

#不在upsert中设置值, 无论文档是否存在都执行script
POST test/_update/6
{
  "scripted_upsert" : true,
  "script" : {
    "source": "ctx._source.counter = params.count",
    "lang": "painless",
    "params" : {
      "count" : 3
    }
  },
  "upsert" :{}
}

#partial doc方式
POST test/_update/3
{
  "doc" : {
    "name" : "miaomei"
  },
  "doc_as_upsert" : true
}

```

> 其他参数

* retry_on_conflicts -> 版本冲突后的重试次数
    * 在get和reindex操作之间，文档可能被修改导致版本冲突
* routing -> 路由到指定的分片
* timeout -> 等待active shard 的超时时间
* wait_for_active_shards -> 需要等待的活跃分片数
* refresh -> 更新后是否refresh，保证文档实时可见性
* _source -> 是否返回_source
    * _source = true -> 默认false
* version -> 指定文档的版本，匹配时才进行更新
* if_seq_no/if_primary_term
    * 匹配时才进行更新
