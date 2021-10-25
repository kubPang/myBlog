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
## index - 索引
索引包含了一堆有相似结构的文档数据，比如商品索引。   
一个索引包含<font color=red>很多 document</font>，一个索引就代表了一类相似或者相同的 document。

## type - 类型 
每个索引里可以有一个或者多个 type，type 是 index 的一个逻辑分类，比如商品 index 下有多个 type：日化商品 type、电器商品 type、生鲜商品 type。    
每个 type 下的 document 的 field 可能不太一样。 
<font color=red>注意：</font> 6.x 只有一个type 7.x后 type取消

## mapping 
自动或手动的为index中的type建立的一种数据结构和相关配置 简称为mapping  
dynamic mapping：自动建立index，创建type，以及type对应的mapping，mapping中包含了每个field对应的数据类型，以及分词等设置  
es 在自动建立mapping的时候，对不同的field设置了data type，而不同data type的 分词、搜索等行为是不一致的，所以会导致 在在_search时，_all_field 和 指定字段的查询方式返回的结果可能不一致

## shard （primary shard 简称 shard）
单台机器无法存储大量数据，es 可以将一个索引中的数据切分为多个 shard，分布在多台服务器上存储。 
有了 shard 就可以横向扩展，存储更多数据，让搜索和分析等操作分布到多台服务器上去执行，提升吞吐量和性能。每个 shard 都是一个 <font color=red>lucene index</font>。

## replica - 副本 (replica shard 简称 replica)
当服务出现宕机时，shard可能会丢失，因此可以为每个shard创建多个replica副本。  
replica 可以在shard出现故障时提供备用服务，保障数据不会丢失，多个replica可以提升搜索的吞吐量和性能。    
primary shard（建立索引时一次设置，不能修改，默认 5 个），replica shard（随时修改数量，默认 1 个），默认每个索引 10 个 shard，5 个 primary shard，5个 replica shard，最小的高可用配置，是 2 台服务器。

![es 集群 结构图](https://kubpang.gitee.io/sourceFile/elasticsearch-2.png) 

# es 核心数据 与 db的比较 
|es|db|
|:---:|:---:|
|index|库|
|type|表|
|document|一行数据|     




资源路径：https://shishan100.gitee.io/docs/#/./docs/high-concurrency/es-introduction

