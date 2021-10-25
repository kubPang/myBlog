---
title: es分布式架构
date: 2021-10-22 00:00:00
author: lh
summary: ""
categories: es
tags:  
    - es
---
# es 分布式框架简述
## es 对分布式机制的透明隐藏特性
Elasticsearch是一套分布式的系统，分布式是为了应对大数据量隐藏了复杂的分布式机制
 * 分片机制 
    将数据写入document时，数据如何进行分片，写入那个shard中等等
 * cluster discovery（集群发现机制）：
    如扩展集群时，启动第二个es进程，那个进程作为一个node自动就发现了集群，并且加入了进去，还接受了部分数据，replica shard
 * shard的负载均衡 
    举例，假设现在有3个节点，总共有25个shard要分配到3个节点上去，es会自动进行均匀分配，以保持每个节点的均衡的读写负载请求  
    其实是通过coordinate node 来处理的

## es 集群的扩容方式
    如现有集群 3集群 每台服务器能容纳1T数据 3个node 3个primary shard replica shard 设置为2  
 * 水平扩容
    集群数量+1 当前服务器*4 增加一个node节点 now 4个node 通过master node 将 replica shard copy过去 （现有primary shard 已经最大）
    优点：提升集群的容错性能，多一个node 可以更好的应对单节点故障所产生的风险，提供并发处理能力
    ![es 集群 结构图](https://kubpang.gitee.io/sourceFile/es集群水平扩容.png) 
 * 垂直扩容
    集群数量不变，增加某台服务内存或者集群内所有的机器加内存，比如 由现有的1T 变为2T 
    优点：使得单台node的处理性能变的更佳
    缺点：垂直扩容，成本相对应的成倍增加，应对节点故障，对比扩容前无改变
  一般采用水平扩容。
    
### 水平扩容的极限 以及如何提高容错性
    * primary&replica自动负载均衡，6个shard 3个primary shard 3个replica shard
    * 每个node 有更少的shard，这是 IO/CUP/Mamory 资源给每个shard也就越多，每个shard的性能也就越好
    * 扩容极限 6个shard（3个primary 3个replica） 最多扩容到6台机器，一个shard独享node的 所有资源 性能最好
    * 超出扩容极限 动态修改replica数量，9个shard（3个primary 6个replica），9台机器比3台机器时 拥有了3倍的吞吐量
    * 容错 3台机器 6个shard（3个primary 3个replica）容忍1台机器宕机，3台机器 9个shard（3个primary 6个replica）容忍2台机器宕机，同一个primary与replica不能同时在一个node上出现

## es 集群的master节点 
    * master 节点不会承载所有的请求，所有不会有单点瓶颈
    * 管理es集群的元数据：
        比如document的创建、修改和删除，维护索引的元数据；节点的新增和删除，维护集群的元数据（水平扩容）
    * 默认情况下，集群会自动选举出一个master节点，如master节点出现故障后，集群会在剩余节点中选出一个master节点
    
## 节点平等的分布式架构
    * 节点对等，每个节点都可以接收到所有请求
    * 自动请求路由 client的请求发送到某个节点时，这个节点会自动去请求对应的index  
    * 相应收集 收到请求的节点，将请求转发出去后，相应结果会返回改节点 有改节点统一发给client 这个节点 就是 coordidata node
    
## 写一致性以及quorum机制
### 写一致性
    在发送任何es的 增删改操作时，都可以带一个consistency参数，来指明想要的写一致性是什么  
    consistency： one、all、quorum
    * one: 要求写操作时，只要<front color=red>有一个primary shard</front> 是active，就可以执行
    * all：要求写操作时，必须<front color=red>所有的primary shard 和replica shard</front> 都是active，才可以执行这个写操作
    * quorum: 要求所有的shard中，必须是<front color=red>大部分shard</front> 都是active，才可以执行这个写操作

### quorum机制
    写之前 需确保大多数的shard是可用的，且只有在 number_of_replica > 1是才生效  
    quroum = int((primary + number_of_replica)/2 ) + 1  

    * 场景1 3台服务器 3个primary replicas 为1 6个shard： 3primary replica：1*3 
        quroum = int ((3+ 1)/2) + 1 = 3  
        也就是说明 3台服务器 6个shard 时 有两个node 也就是4个shard是active时 可以进行写操作
    * 场景2 3台服务器 3个primary 3个replica  
         quroum = int ((3+ 3)/2) + 1 = 4 当shard 中active数>4时可进行写操作  
         注：如果此时宕机2台机器 那么shard的active数 只有3 这时不能进行写操作 但是可以进行读操作
         
    * 场景3 单机场景下 一个primary shard replica 默认为1 0个active  
        因为 number_of_replica = 1 不触发quroum 一致性 也就是说可以支持读写操作
    
    * time_out 当quorum 不齐全时，es会等待wait  
        默认1分钟 time_out可以设置时长 可长可短 time_out=30s time_out=30ms time_out=1m  
        超时则写入失败
        格式 PUT test_index/type/1?time_out=30ms
    
## 如何进行版本控制  
* 乐观锁并发控制方案  
    基于版本号来判断，当前操作的document是否为最新，每次操作前对比当前版本号与es集群之前的差别

* 悲观锁并发控制方案  
    在各种情况下都上锁，上锁之后，就只有一个线程可以操作这一条数据了，不同场景下锁不同，如：行锁，表锁，读锁，写锁等。
    
### es基于乐观锁实现的版本控制
* 基于系统自带的版本号
    * 老版本 进行写操作时，在操作语句后面添加_version=n(n:表示es集群中document的版本号)，与集群中的version比较，小于版本号时不执行更新操作
    * 新版本 在写操作时，取消_version字段，改为 if_seq_no=n&if_primary_term来比较版本，取值分别取查询结果中的_seq_no和_primary_term
```shell
{
  "_index" : "test_index",
  "_type" : "type",
  "_id" : "1",
  "_version" : 4,
  "_seq_no" : 4,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "test_fild" : "test1-3"
  }
}
```

* 业务所用的版本号  
   在写操作时，添加version=5&version_type=external 这里的5位外部业务系统控制的版本号为5
   
### partial update乐观锁并发控制  
partial update 内部会自动执行 乐观锁的并发控制策略 如果发现版本号不一致时，partial update 会自动 fail掉

重试策略： retry策略
* 再次获取document数据和最新的版本号  
* 基于最新的版本号再次去更新，如果成功就return
* 如果失败了，重复执行1和2步骤，具体重试次数,可通过设置retry的参数指定执行次数