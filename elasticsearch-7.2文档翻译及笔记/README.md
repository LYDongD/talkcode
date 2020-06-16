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

#### 按条件更新（UPDATE BY QUERY API)

该api最简单的用法如下，在不更新_source的情况下，匹配文档最新的映射，例如使新加的字段
根据新的mappings进行索引。

```
#conflics=proceed 表示当发生版本冲突时，依然执行更新
POST twitter/_update_by_query?conflicts=proceed


#按条件同步mappings
POST twitter/_update_by_query?conflicts=proceed
{
  "query": { 
    "term": {
      "user": "kimchy"
    }
  }
}

```

此类更新将返回：

```
{
  "took" : 147, //耗时ms
  "timed_out": false, //是否超时
  "updated": 120, //已更新的文档数
  "deleted": 0, //已删除的文档数
  "batches": 1, //scoll search 的批数
  "version_conflicts": 0, //冲突的版本号
  "noops": 0, //ctx.op=noop（不进行更改）的文档数量
  "retries": { 
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0, //因为限流等待的总时长
  "requests_per_second": -1.0, //qps限制
  "throttled_until_millis": 0, // 总是0，在task api中表示下一次限流的等待时间
  "total": 120, //已处理的文档数
  "failures" : [ ] //失败说明，请求因为失败终止退出时该数组不为空
}
```

> 版本校验

_update_by_query 会为索引生成快照，并记录一个内部的版本号。当前请求执行更新索引时，如果
文档被其他请求修改，版本发生变化，将会导致版本冲突。如果不希望进行版本校验，或因为版本校验
失败导致请求终止，可以设置参数conflicts=proceed，例如前面的示例，如果只是同步最新的mappings,
版本冲突不会产生副作用，因此可以跳过它的检测。

> 失败处理

api发生失败时（例如版本冲突），响应结果中将包好failures字段进行说明。api是非原子性的，它通过scroll query（分批），然后
batch update的方式执行更新，后面批次的失败不会回滚前面已经执行了更新的批次，但是会终止请求，
导致后面的批次无法执行。


> 按条件更新文档的_source

使用脚本更新

```
POST twitter/_update_by_query
{
  "script": {
    "source": "ctx._source.likes++",
    "lang": "painless"
  },
  "query": {
    "term": {
      "user": "kimchy"
    }
  }
}

```

script支持动态的修改文档的**操作(ctx.op)**

* noop -> 不修改文档
* delete -> 删除文档

> 特性

1 支持多索引同时更新

```
POST twitter,blog/_update_by_query

```

2 支持通过routing指定更新特定的分片

```
POST twitter/_update_by_query?routing=1

```

3 可设置每个查询批次的大小，默认是100

```
POST twitter/_update_by_query?scroll_size=100

```

4 可以配合ingest功能使用

```

//创建pipeline，包含一个处理器，用于添加字段
PUT _ingest/pipeline/set-foo
{
  "description" : "sets foo",
  "processors" : [ {
      "set" : {
        "field": "foo",
        "value": "bar"
      }
  } ]
}

//更新之前先交给指定的pipeline处理
POST twitter/_update_by_query?pipeline=set-foo

```

> 通用参数支持

* refresh 

更新完成后将刷新所有分片，它和update api不一样，update api仅刷新发生了更新操作的特定分片，且不支持
wait_for相关参数

* wait_for_completion

如果它被设置为false, 则请求采用异步模式，返回一个task对象。task对象可被task api进一步处理，例如
取消任务。

* wait_for_active_shards

该参数要求指定数量的分片活跃时才执行更新，如果不满足，则进行等待；可以通过timeout设置等待时间。

* scroll

_update_by_query 使用scoll search 方式进行搜索，因此可以通过scroll参数控制**搜索上下文的存活时间*，
例如scroll=10m, 默认是5min

* requests_per_second

该参数可以被设置为任意正数的decimal数值，例如1.4，6, 该参数用于限流（qps)，该请求的每个批次之间
因为它的限制可能需要故意等待一段时间。默认值-1，表示关闭该限流功能。

例如，设置qps=500, 一个批次请求为1000，则该批次的限制延时为：1000 / 500 = 2s
如果该批次0.5s就处理完毕，需要等待 2 - 0.5 = 1.5 再处理下一批次

意味着，batch size 越大，因为限流的关系需要执行更长的时间，意味着需要等待更长的时间，这使得
请求曲线不够平滑。 

> 异步，和task api配合使用

当设置参数wait_for_completion=false，请求采用异步，返回task对象。

**获取task当前状态：**

```
GET _tasks?detailed=true&actions=*byquery

GET /_tasks/r1A2WoRbTwKZ516z6NEs5A:36619

```

响应结果：

```
{
  "nodes" : {
    "r1A2WoRbTwKZ516z6NEs5A" : {
      "name" : "r1A2WoR",
      "transport_address" : "127.0.0.1:9300",
      "host" : "127.0.0.1",
      "ip" : "127.0.0.1:9300",
      "attributes" : {
        "testattr" : "test",
        "portsfile" : "true"
      },
      "tasks" : {
        "r1A2WoRbTwKZ516z6NEs5A:36619" : {
          "node" : "r1A2WoRbTwKZ516z6NEs5A",
          "id" : 36619,
          "type" : "transport",
          "action" : "indices:data/write/update/byquery",
          "status" : {  //任务状态   
            "total" : 6154, // 期望执行的文档数 (实际完成 = 期望值时任务结束）
            "updated" : 3500, //已完成更新的文档数
            "created" : 0,
            "deleted" : 0,
            "batches" : 4,
            "version_conflicts" : 0,
            "noops" : 0,
            "retries": {
              "bulk": 0,
              "search": 0
            },
            "throttled_millis": 0
          },
          "description" : ""
        }
      }
    }
  }
}

```

**取消任务**

```
POST _tasks/r1A2WoRbTwKZ516z6NEs5A:36619/_cancel

```

取消任务可能很快也可能花费数s, 当使用task status api查看任务状态时，除非
已经检测到任务被取消且终止，否则将会继续显示该任务。异步请求的开销是需要
在本地磁盘创建一个任务文档：.tasks/task/${taskId}

**任务重新调整qps**

```
//解除流控
POST _update_by_query/r1A2WoRbTwKZ516z6NEs5A:36619/_rethrottle?requests_per_second=-1

```

> 切片，并发的执行scoll search

api 支持 slice scoll 机制以并发的执行整个更新流程，它将请求切分为更小的子请求并提高吞吐量。

* 手动切片 

分成2片：

```
POST twitter/_update_by_query
{
  "slice": {
    "id": 0,
    "max": 2
  },
  "script": {
    "source": "ctx._source['extra'] = 'test'"
  }
}
POST twitter/_update_by_query
{
  "slice": {
    "id": 1,
    "max": 2
  },
  "script": {
    "source": "ctx._source['extra'] = 'test'"
  }
}

//验证2个切片执行成功
GET _refresh
POST twitter/_search?size=0&q=extra:test&filter_path=hits.total

```

* 自动切片

指定切成5片，并发度为5

```
POST twitter/_update_by_query?refresh&slices=5
{
  "script": {
    "source": "ctx._source['extra'] = 'test'"
  }
}

```
如果设置切片数：slices=auto，es的计算策略是根据分片数：
count = min(indexs.shards.count)

请求被切片后，分解为多个子请求：

1. 在task api中，子请求以子任务的形式存在
2. 获取任务状态时，仅返回已完成的切片的子任务状态
3. 可以对子任务分别进行限流或取消操作
4. 对父任务修改限流配置时，仅对那些还未执行完毕的子任务生效
5. 取消请求时，会取消每个子请求
6. 每个切片对应的文档数量不是均衡的，有的切片大一点
7. 像requests_per_second和size这类参数，会按比例分配给子请求
8. 每个子请求生成的索引快照有点不同，因为生成的时间有微小的差异

**如何选择合理的切片数？**

建议将slices设置为auto，交给es进行决策；否则，建议切片数=分片数。
因为如果切片数<分片数，会降低查询性能；如果切片数>分片数，也无法
推升查询性能。而更新性能和可用资源线性相关。

无论是查询还是更新为主，它的性能总是由文档数量和集群资源决定。

> 示例：通过update_by_query 同步最新的mappings

```
PUT test
{
  "mappings": {
    "dynamic": false,   
    "properties": {
      "text": {"type": "text"}
    }
  }
}

POST test/_doc?refresh
{
  "text": "words words",
  "flag": "bar"
}
POST test/_doc?refresh
{
  "text": "words words",
  "flag": "foo"
}

//由于flag未添加mappings，它不会被索引，不支持根据flag搜索

//更新mappings
PUT test/_mapping   
{
  "properties": {
    "text": {"type": "text"},
    "flag": {"type": "text", "analyzer": "keyword"}
  }
}

//同步mappings
POST test/_update_by_query?refresh&conflicts=proceed

同步mappings后字段flag被索引

```



