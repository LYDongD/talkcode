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

#修改所有文档，添加新字段
POST /hamlet-raw/_update_by_query
{
  "script" : {
    "lang" : "painless",
    "source": "ctx._source.speaker2='hamlet'"
  }
}

#使用ingest pipeline 
#定义set processor
PUT /_ingest/pipeline/add-field
{
  "processors" : [{
    "set" : {
      "field" : "speaker3",
      "value" : "hamlet"
    }
  }]
}

POST /hamlet-raw/_update_by_query?pipeline=add-field

```

5 替换字段名

```
#定义rename processor
PUT _ingest/pipeline/rename_pipeline
{
  "processors": [
    {
      "rename": {
        "field": "line",
        "target_field": "text_entry"
      }
    }
  ]
}

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

#方法1：应用stored script
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


#方法2：应用pipeline
#定义一个脚本pipeline, 注意这里不使用_source
PUT _ingest/pipeline/add-is-ghost
{
  "processors": [
    {
      "script": {
        "lang" : "painless",
        "source": "if (ctx.speaker == 'Ghost') {ctx.is_ghost = true} else {ctx.is_ghost=false}"
      }
    }
  ]
}

#应用pipeline
POST /hamlet/_update_by_query?pipeline=add-is-ghost
{
  "query" : {
    "match_all" :{
    }
  }
}

```

7 删除指定文档

注意删除时使用term查询要保证字段类似是：keyword

```
#方法1： update_by_query + op
POST /hamlet/_update_by_query
{
  "script":{
    "lang" : "painless",
    "source" : "ctx.op = 'delete'"
  },
  "query" : {
    "terms" : {
      "speaker.keyword" : ["KING CLAUDIUS", "LAERTES"]
    }
  }
}

#方法2： delete_by_query

POST /hamlet/_delete_by_query
{
  "query" : {
    "terms" : {
      "speaker.keyword" : ["HORATIO", "Ghost"]
    }
  }
}

#方法3：用bool-should 复合查询代替terms
POST /hamlet/_delete_by_query
{
  "query" : {
    "bool" : {
      "should" : [{
        "term" : {
          "speaker.keyword" : "BERNARDO"
        }
      }, {
        "term" : {
          "speaker.keyword" : "FRANCISCO"
        }
      }]
    }
  }
}

```

ps: term查询和match查询的区别？

* term -> 类似于sql查询，支持精确，模糊，范围等查询方式
    * term 查询不会进行分词，因此不适合text类型的字段
    * term 适合keyword类型的字段
    * terms 支持多词项匹配
* match -> 全文搜索
    * match 查询文本会进行分词，并进行分词匹配
    * match 适合text类型的字段


#### 索引模板操作 （index template)

1 创建模板并应用到指定模式

```

#创建hamlet-template， 匹配hamlet-和hamlet_前缀的索引
PUT /_template/hamlet-template
{
  "index_patterns" : ["hamlet-*", "hamlet_*"],
  "settings" : {
    "number_of_shards" : 1,
    "number_of_replicas" : 0
  }
}

#删除多个索引，逗号隔开
DELETE hamlet-test, hamlet2

#更新模板，增加mappings，模板变更不会影响当前索引，需要删除重建
PUT /_template/hamlet-template
{
  "index_patterns" : ["hamlet-*", "hamlet_*"],
  "settings" : {
    "number_of_shards" : 1,
    "number_of_replicas" : 0
  },
  "mappings": {
    "properties": {
      "speaker" : {
        "type" : "keyword"
      },
      "line_number" : {
        "type" : "keyword"
      },
      "text_entry" : {
        "type" : "text",
        "analyzer" : "english"
      }
    }
  }
}



```

2 在模板中限制dynamic mappings

```
#dynamic设置成strict，如果索引未在template定义的字段将返回错误
PUT /_template/hamlet-template
{
  "index_patterns" : ["hamlet-*", "hamlet_*"],
  "settings" : {
    "number_of_shards" : 1,
    "number_of_replicas" : 0
  },
  "mappings": {
    "dynamic" : "strict",
    "properties": {
      "speaker" : {
        "type" : "keyword"
      },
      "line_number" : {
        "type" : "keyword"
      },
      "text_entry" : {
        "type" : "text",
        "analyzer" : "english"
      }
    }
  }
}

```

3 添加dynamic mapping template 影响自动映射的类型推断

```

#1 numner_开头的字段都映射为intger 2 string类型的字段都映射为keyword
PUT /_template/hamlet-template
{
  "index_patterns" : ["hamlet-*", "hamlet_*"],
  "settings" : {
    "number_of_shards" : 1,
    "number_of_replicas" : 0
  },
  "mappings": {
    "dynamic_templates" : [
      {
        "integers" : {
          "match" : "number_*",
          "mapping" : {
            "type" : "integer"
          }
        }
      },{
        "unanalyzed_text" : {
          "match_mapping_type" : "string",
          "mapping" : {
            "type" : "keyword"
          }
        }
      }
    ]
  }
}

```

#### 索引别名 (index alias)

1 定义索引别名，指向两个索引，该别名将合并两个索引的文档

```
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "hamlet-1",
        "alias": "hamlet"
      }
    },
    {
      "add" : {
        "index" : "hamlet-2",
        "alias": "hamlet"
      }
    }
  ]
}

```

2 指定其中一个索引为写入索引，通过别名索引文档时将路由到写入索引

```

#设置hamlet-1为写入索引
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "hamlet-1",
        "alias": "hamlet",
        "is_write_index" : true
      }
    },
    {
      "add" : {
        "index" : "hamlet-2",
        "alias": "hamlet"
      }
    }
  ]
}

#通过别名索引，将路由到hamlet-1写入
POST /hamlet/_doc
{
   "line_number" : "1.1.3",
   "speaker" : "LIAM",
   "text_entry" : "test write index"
}

```
#### reindex + alias 重新索引

通过reindex调整文档结构或修改属性值

```
#定义script并在reindex时使用script
POST /_scripts/control_reindex_batch
{
  "script" : {
    "lang": "painless",
    "source": "if (ctx._source.reindexBatch != null) {ctx._source.reindexBatch += params.increment} else {ctx._source.reindexBatch = 1}"
  }
}

#创建新索引
PUT hamlet-new
{
  "settings": {
    "number_of_shards": 2
    , "number_of_replicas": 0
  }
}

#将别名索引reindex到新的索引, 并启用2个分片线程
POST /_reindex?slices=2
{
  "source": {
    "index" : "hamlet"
  },
  "dest": {
    "index": "hamlet-new"
  },
  "script": {
    "id" : "control_reindex_batch",
    "params": {
      "increment" : 1
    }
  }
}

#调整hamlet别名，指向新的索引，删除旧的索引
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "hamlet-new",
        "alias": "hamlet",
        "is_write_index" : false
      }
    },
    {
      "remove" : {
        "index" : "hamlet-2",
        "alias" : "hamlet"
      }
    },
    {
      "remove" : {
        "index" : "hamlet-1",
          "alias" : "hamlet"
      }
    }
  ]
}

```

#### 通过pipeline处理器对文档进行处理

```
#模拟pipline -> 切割，生成新字段，删除临时字段
POST /_ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
    {
      "split": {
        "field": "line_number",
        "target_field" : "line_number_arr",
        "separator": "\\."
      }
    },
    {
      "set": {
        "field": "number_act",
        "value": "{{line_number_arr.0}}"
      }
    },
    {
      "set": {
        "field": "number_scene",
        "value": "{{line_number_arr.1}}"
      }
    },
    {
      "set": {
        "field": "number_line",
        "value": "{{line_number_arr.2}}"
      }
    },
    {
      "remove": {
        "field": "line_number_arr"
      }
    }
  ]
  },
  "docs": [{
    "_source" : {
      "line_number" : "1.2.3"
    }
  }]
}

#模拟无误后，创建pipeline
PUT /_ingest/pipeline/split_act_scene_line
{
  "processors": [
    {
      "split": {
        "field": "line_number",
        "target_field" : "line_number_arr",
        "separator": "\\."
      }
    },
    {
      "set": {
        "field": "number_act",
        "value": "{{line_number_arr.0}}"
      }
    },
    {
      "set": {
        "field": "number_scene",
        "value": "{{line_number_arr.1}}"
      }
    },
    {
      "set": {
        "field": "number_line",
        "value": "{{line_number_arr.2}}"
      }
    },
    {
      "remove": {
        "field": "line_number_arr"
      }
    }
  ]
}

#在指定索引上应用pipline更新
POST /hamlet-new/_update_by_query?pipeline=split_act_scene_line

```

#### 文档映射 （mappings)

1 设置字段类型和参数

```
#设置line_number不支持聚合查询（doc_values : false)
PUT /hamlet_1
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "speaker" : {
        "type": "text"
      },
      "line_number" : {
        "type": "keyword",
        "doc_values" : false
      },
      "text_entry" : {
        "type": "text"
      }
    }
  }
}

```
2 通过reindex的方式修改索引mappings

* 先创建新的索引，采用新的mappings, 例如为字段添加multi_fields
* 使用reindex api copy 文档

```
#先创建新索引，增加multi_fields, 支持全文搜索
PUT hamlet_2
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "speaker" : {
        "type": "keyword",
        "fields": {
          "tokens" : {
            "type" : "text"
          }
        }
      },
      "line_number" : {
        "type": "keyword",
        "doc_values" : false
      },
      "text_entry" : {
        "type": "text"
      }
    }
  }
}

#reindex
POST /_reindex
{
  "source": {
    "index": "hamlet_1"
  },
  "dest": {
    "index": "hamlet_2"
  }
}

```

3 对象类型查询问题 (object vs nested)

对象类型字段如果是一个对象数组，底层索引时会扁平化，即合并嵌套子对象

例如：

A ：[{B:1, C:2}, {B:3,C:4}] -> A.B : [1,3] , A.C : [2.4]

这样当我们搜索 "A.B=1，A.C=4" 的文档时，依然会返回两个结果；实际上并没有一个子对象同时满足该条件。

**解决方案：声明nested类型 + nested query**

```
#声明relationship为nested类型，子对象包含name和type两个属性
PUT hamlet_2
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name" : {
        "type": "text"
      },
      "relationship" : {
        "type": "nested", 
        "properties": {
          "name" : {
            "type" : "keyword"
          },
          "type" : {
            "type" : "keyword"
          }
        }
      }
    }
  }
}

#用nested query 进行查询

#nested query 要求path下的条件必须在一个nested对象内满足
GET /hamlet_2/_search
{
  "query": {
    "nested": {
      "path": "relationship",
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "relationship.name": "gertrude"
              }
            },
            {
              "match": {
                "relationship.type": "friend"
              }
            }
          ]
        }
      }
    }
  }
}

```

#### 父子文档（parent-child) 

1 为当前索引添加父子文档关联字段 （join类型）

```

#定义新的索引，增加父子文档关联字段：character_or_line, 父子关系为character -> line 为一对多关系 
PUT hamlet_3
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "character_or_line" : {
        "type": "join",
        "relations" : {
          "character" : "line"
        }
      }
    }
  }
}

```
2 给name=hamlet(_id=C0) 的 文档添加关联字段，作为父文档；给speaker=hamlet的文档添加关联字段，作为子文档

```

#更新父文档
POST /hamlet_3/_update_by_query
{
  "script" : "ctx._source.character_or_line='character'",
  "query" : {
    "term": {
      "_id": {
        "value": "C0"
      }
    }
  }
}

#用script + pipline 更新子文档
#定义脚本，为特定子文档添加关联字段，设置parent
PUT /_scripts/init_lines
{
  "script" : {
    "lang" : "painless",
    //用临时变量hashmap避免NPL异常
    "source" : """
      HashMap map = new HashMap();
      map.name = 'line';
      map.parent = params.characterId;
      ctx._source.character_or_line = map
    """
  }
}

#为子文档添加父文档信息（parent)时需要保证在同一个分片，即需要指定父文档的routing

#通过pipeline修改_routing
PUT _ingest/pipeline/add-routing
{
  "processors" : [
    {
      "script" : {
        "lang" : "painless",
        "source": "ctx._routing = 'C0'"
      }
    }  
  ]
}

POST /hamlet_3/_update_by_query?pipeline=add-routing
{
  "script" : {
    "id" : "init_lines",
    "params" : {
      "characterId" : "C0"
    }
  },
  "query" : {
    "term" : {
      "speaker.keyword" : "HAMLET"
    }
  }
}

```

ps: 为什么不能直接通过script修改_routing元信息

script是routing到指定节点后对文档进行更新，此时无法修改；
pipline通过ingest node 取出数据后可重新routing，因此它可以修改。


#### 自定义分词器 （analyzer)

1 自定义分词器
2 测试分词器
3 mapping中对指定字段设置分词器
4 搜索，测试

```
#分词器
#1 Hamlet词项转换为[CENSORED] 2 按空格分词 3 丢弃长短 < 5 的token
PUT hamlet_2
{
  "settings": {
    "analysis": {
      "analyzer": {
        "shy_hamlet_analyzer" : {
          "type" : "custom",
          "char_filter" : ["replace-hamlet"],
          "tokenizer" : "whitespace",
          "filter" : ["length-filter"]
        }
      },
      "char_filter" : {
          "replace-hamlet" : {
            "type" : "mapping",
            "mappings" : [
              "Hamlet  => [CENSORED]"
            ]
          }
        },
      "filter" : {
        "length-filter" : {
           "type" : "length",
           "min" : 5
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "text_entry" : {
        "type": "text",
        "analyzer": "shy_hamlet_analyzer"
      }
    }
  }
}

#测试
POST /hamlet_2/_analyze
{
  "analyzer" : "shy_hamlet_analyzer",
  "text" : "Though yet of Hamlet our dear brothers death" 
}

#reindex，对文档特定字段按新的分词器进行分词
POST _reindex
{
  "source": {
    "index": "hamlet_1"
  },
  "dest": {
    "index": "hamlet_2"
  }
}


#搜索
GET hamlet_2/_search
{
  "query": {
    "match": {
      "text_entry": "[CENSORED]"
    }
  }
}

```

#### 搜索 (_search)

1 match 全文搜索

```
#分页，注意默认的standard分词器会将token转换成小写建立倒排索引
GET kibana_sample_data_logs/_search
{
  "from": 50, 
  "size": 50, 
  "query": {
    "match": {
      "message": "Firefox"
    }
  }
}

#单字段逻辑“或”匹配，用minumum_should_match保证至少有k个词项匹配
GET kibana_sample_data_logs/_search 
{
  "query": {
    "match": {
      "message": {
        "query": "Firefox Kibana 159.64.35.129",
        "operator": "or",
        "minimum_should_match": 2
      }
    }
  },
  "profile": "true"
}

#也可以使用复合查询
GET kibana_sample_data_logs/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "message": "Firefox"
          }
        },
        {
          "match": {
            "message": "Kibana"
          }
        }
      ]
    }
  }
}

#单字段逻辑与匹配
GET kibana_sample_data_logs/_search 
{
  "profile": "true", 
  "query": {
    "match": {
      "message": {
        "query": "Firefox Kibana 159.64.35.129",
        "operator": "and"
      }
    }
  }
}

#高亮，并自定义高亮修饰符
GET kibana_sample_data_logs/_search
{
  "query": {
    "match": {
      "message": {
        "query": "Firefox Kibana",
        "operator": "or"
      }
    }
  },
  "highlight": {
    "fields": {
      "message": {
        "pre_tags": [
          "{{"
        ],
        "post_tags": [
          "}}"
        ]
      }
    }
  }
}

#使用search_after进行分页查询
# 1 根据时间和id排序，先查询10条；2 下一页从当前最后一条文档的时间和id往后查询
GET kibana_sample_data_logs/_search
{
  "size": 10,
  "query": {
    "match": {
      "message": {
        "query": "Firefox Kibana",
        "operator": "or"
      }
    }
  },
  "sort": [
    {
      "utc_time": {
        "order": "asc"
      }
    },
    {
      "_id": {
        "order": "asc"
      }
    }
  ]
}

#最后一条：
"sort" : [
          1593929408359,
          "sfeaW3MBJrOrL1jRF7q1"
        ]

#查询下一页
GET kibana_sample_data_logs/_search
{
  "size" : 10,
  "query": {
    "match": {
      "message": {
        "query": "Firefox Kibana",
        "operator": "or"
      }
    }
  },
  "sort": [
    {
      "utc_time": {
        "order": "asc"
      },
      "_id" : {
        "order": "asc"
      }
    }
  ],
  "search_after" : [1593922692345,"sfeaW3MBJrOrL1jRF7q1"]
}

```

2 match_pharse 短语搜索

短语要求完全匹配，包括词项的相对位置

```

#
GET kibana_sample_data_logs/_search
{
  "query": {
    "match_phrase": {
      "message": "HTTP/1.1 200 51"
    }
  },
  "sort": [
    {
      "machine.os.keyword": {
        "order": "desc"
      }
    },
    {
      "timestamp": {
        "order": "asc"
      }
    }
  ]
}
``` 

ps: 排序字段不能为text 

#### 结构化查询 （term)

结构化查询针对非文本(text)类型的字段，例如number, date, ip address, keyword等

1 使用过滤器 (bool + filter)

过滤器不影响得分，而且可以缓存，优先推荐该方式进行结构化查询

```
#range/term查询
GET kibana_sample_data_logs/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "range" : {
            "response" :{"gte" : 400, "lt" : 500}
          }
        },
        {
          "term" : {
            "referer" : "http://twitter.com/success/guion-bluford"
          }
        }
      ]
    }
  }
}

#prefix查询
GET kibana_sample_data_logs/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "prefix" : {
            "referer" : "http://twitter.com/success"
          }
        },
         {
          "prefix" : {
            "request.keyword" : "/people"
          }
        }
      ]
    }
  }
}

#空与非空查询（exists)
GET kibana_sample_data_logs/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "exists" : {
            "field" : "memory"
          }
        }
      ]
    }
  }
}

GET kibana_sample_data_logs/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "exists": {
            "field": "memory"
          }
        }
      ]
    }
  }
}


#时间范围过滤
GET kibana_sample_data_logs/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "range" : {
            "timestamp" : {
              "gte" : "now-7d/d",
              "lte" : "now/d"
            }
          }
        }  
      ]
    }
  }
}

```

2 复合查询（must + must_not + filter + should)

```

#与操作+过滤
GET kibana_sample_data_logs/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "agent": "Windows"
          }
        },
        {
          "match_phrase": {
            "url": "name:john"
          }
        }
      ],
      "filter": [
        {
          "exists" : {
            "field" : "phpmemory"
          }
        }  
      ]
    }
  }
}

#或操作
#GET kibana_sample_data_logs/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "range": {
            "response.keyword": {
              "gte": 400
            }
          }
        },
        {
          "term": {
            "tags.keyword": "error"
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}

#非操作
GET kibana_sample_data_logs/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "terms": {
            "tags.keyword": [
              "warning",
              "error",
              "info"
            ]
          }
        }
      ]
    }
  }
}
```

3 模糊查询

```

#fuziness=2指定最大编辑距离为2，因此仅有2个字符不同的文档可以被查询出来
GET kibana_sample_data_flights/_search
{
  "from": 40,
  "size": 10, 
  "query": {
    "bool": {
      "should": [
        {
          "fuzzy": {
            "OriginCityName": {
              "value": "Sidnei",
              "fuzziness" : 2
            }
          }
        },
        {
          "fuzzy": {
            "DestCityName": {
              "value": "Sidnei",
              "fuzziness" : 2
            }
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}


```

* fuzziness 指定最大编辑距离，可指定为0，1，2 或auto
    * auto, 默认为auto:3,6，根据查询文本长度动态调整最大编辑距离
        * 长度为<3 -> 最大编辑距离0
        * 长度3-5 -> 最大编辑距离1
        * 长度 >5 -> 最大编辑距离2
        * 例如sidnei, 长度为6，则最大编辑距离为2，相当于fuziness=2
* expansion
    * 指定分片匹配的最大文档数量，多分片时返回结果不准确




