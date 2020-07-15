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

#### 批量读(MULTI GET API)

批量读接口可根据文档的索引，类型和id一次性返回多条文档。返回结果将包含一个docs数组，
数组包含查找到的文档或错误。

```
GET /_mget
{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "_doc",
            "_id" : "1"
        },
        {
            "_index" : "test",
            "_type" : "_doc",
            "_id" : "2"
        }
    ]
}

//可以将索引参数放到url上
GET /test/_mget
{
    "docs" : [
        {
            "_type" : "_doc",
            "_id" : "1"
        },
        {
            "_type" : "_doc",
            "_id" : "2"
        }
    ]
}

//可以将类型参数放到url上
GET /test/_doc/_mget
{
    "docs" : [
        {
            "_id" : "1"
        },
        {
            "_id" : "2"
        }
    ]
}

//可以合并id，简化请求参数
GET /test/_doc/_mget
{
    "ids" : ["1", "2"]
}

//针对返回结果_source进行过滤（前提是_source已被stored)
GET /_mget
{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "_doc",
            "_id" : "1",
            "_source" : false
        },
        {
            "_index" : "test",
            "_type" : "_doc",
            "_id" : "2",
            "_source" : ["field3", "field4"]
        },
        {
            "_index" : "test",
            "_type" : "_doc",
            "_id" : "3",
            "_source" : {
                "include": ["user"],
                "exclude": ["user.location"]
            }
        }
    ]
}

//仅stored特定字段
GET /_mget
{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "_doc",
            "_id" : "1",
            "stored_fields" : ["field1", "field2"]
        },
        {
            "_index" : "test",
            "_type" : "_doc",
            "_id" : "2",
            "stored_fields" : ["field3", "field4"]
        }
    ]
}

GET /test/_doc/_mget?stored_fields=field1,field2
{
    "docs" : [
        {
            "_id" : "1" 
        },
        {
            "_id" : "2",
            "stored_fields" : ["field3", "field4"] 
        }
    ]
}

```

_source 和 stored_field 的区别：

搜索的时候，可以指定从_source(source_filtering)或store(stored_fields)获取。
如果从store获取，需要保证mappings的该字段store属性设置为true

```
PUT test2
{
  "mappings": {
    "_source": {
      "enabled": false
    },
    "properties": {
      "key1": {
        "type": "keyword",
        "store": true
      },
      "key2": {
        "type": "text",
        "store": false
      }
    }
  }
}

//由于关闭了_source, 将无法返回source
GET test2/_search 

//由于key1的store设置为true，key2为false，将只返回key1
GET test2/_search?stored_fields=key1,key2

```

#### 批量写 (BULK API)

bulk api 使得一次请求可以处理多个索引或删除操作，提高了索引的速度。

api的请求体格式(ndjson)如下：

请求头中Content-Type必须为application/x-ndjson

1. 每行json用换行符分隔
2. 最后一行也需要以回车换行结尾

```
action_and_meta_data //操作，例如index/create/delete/update，和元数据
optional_source //可选的source，例如index/create时需要添加文档source
action_and_meta_data
optional_source
...
action_and_meta_data
optional_source


//实例
POST _bulk
{ "index" : { "_index" : "test", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_id" : "2" } }
{ "create" : { "_index" : "test", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }


//返回
{
   "took": 30,
   "errors": false,
   "items": [
      {
         "index": {
            "_index": "test",
            "_type": "_doc",
            "_id": "1",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 201,
            "_seq_no" : 0,
            "_primary_term": 1
         }
      },
      {
         "delete": {
            "_index": "test",
            "_type": "_doc",
            "_id": "2",
            "_version": 1,
            "result": "not_found",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 404,
            "_seq_no" : 1,
            "_primary_term" : 2
         }
      },
      {
         "create": {
            "_index": "test",
            "_type": "_doc",
            "_id": "3",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 201,
            "_seq_no" : 2,
            "_primary_term" : 3
         }
      },
      {
         "update": {
            "_index": "test",
            "_type": "_doc",
            "_id": "1",
            "_version": 2,
            "result": "updated",
            "_shards": {
                "total": 2,
                "successful": 1,
                "failed": 0
            },
            "status": 200,
            "_seq_no" : 3,
            "_primary_term" : 4
         }
      }
   ]
}

```
> action

* create -> 如果索引存在会失败
* index -> 增加或替换文档
* delete -> 下一行不需要source
* update -> 下一行是partial doc, upsert 或script

如果使用curl执行请求，建议使用--data-binary的方式从文件中读取请求体

```
$cat requests //请求体保存在requests文件中
{"index" : {"_index" : "test", "_id" : "1"}}
{"field1" : "value1"}

$curl -s -H "Content-Type=applicatio/x-ndjson" -XPOST localhost:9200/_bulk --data-binary "@requests"; 

```

> 其他参数

1.乐观并发控制（乐观锁 optimistic concurrency control） 

每个index/delete 动作, 可以在metadata line 添加 if_seq_no 和 if_primary_no 参数，请求时会比较该参数，匹配才执行

2. 版本控制(versioning)

每个bulk项都可以添加version参数，index/delete请求时将自动使用version映射

3. 路由(routing)

每个bulk项都可以添加routing参数，index/delete请求时将自动使用routing映射

4. 等待活跃分片(wait for active shards)

添加该参数后将等待指定数量的活跃副本分片

5. 刷新（refresh)

执行api后，立即刷新保证新的变更对其他操作可见。注意，这里不会刷新所有的分片，
仅刷新执行了bulk请求的分片，即它是分片级别的，并非索引级别的刷新（不会刷新索引下
的所有分片）

6. 更新 (update)

update 动作可以通过retry_on_conflict 指定一个版本冲突重试次数。
类似于update api，它支持doc和script两种更新方式，对于script，支持
params和upsert等选项

```
POST _bulk
{ "update" : {"_id" : "1", "_index" : "index1", "retry_on_conflict" : 3} }
{ "doc" : {"field" : "value"} }
{ "update" : { "_id" : "0", "_index" : "index1", "retry_on_conflict" : 3} }
{ "script" : { "source": "ctx._source.counter += params.param1", "lang" : "painless", "params" : {"param1" : 1}}, "upsert" : {"counter" : 1}}
{ "update" : {"_id" : "2", "_index" : "index1", "retry_on_conflict" : 3} }
{ "doc" : {"field" : "value"}, "doc_as_upsert" : true }
{ "update" : {"_id" : "3", "_index" : "index1", "_source" : true} }
{ "doc" : {"field" : "value"} }
{ "update" : {"_id" : "4", "_index" : "index1"} }
{ "doc" : {"field" : "value"}, "_source": true}

```

7. 部分响应

bulk API 的响应未必是完整的，当有的分片失败时，为了保证快速响应，它依旧返回结果，该
结果仅包含成功分片的数据，因此它未必是完整的。

#### 重新索引(REINDEX API)

reindex api 要求源索引开启_source

Reindex 不会尝试去设置新索引，即它不会自动的拷贝源索引的配置，需要在reindex前
手动的为新索引指定设置，映射，分片等。

最常见的操作是将文档集从一个索引复制到另一个索引

```

POST _reindex {
    "source" : {
	"index" : "twitter"
    },
    "dest" : {
	"index" : "new_twitter"
    }
}

``` 

> version_type

_reindex 请求会为source index 创建一个snapshot，但是它不会涉及version conflicts
的问题，因为该请求处理的是2个不同的索引。dest 元素可以设置version_type=internel
或忽略该参数，当reindex发现文档的type和id恰好相同时，将发生overwrite.

```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "internal"
  }
}

```

如果version_type设置为externel, 目标索引将会保存源索引的版本，reindex时：

对于target index:

1. 文档不存在，则创建文档
2. 文档的版本比较老时，将更新文档

如果想要避免target的文档被更新，可以通过设置op_type=create来保证只有文档
不存在时进行创建，否则将发生version conflict

```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}

```

默认情况下，发生version conflicts会导致reindex请求退出终止。如果想要避免
请求终止，可以设置conflicts:proceed属性，发生conflicts时仅记录，仍会继续
处理下一个文档，返回结果将包含conflict的文档数量。

```
POST _reindex
{
  "conflicts": "proceed",
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}

```

如果想按条件进行reindex,可以为source index 增加query参数进行过滤：

```
POST _reindex
{
  "source": {
    "index": "twitter",
    "query": {
      "term": {
        "user": "kimchy"
      }
    }
  },
  "dest": {
    "index": "new_twitter"
  }
}

```

如果想合并多个索引到一个新的索引，可以设置source为数组并添加它们

```
POST _reindex
{
  "source": {
    "index": ["twitter", "blog"]
  },
  "dest": {
    "index": "all_together"
  }
}

```

需要注意，如果索引存在id冲突，es会选择较后处理的作为最终文档，因此最后避免
这种不确定性行为。


我们可以通过size属性限制reindex文档的数量

```
POST _reindex
{
  "size": 1, //仅复制一条
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}

```

例如按时间排序获取前1000条：

```
POST _reindex
{
  "size": 10000,
  "source": {
    "index": "twitter",
    "sort": { "date": "desc" }
  },
  "dest": {
    "index": "new_twitter"
  }
}

```

我们可以通过指定_source仅复制部分字段

```
POST _reindex
{
  "source": {
    "index": "twitter",
    "_source": ["user", "_doc"]
  },
  "dest": {
    "index": "new_twitter"
  }
}

```

类似于_update_by_query，可以在复制的时候添加script，动态修改文档，
包括文档的内容或元信息。例如修改特定文档的版本并删除字段：

```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "external"
  },
  "script": {
    "source": "if (ctx._source.foo == 'bar') {ctx._version++; ctx._source.remove('foo')}",
    "lang": "painless"
  }
}

```

script可以动态调整op_type为noop或delete以决定是否保持或删除文档。

reindex请求可能会更改的四个元信息

* _id
* _index
* _version
    * 如果_version被设置为null，相当于version_type=internel, 当文档冲突时，会发生overwriter.
* _routing
    * keep -> 保持和source的routing一致
    * discard -> 丢弃，routing设置为null
    * =<some text> 设置为特定的routing

例如，将匹配的文档复制到dest，并设置它的routing为cat
```
POST _reindex
{
  "source": {
    "index": "source",
    "query": {
      "match": {
        "company": "cat"
      }
    }
  },
  "dest": {
    "index": "dest",
    "routing": "=cat"
  }
}

```

默认情况下，reindex每次复制1000个文档，可以通过source.size属性调整

```
POST _reindex
{
  "source": {
    "index": "source",
    "size": 100
  },
  "dest": {
    "index": "dest",
    "routing": "=cat"
  }
}

```

redindex也可以使用ingest node提供的pipeline对文档进行处理，通过dest.pipeline设置

```

POST _reindex
{
  "source": {
    "index": "source"
  },
  "dest": {
    "index": "dest",
    "pipeline": "some_ingest_pipeline"
  }
}

```

> 从远程ES集群reindex

指定source.remote，并在配置文件elasticsearch.yml中设置主机白名单

```
//设置白名单(逗号分隔，不需要加scheme（例如http://)
reindex.remote.whitelist: "otherhost:9200, another:9200, 127.0.10.*:9200, localhost:*"

//remote index
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200", //必须包含scheme,例如http://
      "username": "user", //开启base auth时需要
      "password": "pass" //开启base auth时需要
    },
    "index": "source",
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}

```

ps: remote reindex 不支持slice切片操作，无论是手动还是自动模式。

* heap buffer 的限制

remote reindex 使用堆内缓冲区缓存批次数据，最大限制为100M; 这意味着如果单个批次
文档较大，可能会打爆buffer，这种情况应该调小批次的数量，避免总量超限。

通过source.size属性设置, 注意，size和source.size作用不同，前者是限制reindex文档的数量。

```
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200"
    },
    "index": "source",
    "size": 10, //减小到10，默认是1k
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}

```

* 超时设置

remote reindex 涉及远程通信，对socket和connection可设置超时时间，默认
socket_read 1min， connection 10s。可通过source.remote.socket_timeout
和source.remote.connect_timeout 设置

```
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200",
      "socket_timeout": "1m",
      "connect_timeout": "10s"
    },
    "index": "source",
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}

```

> ssl 设置

可在elasticsearch.yml中对ssl配置，实现加密访问（https)。ssl支持配置许多
参数，例如reindex.ssl.certificate_authorities等（具体参考原文档）

> 通用 URL 参数支持

reindex 同样也支持写操作相关的一些通用参数：

* wait_for_completion -> task
    * task api
        * cancle -> 取消任务
        * rethrottling -> 重新调整限流策略
* refresh
* wait_for_active_shards -> timeout
* scroll
* requests_per_second

>  调整字段名

通过reindex可以在新的索引中替换旧索引的字段名，只需要通过script做一下替换：

```
POST _reindex
{
  "source": {
    "index": "test"
  },
  "dest": {
    "index": "test2"
  },
  "script": {
    //将flag字段替换为tag字段
    "source": "ctx._source.tag = ctx._source.remove(\"flag\")"
  }
}
```

> 切片

类似于update_by_query/delete_by_query, 对于批操作，可以使用切片机制并发的执行请求.

* 支持手动和自动切片
* 注意父子请求URL参数的映射关系
* 选择合适的切片数
    * 最佳实践：=分片数

> 批索引reindex

不建议通过pattern匹配多个索引一次请求进行reindex，因为灵活性比较差。建议自己控制
多个索引的reindex，例如实现一个脚本，循环串行或并行的处理它们，在部分索引失败后，
只需要对失败的部分重试即可。

例如：

循环串行执行：

```
for index in i1 i2 i3 i4 i5; do
  curl -HContent-Type:application/json -XPOST localhost:9200/_reindex?pretty -d'{
    "source": {
      "index": "'$index'"
    },
    "dest": {
      "index": "'$index'-reindexed"
    }
  }'
done

```

> reindex的其他用法

* 通过reindex让现有文档适配新的index template

新的template只能作用于新的索引，那么如何让旧的文档也采用新的template配置呢?

```
POST _reindex
{
  "source": {
    "index": "metricbeat-*"
  },
  "dest": {
    "index": "metricbeat"
  },
  "script": { //脚本创建新的索引名，例如metricbeat-2020.06.30 -> metricbeat-2020.06.30-1
    "lang": "painless",
    "source": "ctx._index = 'metricbeat-' + (ctx._index.substring('metricbeat-'.length(), ctx._index.length())) + '-1'"
  }
}

```

* 通过reindex 提取文档的子集（部分文档）

提取随机的文档子集

```
POST _reindex
{
  "size": 10, //限制仅提取10条
  "source": {
    "index": "twitter",
    "query": { //function_score_query -> 使用function调整文档分值
      "function_score" : {
        "query" : { "match_all": {} },
        "random_score" : {} //随机函数，生成随机分值（0-1）
      }
    },
    "sort": "_score" //根据_score进行排序， reindex默认是按照_doc进行排序
  },
  "dest": {
    "index": "random_twitter"
  }
}

```

#### 词条向量 (Term Vector)

通过term vector，可以查询文档或文档某个字段的词条统计信息，包括字段包含的词条，它们出现的位置，次数等信息。
要查询term vector, 必须在索引的mappings添加term_vector

```

//配置term_vector
PUT /twitter3
{
  "mappings" : {
      "properties" : {
        "user" : {
          "term_vector" : "with_positions_offsets_payloads",
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
  }
}

//按文档或字段查看term vector 词条统计

GET /twitter/_termvectors/1
GET /twitter/_termvectors/1?fields=message

```

词条向量支持查询词条信息，词条统计（默认关闭）和字段统计， 可以通过设置term_statistics=true开启词条统计（性能较差)

* 词条信息（term information)
    * 词条在字段中的频率
    * 词条位置(position)
    * 词条起始偏移量(start offset / end offsert)
    * 词条大小（payloads)
* 词条统计
    * 总词条次数(term_freq) -> 在所有文档中出现的次数
    * 文档次数(doc_freq) -> 保护该词条的文档数
* 字段统计
    * 文档数（doc_count)
    * 累加每个词条文档数的综合（sum_doc_freq）
    * 所有词条在文档的出现次数的总和（sum_ttf）
 
> 词条过滤

term支持一些可选参数，定义以上词条字段信息进行过滤，例如max_num_terms，
限制每个字段能返回的最大词条数，默认是25 


