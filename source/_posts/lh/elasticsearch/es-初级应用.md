---
title: es初级应用
date: 2021-09-18 00:00:00
author: lh
summary: ""
categories: es
tags: 
    - es
---

# es 集群健康状态
## es 查询集群健康状态

  
```ssh
GET /_cat/health?v

epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1488006741 15:12:21  elasticsearch yellow          1         1      1   1    0    0        1             0                  -                 50.0%

epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1488007113 15:18:33  elasticsearch green           2         2      2   1    0    0        0             0                  -                100.0%

epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1488007216 15:20:16  elasticsearch yellow          1         1      1   1    0    0        1             0                  -                 50.0%
```

## 集群健康状态 green、yellow、red
* green：每个索引的primary shard和replica shard都是active状态的
* yellow：每个索引的primary shard都是active状态的，但是部分replica shard不是active状态，处于不可用的状态
* red：不是所有索引的primary shard都是active状态的，部分索引有数据丢失了

## 为什么会会处于yellow状态？
比如当前es只有一台服务启动即只有一个node，此时只启动了一个primary shard，而依据es的 容错机制 replica shard 不予 primary shard 同服务，即单机情况下 集群状态为yellow     
也有当集群环境下 一台机器出现故障 处理正常运行的 primary shard 的 replica shard 恰好在故障服务器上 这时也是会显示yellow

# es 基础操作
## 查询集群下的 索引
```ssh
GET /_cat/indices?v

health status index   uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   student rUm9n9wMRQCCrRDEhqneBg   1   1          1            0      3.1kb          3.1kb

```

## 新建索引
```ssh
PUT /student?pretty

```


## 删除索引
```ssh
DELETE /student?pretty

```

## 索引的 CURD
* 新建搜索内容 document  
PUT /index/type/id (这里的id 不添加时，系统会默认一个值)    
{   
    "filed":"value"     
} // 整个大括号 则为 一个document 
<font color="red">注意:</font> 当PUT作为update操作时，需要将所有filed全部写上，否则全覆盖，一般不建议用于update

```ssh
PUT /student/class/1
{
    "name":"tom"
    "age":15
}

//数据下面有用到
PUT /student/class/2
{
    "name":"ajo"
    "age":20
}

PUT /student/class/3
{
    "name":"jack"
    "age":25
}
```

* 修改搜索内容    
 通过调用api _update 对指定索引下的字段进行更新
POST /index/type/id/_update (这里的id必填)   
{   
    "doc":{     
        "filed":"value" //filed 表示要修改的字段    
    }     
} 

```ssh
PUT /student/class/1/_update
{
    "doc":{
        "name":"jack"
    }
}

```


* 删除指定搜索内容      
删除索引下的一条记录
```ssh
DELETE /student/class/1

```

## es 查询操作相关
* 查询索引下全部的内容（全文搜索）
```ssh
GET /student/class/_search
{
  "query":{
    "match_all":{}
  }
}

```

* 查询 包含指定关键字        
这里搜索通过倒排索引 去搜索 如下：name 为tom 则会通过上面已经新增的document 得到一个列表   

|id|doc|     
|:---:| :---:|     
|1|tom|
|2|ajo|

|doc|id|
|---|---|
|tom|1|
|o|1,2|

即返回结果为 document id 为1、2的数据

```ssh
GET /student/class/_search
{
  "query":{
    "match":{
      "name":"tom"  
    }
  }
}

```

* phrase search 短语查询    
跟全文检索相对应，相反，全文检索会将输入的搜索串拆解开来，去倒排索引里面去一一匹配，只要能匹配上任意一个拆解后的单词，就可以作为结果返回
phrase search，要求输入的搜索串，必须在指定的字段文本中，完全包含一模一样的，才可以算匹配，才能作为结果返回

```ssh
GET /student/class/_search
{
  "query":{
    "match_phrase":{
      "name":"tom"  
    }
  }
}

```

* 查询条件
```ssh
GET /student/class/_search
{
  "query":{
    "bool": {
      "filter": [
        {"range": {
          "ago": {
            "gte": 10,//大于等于
            "lte": 20//小于等于
          }
        }}
      ]
    }
  }
}
```

* highlight search（高亮搜索结果）
```ssh
GET /ecommerce/product/_search
{
    "query" : {
        "match" : {
            "producer" : "producer"
        }
    },
    "highlight": {
        "fields" : {
            "producer" : {}
        }
    }
}
```

## mget批量查询api  
mget批量查询可以将多个操作合并一个读操作去执行，可以减少网络请求次数，提升系统性能，当然如果请求数据过多，也会在一定程度上影响了影响速度  
mget是很重要的，一般来说，在进行查询的时候，如果一次性要查询多条数据的话，那么一定要用batch批量操作的api，尽可能减少网络开销次数，可能可以将性能提升数倍，甚至数十倍，非常非常之重要  

* 场景1 不同index下 的查询
```ssh
GET /_mget
{
   "docs" : [
      {
         "_index" : "test_index1",
         "_type" :  "test_type",
         "_id" :    1
      },
      {
         "_index" : "test_index2",
         "_type" :  "test_type",
         "_id" :    2
      }
   ]
}
```

* 场景2 同一index下 的查询
```ssh
GET /test_index/type/_mget
{
   "ids": [1, 2]
}
```

## 批量写操作 _bulk 
bulk api对json的语法有严格要求，每个json串不能换行，只能放一行，同时一个json串和一个json串之间，必须有一个换行，bulk 的语法格式如下
```ssh
{"action": {"metadata"}}
{"data"}
```

bulk 主要支持一下类型的操作：
* delete: 删除一个文档，只要1个json串就可以了
* create: PUT /index/type/id/_create，强制创建
* index：普通的put操作，可以是创建文档，也可以是全量替换文档
* update：执行的partial update操作

```ssh
POST /_bulk
{ "delete": { "_index": "test_index", "_type": "test_type", "_id": "3" }} 
{ "create": { "_index": "test_index", "_type": "test_type", "_id": "12" }}
{ "test_field":    "test12" }
{ "index":  { "_index": "test_index", "_type": "test_type", "_id": "2" }}
{ "test_field":    "replaced test2" }
{ "update": { "_index": "test_index", "_type": "test_type", "_id": "1", "_retry_on_conflict" : 3} }
{ "doc" : {"test_field2" : "bulk test1"} }
```

### bulk 操作出现异常
bulk操作中，任意一个操作失败，是不会影响其他的操作，但是会在返回的结果里面告诉异常日志  

### bulk 性能
bulk request会加载到内存里，如果太大的话，性能反而会下降，需要反复尝试得出一个最佳的bulk size，一般从1000-5000条数据开始，尝试逐渐增加  
如果看大小的话，最好控制在5~15MB之间

## 分页查询
分页主要有两个参数  
    size：每一页多少条数据 
    from：表示从第多少条开始分页 <font color=red>起始页为0</font> 为0 与不传from 单传size效果一致
    
请求方式主要有：
```shell script
GET _search?size=2
GET _search?size=2&from=0 # 返回结果与第一条执行结果一致
GET _search?size=2&from=2
```

### deep paging 问题
集群环境下，当查询的目标数据很多，比如超过几十万甚至几百万数据时，这个时候进行分页，这个时候:
* step1 client 分页请求 如 _search?size=1000&from=1000000  到 coordinate node ,节点会找到该查询多个对应的shard（假设3shard）
* step2 请求到达对应的shard 开始执行分页操作 _search?size=1000&from=1000000 取1000条数据 并返回给coordinate node
* step3 coordinate收到shard返回的总数据条数为3000条，这个时候开始做重排序，默认是<font color=red>通过相关度分数排序</font>,取前1000条数据返回给client 

deep paging问题 其实是一个深度查询的问题，如涉及分页查询较深时且数据较大时，非常消耗网络带宽，消耗内存，所以存在性能问题，应尽量避免 deep paging操作

# 多种搜索方式
## query string search
query string search的由来：
因为search参数都是以http请求的query string来附带的。
注意，这个查询在生产环境上使用的是不多的。

（1）搜索所有的商品：
```
GET /sell/product/_search
返回值：
{
  "took": 12,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 1,
    "hits": [
      {
        "_index": "sell",
        "_type": "product",
        "_id": "2",
        "_score": 1,
        "_source": {
          "name": "oyr yaogao",
          "desc": "zheshi oyr yagao",
          "price": 3000,
          "producer": "oyr yagao producer",
          "tags": [
            "lengcang",
            "baoxian"
          ]
        }
      },
      {
        "_index": "sell",
        "_type": "product",
        "_id": "1",
        "_score": 1,
        "_source": {
          "name": "gaolujie yagao",
          "desc": "zheshi gaolujie yigeyagao",
          "price": 30,
          "producer": "gaolujie yagao producer",
          "tags": [
            "meibai",
            "fangzhu"
          ]
        }
      },
      {
        "_index": "sell",
        "_type": "product",
        "_id": "3",
        "_score": 1,
        "_source": {
          "name": "latiao",
          "desc": "zheshi latiao yagao",
          "price": 15,
          "producer": "latiao yagao producer",
          "tags": [
            "la",
            "meiwei"
          ]
        }
      }
    ]
  }
}
```
返回值说明：
took：耗费了多少毫秒
timed_out：是否超时，这里是没有
_shards：数据拆成了5个分片，所以对于搜索请求，会打到所有的primary shard（或者是它的某个replica shard也是可以）
hits.total：查询结果的数量，3个document
hits.max_score:score的含义：就是document对应一个search的相关度的匹配分数，越相关，就越匹配，分数也越高。这里显示的是最大的一个匹配分数
hits.hits：包含了匹配搜索的document的详细数据

（2）搜索商品名称中包含yagao的商品，而且按照售价降序排序
```
GET /sell/product/_search?q=name:yagao&sort=price:desc
返回值：
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": null,
    "hits": [
      {
        "_index": "sell",
        "_type": "product",
        "_id": "2",
        "_score": null,
        "_source": {
          "name": "oyr yagao",
          "desc": "zheshi oyr yagao",
          "price": 3000,
          "producer": "oyr yagao producer",
          "tags": [
            "lengcang",
            "baoxian"
          ]
        },
        "sort": [
          3000
        ]
      },
      {
        "_index": "sell",
        "_type": "product",
        "_id": "1",
        "_score": null,
        "_source": {
          "name": "gaolujie yagao",
          "desc": "zheshi gaolujie yigeyagao",
          "price": 30,
          "producer": "gaolujie yagao producer",
          "tags": [
            "meibai",
            "fangzhu"
          ]
        },
        "sort": [
          30
        ]
      },
      {
        "_index": "sell",
        "_type": "product",
        "_id": "3",
        "_score": null,
        "_source": {
          "name": "latiao yagao",
          "desc": "zheshi latiao yagao",
          "price": 15,
          "producer": "latiao yagao producer",
          "tags": [
            "la",
            "meiwei"
          ]
        },
        "sort": [
          15
        ]
      }
    ]
  }
}
```


## query DSL
DSL：Domain Specified Language，特定领域的语言
参数是放在http request body中的。
http request body：请求体，可以用json的格式来构建查询语法，比较方便，可以构建各种复杂的语法，比query string search肯定强大多了

（1）查询所有的商品
```
GET /sell/product/_search
{
  "query": {
    "match_all": {}
  }
}
```

（2）查询名称包含yagao的商品，同时按照价格降序排序
```
GET /sell/product/_search
{
  "query": {
    "match": {
      "name": "yagao"
    }
  }
  , "sort": [
    {
      "price": "desc"
    }
  ]
}
或

GET /sell/product/_search
{
  "query": {
    "match": {
      "name": "yagao"
    }
  },
  "sort": [
    {
      "price": {
        "order": "desc"
      }
    }
  ]
}
```

（3）分页查询商品，总共3条商品，假设每页就显示1条商品，现在显示第2页，所以就查出来第2个商品
```
GET /sell/product/_search
{
  "query": {
    "match_all": {}
  },
  "from": 1, // 从第一条开始查,并不包含第一条
  "size": 1 // 查一条数据
}
```

（4）指定要查询出来商品的名称和价格就可以，也就是具体要显示哪些field。
```
GET /sell/product/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["name", "price"]
}
```

## query filter

（1）搜索商品名称包含yagao，而且售价大于25元的商品
```
GET /sell/product/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "name": "yagao"
        }
      },
      "filter": {
        "range": {
          "price": {
            "gt": 25
          }
        }
      }
    }
  }
}
```
## full-text search
全文检索
```
GET /sell/product/_search
{
  "query": {
    "match": {
      "producer": "oyr yagao producer"
    }
  }
}
```
producer这个字段，会先被分词拆解
yagao
producer
然后一个个去匹配文档中producer对应的倒排索引
```
返回值：
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0.7594807,
    "hits": [
      {
        "_index": "sell",
        "_type": "product",
        "_id": "2",
        "_score": 0.7594807,
        "_source": {
          "name": "oyr yagao",
          "desc": "zheshi oyr yagao",
          "price": 3000,
          "producer": "oyr yagao producer",
          "tags": [
            "lengcang",
            "baoxian"
          ]
        }
      },
      {
        "_index": "sell",
        "_type": "product",
        "_id": "1",
        "_score": 0.5063205,
        "_source": {
          "name": "gaolujie yagao",
          "desc": "zheshi gaolujie yigeyagao",
          "price": 30,
          "producer": "gaolujie yagao producer",
          "tags": [
            "meibai",
            "fangzhu"
          ]
        }
      },
      {
        "_index": "sell",
        "_type": "product",
        "_id": "3",
        "_score": 0.5063205,
        "_source": {
          "name": "latiao yagao",
          "desc": "zheshi latiao yagao",
          "price": 15,
          "producer": "latiao yagao producer",
          "tags": [
            "la",
            "meiwei"
          ]
        }
      }
    ]
  }
}
```


## phrase search
短语搜索
跟全文检索相对应，相反，全文检索会将输入的搜索串拆解开来，去倒排索引里面去一一匹配，只要能匹配上任意一个拆解后的单词，就可以作为结果返回
phrase search，要求输入的搜索串，必须在指定的字段文本中，完全包含一模一样的，才可以算匹配，才能作为结果返回
```
GET /sell/product/_search
{
  "query": {
    "match_phrase": {
      "producer": "oyr yagao producer"
    }
  }
}
返回值：
{
  "took": 15,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.7594808,
    "hits": [
      {
        "_index": "sell",
        "_type": "product",
        "_id": "2",
        "_score": 0.7594808,
        "_source": {
          "name": "oyr yagao",
          "desc": "zheshi oyr yagao",
          "price": 3000,
          "producer": "oyr yagao producer",
          "tags": [
            "lengcang",
            "baoxian"
          ]
        }
      }
    ]
  }
}
```


## 聚合查询

（1）计算每个tag下的商品数量
先将文本field的fielddata属性设置为true，不然执行聚合查询会报错。
```
PUT /sell/_mapping/product
{
  "properties": {
    "tags":{
      "type": "text",
      "fielddata": true
    }
  }
}

查询：
GET /sell/product/_search
{
  "size": 0, 
  "aggs": {
    "group_by_tags": {
      "terms": {
        "field": "tags"
      }
    }
  }
}
```
类似于数据库sql:
```sql
Select tags, count(*) from product group by tags
```
返回值：
```
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_tags": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "meibai",
          "doc_count": 2
        },
        {
          "key": "meiwei",
          "doc_count": 2
        },
        {
          "key": "fangzhu",
          "doc_count": 1
        }
      ]
    }
  }
}
```

（2）对名称中包含yagao的商品，计算每个tag下的商品数量
```
GET /sell/product/_search
{
  "size": 0, 
  "query": {
    "match": {
      "name": "yagao"
    }
  }, 
  "aggs": {
    "group_by_tags": {
      "terms": {
        "field": "tags"
      }
    }
  }
}
```
类似于数据库sql:
```sql
Select tags, count(*) from product where name='yagao' group by tags
```
返回值：
```
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_tags": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "meibai",
          "doc_count": 2
        },
        {
          "key": "meiwei",
          "doc_count": 2
        },
        {
          "key": "fangzhu",
          "doc_count": 1
        }
      ]
    }
  }
}
```

（3）先分组，再算每组的平均值，计算每个tag下的商品的平均价格
```
GET /sell/product/_search
{
  "size": 0,
  "aggs": {
    "group_by_tags": {
      "terms": {
        "field": "tags"
      },
      "aggs": {
        "avg_price": {
          "avg": {"field": "price"}
        }
      }
    }
  }
}
```
类似于数据库sql:
```sql
Select tags, count(*), avg(price) from product group by tags
```
返回值：
```
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_tags": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "meibai",
          "doc_count": 2,
          "avg_price": {
            "value": 1515
          }
        },
        {
          "key": "meiwei",
          "doc_count": 2,
          "avg_price": {
            "value": 1507.5
          }
        },
        {
          "key": "fangzhu",
          "doc_count": 1,
          "avg_price": {
            "value": 30
          }
        }
      ]
    }
  }
}
```

（4）计算每个tag下的商品的平均价格，并且按照平均价格降序排序
```
GET /sell/product/_search
{
  "size": 0,
  "aggs": {
    "group_by_tags": {
      "terms": {
        "field": "tags",
        "order": {
          "avg_price": "desc"
        }
      },
      "aggs": {
        "avg_price": {
          "avg": {"field": "price"}
        }
      }
    }
  }
}
```
类似于数据库sql:
```sql
Select tags, count(*), avg(price) as avg_price from product group by tags order by avg_price
```
返回值：
```
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_tags": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "meibai",
          "doc_count": 2,
          "avg_price": {
            "value": 1515
          }
        },
        {
          "key": "meiwei",
          "doc_count": 2,
          "avg_price": {
            "value": 1507.5
          }
        },
        {
          "key": "fangzhu",
          "doc_count": 1,
          "avg_price": {
            "value": 30
          }
        }
      ]
    }
  }
}
```

（5）按照指定的价格范围区间进行分组，然后在每组内再按照tag进行分组，最后再计算每组的平均价格
```
GET /sell/product/_search
{
  "size": 0,
  "aggs": {
    "group_by_price": {
      "range": {
        "field": "price",
        "ranges": [
          {
            "from": 0,
            "to": 20
          },
          {
            "from": 20,
            "to": 10000
          }
        ]
      },
      "aggs": {
        "group_by_tags": {
          "terms": {
            "field": "tags"
          },
          "aggs": {
            "avg_price": {
              "avg": {
                "field": "price"
              }
            }
          }
        }
      }
    }
  }
}
```
返回值：
```
{
  "took": 3,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_price": {
      "buckets": [
        {
          "key": "0.0-20.0",
          "from": 0,
          "to": 20,
          "doc_count": 1,
          "group_by_tags": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "meiwei",
                "doc_count": 1,
                "avg_price": {
                  "value": 15
                }
              }
            ]
          }
        },
        {
          "key": "20.0-10000.0",
          "from": 20,
          "to": 10000,
          "doc_count": 2,
          "group_by_tags": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "meibai",
                "doc_count": 2,
                "avg_price": {
                  "value": 1515
                }
              },
              {
                "key": "fangzhu",
                "doc_count": 1,
                "avg_price": {
                  "value": 30
                }
              },
              {
                "key": "meiwei",
                "doc_count": 1,
                "avg_price": {
                  "value": 3000
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

## filter 与 query的深入比较
### filter与query示例
```shell script
PUT /company/employee/2
{
  "address": {
    "country": "china",
    "province": "jiangsu",
    "city": "nanjing"
  },
  "name": "tom",
  "age": 30,
  "join_date": "2016-01-01"
}

PUT /company/employee/3
{
  "address": {
    "country": "china",
    "province": "shanxi",
    "city": "xian"
  },
  "name": "marry",
  "age": 35,
  "join_date": "2015-01-01"
}

#搜索请求：年龄必须大于等于30，同时join_date必须是2016-01-01
GET /company/employee/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "join_date": "2016-01-01"
          }
        }
      ],
      "filter": {
        "range": {
          "age": {
            "gte": 30
          }
        }
      }
    }
  }
}

```

### filter 与 query对比 
* filter: 仅仅只是按搜索条件过滤出所需要的数据而已，不涉及相关度分数的计算，对相关度没有影响
* query： 会去计算每个document相对搜索条件的相关度，并按相关度进行排序

总结： 
* 如果是在进行搜索，需要将最匹配搜索条件的数据先返回，那么用query。
* 如果只是想根据搜索条件筛选出一部分数据，那么用filter
* 如果需要将符合条件的document排名靠前，用query包含，如果想其他的条件不影响到前面的document排序则用filter过滤

### filter与query性能
* filter： 不需要计算相关度分数，不需要按照相关度分数进行排序，同时还有内置的自动cache最常使用filter的数据
* query： 相反，要计算相关度分数，按照分数进行排序，而且无法cache结果

### 单执行filter需注意
```shell script
# 参数 constant_score 必传 否则会出现异常
GET /student/class/_search 
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "mark": {
            "gte": 30
          }
        }
      }
    }
  }
}
```

## 几种query的搜索语法
### match all 
    查询所有
```shell script
#查询所有index的内容
GET /_search
{
    "query": {
        "match_all": {}
    }
}
```
```shell script
#查询指定index下的所有内容
GET student/class/_search
{
  "query":{
    "match_all": {}
  }
}
```
### match
    匹配某一个filed是否包含某个文本，会触发分词，相当于是 full text(全文检索)
```shell script
# 查询所有index 指定field是否包含查询内容
GET /_search
{
  "query":{
    "match": {
      "title": "学习"
    }
  }
}
```
```shell script
# 查询指定index 指定field是否包含查询内容
GET student/class/_search
{
  "query":{
    "match": {
      "title": "学习"
    }
  }
}
```

### multi match
    查询多个field是否包含查询内容
```shell script
# 查询所有index 指定的fields下是否包含查询内容
GET /_search
{
  "query":{
    "multi_match":{
      "query":"学习",
      "fields":["title","desc"]
    }
  }
}
```
```shell script
# 查询指定index 指定的fields下是否包含查询内容
GET student/class/_search
{
  "query":{
    "multi_match":{
      "query":"学习",
      "fields":["title","desc"]
    }
  }
}

```
    
### range query
    查询field是否在指定的范围值内，放query里会对相关度产生影响，放filter里面无影响
```shell script
# 查询所有index 指定的field是否在查询内容范围内
GET /_search
{
  "query":{
    "range":{
      "mark":{
        #"gt":80, 大于
        #"lt":120 小于
        "gte":80, #大于等于
        "lte":120 #小于等于

      }
    }
  }
}
```
```shell script
GET student/class/_search
{
  "query":{
    "range":{
      "mark":{"gte":80}
    }
  }
}
```

### term query
   把查询的数据当成 exact value（精准匹配）进行查询。  
   这里查询的内容是要精准匹配field内容  
   需要建立索引的时候，指定<font color=red>field不分词</font>才能查询到  
   或者field已经是<font color=red>最小分词单位</font>
```shell script
GET /_search
{
  "query":{
    "term": {
      "name": "八"
    }
  }
}
```
```shell script
GET student/class/_search
{
  "query":{
    "term": {
      "name": "八"
    }
  }
}
```

### terms query
    原理与term一致，不过对指定的field可以查询指定多个搜索词
```shell script
GET student/class/_search
{
  "query":{
    "terms": {
      "name": ["八","九"]
    }
  }
}
```

## 组合查询
参数 bool 多条件组合查询参数，bool中可以使用 must、 must_not 、should 来组合查询条件 ,bool 可嵌套  
一下参数在与match、multi match、term、terms一起使用时注意查询方式是exact value or full value 
* must: 需要满足条件 ==或like
* must_not: 不需要在满足条件内的 !=或 not like
* should: should中的两个条件至少满足一个就可以,should下有多个条件时注意加参数 minimum_should_match
* filter

```shell script
# 查询学习成绩在30~100的学生信息，成绩倒叙，备注含学习,李四作弊取消成绩
GET student/class/_search?sort=mark:desc
{
  "query":{
    "bool": {
     "must_not": [
       {"match": {
         "name": "李四"
       }}
     ],
     "minimum_should_match":2, # should下满足几个条件
     "should": [
      {"match":{"title":"委员"}}
      ,
      {"match":{"desc":"突出"}}
     ], 
      "filter": {
        "bool": {
          "must":
          [
            {"range":{"mark": {"gte":20,"lte":100}}},
            {"match":{"desc": "学习"}}
          ]
        }
        
      }
    }
  }
}
```

## 校验不合法的搜索
特别复杂庞大的搜索下，比如你一下子写了上百行的搜索，这个时候可以先用validate api去验证一下，搜索是否合法
```shell script
#格式: GET index/type/_validate/query?explain
GET student/class/_validate/query?explain
{
  "query":{
    "match_all": {}
  }
}
```
返回结果
```shell script
{
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "valid" : true,
  "explanations" : [
    {
      "index" : "student",
      "valid" : true,
      "explanation" : "+*:* #*:*"
    }
  ]
}
```