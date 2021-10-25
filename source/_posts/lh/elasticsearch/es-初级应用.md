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
<front color="red">注意:</front> 当PUT作为update操作时，需要将所有filed全部写上，否则全覆盖，一般不建议用于update

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
bulk request会加载到内存里，如果太大的话，性能反而会下降，需要反复尝试得出一个最佳的bulk size，一般从1000~5000条数据开始，尝试逐渐增加  
如果看大小的话，最好控制在5~15MB之间

