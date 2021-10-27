---
title: es基础知识
date: 2021-09-17 00:00:00
author: lh
summary: ""
categories: es
tags:  
    - es
---

# 什么是 elasticsearch？
它是一个分布式、可扩展、实时的搜索与数据分析引擎。  
它能从项目一开始就赋予你的数据以搜索、分析和探索的能力，这是通常没有预料到的。  
以下 简称 es

## es在Windows环境下的安装
1、安装JDK，至少1.8.0_73以上版本，java -version  
2、下载和解压缩Elasticsearch安装包，目录结构,<a href="https://pan.baidu.com/s/1HsHIWr2NMdQsg5jC1F9LwQ"><font color=red>安装包链接</font></a>,提取码：bepq
3、启动Elasticsearch：bin\elasticsearch.bat，es本身特点之一就是开箱即用，如果是中小型应用，数据量少，操作不是很复杂，直接启动就可以用了出现started就是启动成功了。
![es-windows启动](https://kubpang.gitee.io/sourceFile/elasticsearch/es-windows启动.jpg) 
4、检查ES是否启动成功：http://localhost:9200/?pretty
```
name: node名称
cluster_name: 集群名称（默认的集群名称就是elasticsearch）
version.number: 5.2.0，es版本号
{
  "name" : "4onsTYV",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "nKZ9VK_vQdSQ1J0Dx9gx1Q",
  "version" : {
    "number" : "5.2.0",
    "build_hash" : "24e05b9",
    "build_date" : "2017-01-24T19:52:35.800Z",
    "build_snapshot" : false,
    "lucene_version" : "6.4.0"
  },
  "tagline" : "You Know, for Search"
}
```
5、修改集群名称：elasticsearch.yml
6、下载和解压缩Kibana安装包，使用里面的开发界面，去操作elasticsearch，作为我们学习es知识点的一个主要的界面入口
7、启动Kibana：bin\kibana.bat
8、访问http://localhost:5601 进入Dev Tools界面
9、在Dev Tools界面上运行 GET _cluster/health

## 端口号9300与9200区别
    区别：  
    9300端口：ES节点之间通讯使用  
    9200端口：ES节点和外部通讯使用  

    9300是TCP协议端口号，ES集群之间通讯的端口号  
    9200端口号，暴露ES Restful接口端口号

# lucene 与 es 的关系
lucene 是最先进、功能最强大的搜索库。  
如果直接基于 lucene 开发，非常复杂，即便写一些简单的功能，也要写大量的 Java 代码，需要深入理解原理。  

es 基于lucene， 隐蔽了 lucene的复杂性，提供了restful api 和 java api接口（还有其他语言的api接口）  
特点：  
* 分布式的文档存储引擎
* 分布式的搜索引擎和分析引擎
* 分布式 支持PB级数据

# es 的核心概念 
    near realtime(准实时)、cluster集群、node节点、  
    document&field、index、shard、replica

## Near realtime 准实时
* 从写入数据到数据可以被搜索有一个时间延迟（大概1s）
* 基于es执行搜索和分析可以达到秒级 
  
## Node 节点 
Node 是集群中的一个节点，节点也有一个名称，默认是随机分配的。  
默认节点会去加入一个名称为 elasticsearch 的集群。  
如果直接启动一堆节点，那么它们会自动组成一个 elasticsearch 集群，当然一个节点也可以组成 elasticsearch 集群。

## cluster 集群
集群包含多个节点，每个节点属于哪个集群可以<font color=red>通过 elasticsearch.yml </font>配置文件来决定  
中小型应用开始一个集群一个节点也正常

## Document & field
文档是 es 中最小的数据单元，一个 document 可以是一条客户数据、一条商品分类数据、一条订单数据。  
通常用 json 数据结构来表示。每个 index 下的 type，都可以存储多条 document。  
一个 document 里面<font color=red>有多个 field</font>，每个 field 就是一个数据字段。
```json
{
    "product_id": "1",
    "product_name": "iPhone X",
    "product_desc": "苹果手机",
    "category_id": "2",
    "category_name": "电子产品"
}
```  
### document -文档 
es集群中document 有点类似于 DB中的表，而document中的field则对应DB中的表字段 所以es依据这个结构特性，适应在NOSQL中使用

### document查询内部原理
* client 请求到任意一个node 使其成为coordinate node
* coordinate node 对document进行路由，将请求转发对应的node，此时会使用round-robin随机轮询算法，在所有的primary shard 和replica shard中随机取一个，让读请求负载均衡
* 接收请求shard对应的node 会将响应结果返回coordinate node
* coordinate node 再返回的client

特殊场景可能无法读取到document：document还在建立索引过程中，可能只在primary shard上有，replica没有，此时round-robin刚好随机指定了replica，从而导致document无法正常读取到，但是在其索引建立完成后，primary 和 replica可以正常读取

![document读请求内部原理](https://kubpang.gitee.io/sourceFile/elasticsearch/document读请求内部原理.png) 

## index - 索引
索引包含了一堆有相似结构的文档数据，比如商品索引。   
一个索引包含<font color=red>很多 document</font>，一个索引就代表了一类相似或者相同的 document,操作时<font color=red>index能是小写，可以包含下划线</font>。

## type - 类型 
每个索引里可以有一个或者多个 type，type 是 index 的一个逻辑分类，比如商品 index 下有多个 type：日化商品 type、电器商品 type、生鲜商品 type。    
每个 type 下的 document 的 field 可能不太一样。 
<font color=red>注意：</font> 6.x 只有一个type 7.x后 type取消


## mapping 
index的type的元数据，每个type都有一个自己的mapping，决定了数据类型，建立倒排索引的行为，还有进行搜索的行为，简称为mapping  
dynamic mapping：自动建立index，创建type，以及type对应的mapping，mapping中包含了每个field对应的数据类型，以及分词等设置  
es 在自动建立mapping的时候，对不同的field设置了data type，而不同data type的 分词、搜索等行为是不一致的，所以会导致 在在_search时，_all_field 和 指定字段的查询方式返回的结果可能不一致  
查看mapping
```shell script
#GET /index/_mapping/?pretty
GET /student/_mapping/?pretty
```

### 精准匹配与全文搜索的对比
* exact value: 精准匹配
    只有搜索内容与查询内容一致时才可以被查询出来
* full text： 全文检索 不是说单纯的只是匹配完整的一个值，而是可以对值进行拆分词语后（分词）进行匹配，也可以通过缩写、时态、大小写、同义词等进行匹配
    * 缩写 vs. 全程：cn vs. china
    * 格式转化：like liked likes
    * 大小写：Tom vs tom
    * 同义词：like vs love

### mapping的核心数据类型
字符串类型：string
整形：byte，short，integer，long
浮点类型：float，double
布尔类型：boolean
日期类型：date

### dynamic mapping
true or false	-->	boolean
123		-->	long
123.45		-->	double
2017-01-01	-->	date
"hello world"	-->	string/text

### 如何建立索引
analyzed ：建立分词
not_analyzed: 不建立分词
no：不被索引和搜索

### 修改mapping
只能建立index是手动建立mapping，或者新增field mapping，但是不能 update field mapping
```shell script
#6.x 版本正常运行 7.x 版本执行错误
#新建field mapping
PUT /website
{
  "mappings": {
    "article": {
      "properties": {
        "author_id": {
          "type": "long"
        },
        "title": {
          "type": "text",
          "analyzer": "english"
        },
        "content": {
          "type": "text"
        },
        "post_date": {
          "type": "date"
        },
        "publisher_id": {
          "type": "text",
          "index": "not_analyzed"
        }
      }
    }
  }
}

#7.x 去掉了type 
PUT /website
{
  "mappings": {
      "properties": {
        "author_id": {
          "type": "long"
        },
        "title": {
          "type": "text",
          "analyzer": "english"
        },
        "content": {
          "type": "text"
        },
        "post_date": {
          "type": "date"
        },
        "publisher_id": {
          "type": "text",
          "index": false
        }
      }
  }
}

PUT /website/_mapping/article
{
  "properties" : {
    "new_field" : {
      "type" :    "string",
      "index":    "not_analyzed"
    }
  }
}

# 修改field mapping 异常
PUT /website
{
  "mappings": {
    "article": {
      "properties": {
        "author_id": {
          "type": "text"
        }
      }
    }
  }
}

{
  "error": {
    "root_cause": [
      {
        "type": "index_already_exists_exception",
        "reason": "index [website/co1dgJ-uTYGBEEOOL8GsQQ] already exists",
        "index_uuid": "co1dgJ-uTYGBEEOOL8GsQQ",
        "index": "website"
      }
    ],
    "type": "index_already_exists_exception",
    "reason": "index [website/co1dgJ-uTYGBEEOOL8GsQQ] already exists",
    "index_uuid": "co1dgJ-uTYGBEEOOL8GsQQ",
    "index": "website"
  },
  "status": 400
}
```

### mapping 总结
* 往es里面直接插入数据，es会自动建立索引，同时建立type以及对应的mapping
* mapping中就自动定义了每个field的数据类型
* 不同的数据类型（比如说text和date），可能有的是exact value，有的是full text
* exact value，在建立倒排索引的时候，分词的时候，是将整个值一起作为<font color=red>一个关键词</font>建立到倒排索引中的；full text，会经历各种各样的处理，分词，normaliztion（时态转换，同义词转换，大小写转换），才会建立到倒排索引中
* 同时呢，exact value和full text类型的field就决定了，在一个搜索过来的时候，对exact value field或者是full text field进行搜索的行为也是不一样的，会跟建立倒排索引的行为保持一致；比如说exact value搜索的时候，就是直接按照整个值进行匹配，full text query string，也会进行分词和normalization再去倒排索引中去搜索
* 可以用es的dynamic mapping，让其自动建立mapping，包括自动设置数据类型；也可以提前手动创建index和type的mapping，自己对各个field进行设置，包括数据类型，包括索引行为，包括分词器，等等

## shard & replica
* index包含多个shard

* 每个shard都是一个最小工作单元，承载部分数据，每个 shard 都是一个 <font color=red>lucene 实例</font>。，有完整的建立索引和处理请求的能力。

* 增减节点时，shard会自动在nodes中负载均衡（尽量保证每个节点都是一样的负载）

* primary shard和replica shard，每个document肯定只存在于某一个primary shard以及其对应的replica shard中，不可能存在于多个primary shard

* replica shard是primary shard的副本，负责容错，以及承担读请求负载

* primary shard的数量在创建索引的时候就固定了，replica shard的数量可以随时修改，primary shard的数量是不能的修改的。

* primary shard的默认数量是5，replica默认是1，默认有10个shard，5个primary shard，5个replica shard（每个primary shard都对应一个replica shard）

* primary shard不能和自己的replica shard放在同一个节点上（否则节点宕机，primary shard和副本都丢失，起不到容错的作用），但是可以和其他primary shard的replica shard放在同一个节点上

## replica - 副本 (replica shard 简称 replica)
当服务出现宕机时，shard可能会丢失，因此可以为每个shard创建多个replica副本。  
replica 可以在shard出现故障时提供备用服务，保障数据不会丢失，多个replica可以提升搜索的吞吐量和性能。    
primary shard（建立索引时一次设置，不能修改，默认 5 个），replica shard（随时修改数量，默认 1 个），默认每个索引 10 个 shard，5 个 primary shard，5个 replica shard，最小的高可用配置，是 2 台服务器。

![es 集群 结构图](https://kubpang.gitee.io/sourceFile/elasticsearch/elasticsearch-2.png) 

# es 核心数据 与 db的比较 
|es|db|
|:---:|:---:|
|index|库|
|type|表|
|document|一行数据|     




资源路径：https://shishan100.gitee.io/docs/#/./docs/high-concurrency/es-introduction

