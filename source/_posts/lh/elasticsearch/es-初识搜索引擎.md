---
title: es-搜索引擎
date: 2021-10-25 00:00:00
author: lh
summary: ""
categories: es
tags:  
    - es
---

# 为什么要用搜索引擎
* 数据类型  
全文索引搜索支持非结构化数据的搜索，可以更好地快速搜索大量存在的任何单词或单词组的非结构化文本。
例如 Google，百度类的网站搜索，它们都是根据网页中的关键字生成索引，我们在搜索的时候输入关键字，它们会将该关键字即索引匹配到的所有网页返回；还有常见的项目中应用日志的搜索等等。对于这些非结构化的数据文本，关系型数据库搜索不是能很好的支持。

* 索引的维护  
一般传统数据库，全文检索都实现的很鸡肋，因为一般也没人用数据库存文本字段。进行全文检索需要扫描整个表，如果数据量大的话即使对SQL的语法优化，也收效甚微。建立了索引，但是维护起来也很麻烦，对于 insert 和 update 操作都会重新构建索引。

## 什么时候使用全文搜索引擎
* 搜索的数据对象是大量的非结构化的文本数据。
* 文件记录量达到数十万或数百万个甚至更多。
* 支持大量基于交互式文本的查询。
* 需求非常灵活的全文搜索查询。
* 对高度相关的搜索结果的有特殊需求，但是没有可用的关系数据库可以满足。
* 对不同记录类型、非文本数据操作或安全事务处理的需求相对较少的情况。

## 倒排索引
倒排索引是实现“单词-文档矩阵”的一种具体存储形式，通过倒排索引，可以根据单词快速获取包含这个单词的文档列表。倒排索引主要由两个部分组成：“单词词典”和“倒排文件”。例如   
document1 ： welcome to china
document2 ： china is beautiful  
document3 ： tom like china  

|单词字典|倒排文件|
|:---:|:---:|
|welcome|1|
|to|1|
|china|1,2,3|
|is|2|
|beautiful|2|  
|tom|3|  
|like|3|  

在建立倒排索引时，会<font color=red>使用标准化规则（normalization）</font>，会将document分词出来的单词字典 进行标准化处理，比如：  
        * 缩写 vs. 全程：cn vs. china
        * 格式转化：like liked likes
        * 大小写：Tom vs tom
        * 同义词：like vs love
        
## 什么是分词器
切分词语，normalization（提升recall召回率），将一段文本进行各种处理，最后处理好的结果才会拿去建立倒排索引  
recall-召回率：搜索时增加能够搜索到的结果数量 

分词其包含三部分：
* character filter-过滤特殊字符：分词前先进行预处理，最常见的就是过滤html标签等
* tokenizer-分词： hello you and me --> hello, you, and, me
* token filter-进行标准化处理：标准化处理，比如缩写、格式转化、大小写、同义词等



## 内置分词器的介绍
document 案例：Set the shape to semi-transparent by calling set_trans(5)
* standard analyzer： 大小写转化，去除一些特殊符号、大小写拆分(es默认的分词器)  
    set, the, shape, to, semi, transparent, by, calling, set_trans, 5
* simple analyzer：去除一些特殊符号,可以依据-,_来拆分字符  
    set, the, shape, to, semi, transparent, by, calling, set, trans
* whitespace analyzer： 自会根据空格进行拆分，不会处理大小写 特殊字符  
    Set, the, shape, to, semi-transparent, by, calling, set_trans(5)
* language analyzer: 特定的语言的分词器，比如说，english，英语分词器  
    set, shape, semi, transpar, call, set_tran, 5
    
## 测试分词器 
```shell script
GET /_analyze
{
  "analyzer": "standard", # 给定的分词器
  "text": "Text to analyze" # 待分词的文本
}
```
返回结果
```shell script
{
  "tokens" : [
    {
      "token" : "text",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "to",
      "start_offset" : 5,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "analyze",
      "start_offset" : 8,
      "end_offset" : 15,
      "type" : "<ALPHANUM>",
      "position" : 2
    }
  ]
}
```
