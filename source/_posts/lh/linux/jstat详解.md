---
title: jstat详解
date: 2021-10-27 00:00:00
author: lh
summary: ""
categories: linux
tags: 
    - jstat详解
---
# 什么是jstat
Jstat是JDK自带的一个轻量级小工具。全称“Java Virtual Machine statistics monitoring tool”。  
它位于java的bin目录下，主要利用JVM内建的指令对Java应用程序的资源和性能进行实时的命令行的监控，包括了对Heap size和垃圾回收状况的监控。  
可见，Jstat是轻量级的、专门针对JVM的工具，非常适用。



# jstat 命令相关参数
```text
jstat命令命令格式：
jstat [Options] vmid [interval] [count]
 
命令参数说明：
Options，一般使用 -gcutil 或  -gc 查看gc 情况
pid，当前运行的 java进程号 
interval，间隔时间，单位为秒或者毫秒 
count，打印次数，如果缺省则打印无数次
 
Options 参数如下：
-gc：统计 jdk gc时 heap信息，以使用空间字节数表示
-gcutil：统计 gc时， heap情况，以使用空间的百分比表示
-class：统计 class loader行为信息
-compile：统计编译行为信息
-gccapacity：统计不同 generations（新生代，老年代，持久代）的 heap容量情况
-gccause：统计引起 gc的事件
-gcnew：统计 gc时，新生代的情况
-gcnewcapacity：统计 gc时，新生代 heap容量
-gcold：统计 gc时，老年代的情况
-gcoldcapacity：统计 gc时，老年代 heap容量
-gcpermcapacity：统计 gc时， permanent区 heap容量
-printcompilation (HotSpot编译统计)
```

## -gc(GC堆状态) 
示例：jstat -gc 1601 5000 3
说明：每5秒一次 打印3次 显示进程号为1601的 java进成的 GC情况，结果如下图
```shell script
[root@iZ94sbzk035Z ~]# jstat -gc 1601 5000 3
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
1216.0 1216.0 188.7   0.0    9920.0   3300.8   12292.0     8239.6   24960.0 20411.5 5248.0 2736.0  22062   44.128   3      0.147   44.275
1216.0 1216.0 188.7   0.0    9920.0   3499.5   12292.0     8239.6   24960.0 20411.5 5248.0 2736.0  22062   44.128   3      0.147   44.275
1216.0 1216.0 188.7   0.0    9920.0   3500.9   12292.0     8239.6   24960.0 20411.5 5248.0 2736.0  22062   44.128   3      0.147   44.275

```
参数说明

|显示列名|具体描述|
|:---:|:---:|
|S0C|年轻代中第一个survivor（幸存区）的容量 (KB)|
|S1C|年轻代中第二个survivor（幸存区）的容量 (KB)|
|S0U|年轻代中第一个survivor（幸存区）目前已使用空间 (KB)|
|S1U|年轻代中第二个survivor（幸存区）目前已使用空间 (KB)|
|EC|年轻代中Eden（伊甸园）的容量 (KB)|
|EU|年轻代中Eden（伊甸园）目前已使用空间 (KB)|
|OC|老年代大小(KB)|
|OU|老年代目前使用大小(KB)|
|MC|方法区大小|
|MU|方法区使用大小|
|CCSC|压缩类空间大小|
|CCSU|压缩类空间使用大小|
|YGC|年轻代垃圾回收次数：从应用程序启动到采样时年轻代中gc次数|
|YGCT|年轻代垃圾回收消耗时间：从应用程序启动到采样时年轻代中gc所用时间(s)|
|FGC|老年代垃圾回收次数：从应用程序启动到采样时old代(全gc)gc次数|
|FGCT|老年代垃圾回收消耗时间：从应用程序启动到采样时old代(全gc)gc所用时间(s)|
|GCT|垃圾回收消耗总时间：从应用程序启动到采样时gc用的总时间(s)|


## -gcutil (GC统计汇总)
示例：jstat -gcutil 1601 5000 3
说明：每5秒一次 打印3次 显示进程号为1601的 java进程垃圾回收统计情况，结果如下图
```shell script
[root@iZ94sbzk035Z ~]# jstat -gcutil 1601 5000 3
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
 28.03   0.00  34.41  67.03  81.78  52.14  22064   44.131     3    0.147   44.278
 28.03   0.00  34.43  67.03  81.78  52.14  22064   44.131     3    0.147   44.278
 28.03   0.00  36.41  67.03  81.78  52.14  22064   44.131     3    0.147   44.278

```
参数说明：

|显示列名|具体描述|
|:---:|:---:|
|S0|年轻代中第一个survivor（幸存区）的容量 (KB)|
|S1|年轻代中第二个survivor（幸存区）的容量 (KB)|
|E|伊甸园区使用比例|
|O|老年代使用比例|
|M|元数据区使用比例|
|CCS|压缩使用比例|
|YGC|年轻代垃圾回收次数|
|YGCT|从应用程序启动到采样时年轻代中gc所用时间(s)|
|FGC|老年代垃圾回收次数|
|FGCT|老年代垃圾回收消耗时间|
|GCT|垃圾回收消耗总时间|


## -class(类加载器) 
示例：每5秒一次 打印3次 显示进程号为1601的加载class的数量，及所占空间等信息
```shell script
[root@iZ94sbzk035Z ~]# jstat -class 1601 5000 3
Loaded  Bytes  Unloaded  Bytes     Time   
  6615 11426.1     2417  3735.7       6.31
  6615 11426.1     2417  3735.7       6.31
  6615 11426.1     2417  3735.7       6.31

```
参数说明：

|显示列名|具体描述|
|:---:|:---:|
|Loaded|装载的类的数量|
|Bytes|装载类所占用的字节数|
|Unloaded|卸载类的数量|
|Bytes|卸载类的字节数|
|Time|装载和卸载类所花费的时间|

## -compiler (JIT,统计编译行为信息)
示例：每5秒一次 打印3次 显示进程号为1601的显示JVM实时编译的数量等信息
```shell script

[root@iZ94sbzk035Z ~]# jstat -compiler 1601 5000 3
Compiled Failed Invalid   Time   FailedType FailedMethod
       -      -       -        -          - -           
       -      -       -        -          - -           
       -      -       -        -          - -  
```
参数说明：

|显示列名|具体描述|
|:---:|:---:|
|Compiled|编译任务执行数量|
|Failed|编译任务执行失败数量|
|Invalid|编译任务执行失效数量|
|Time|编译任务消耗时间|
|FailedType|最后一个编译失败任务的类型|
|FailedMethod|最后一个编译失败任务所在的类及方法|


## -gccapacity (各区大小) 
    统计不同 generations（新生代，老年代，持久代）的 heap容量情况
示例：每5秒一次 打印3次 显示进程号为1601的显示JVM内存中三代（young,old,perm）对象的使用和占用大小
```shell script
[root@iZ94sbzk035Z ~]# jstat -gccapacity  1601 5000 3
 NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN      OGCMX       OGC         OC       MCMN     MCMX      MC     CCSMN    CCSMX     CCSC    YGC    FGC 
  8192.0  16384.0  12352.0 1216.0 1216.0   9920.0     8192.0    16384.0    12292.0    12292.0      0.0 1069056.0  24960.0      0.0 1048576.0   5248.0  22082     3
  8192.0  16384.0  12352.0 1216.0 1216.0   9920.0     8192.0    16384.0    12292.0    12292.0      0.0 1069056.0  24960.0      0.0 1048576.0   5248.0  22082     3
  8192.0  16384.0  12352.0 1216.0 1216.0   9920.0     8192.0    16384.0    12292.0    12292.0      0.0 1069056.0  24960.0      0.0 1048576.0   5248.0  22082     3

```
参数说明：

|显示列名|具体描述|
|:---:|:---:|
|NGCMN|年轻代(young)中初始化(最小)的大小(KB)|
|NGCMX|年轻代(young)的最大容量 (KB)|
|NGC|年轻代(young)中当前的容量 (KB)|
|S0C|年轻代中第一个survivor（幸存区）的容量 (KB)|
|S1C|年轻代中第二个survivor（幸存区）的容量 (KB)|
|EC|年轻代中Eden（伊甸园）的容量 (KB)|
|OGCMN|old代中初始化(最小)的大小 (KB)|
|OGCMX|old代的最大容量(KB)|
|OGC|old代当前新生成的容量 (KB)|
|OC|Old代的容量 (KB)|
|YGC|从应用程序启动到采样时年轻代中gc次数|
|FGC|从应用程序启动到采样时old代(全gc)gc次数|

## -gccause(统计引起 gc的事件)
示例：每5秒一次 打印3次 显示进程号为1601统计引起 gc的事件
```shell script
[root@iZ94sbzk035Z ~]# jstat -gccause  1601 5000 3
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC                 
 22.08   0.00  97.71  67.04  81.78  52.14  22088   44.176     3    0.147   44.323 Allocation Failure   No GC               
 22.08   0.00  97.75  67.04  81.78  52.14  22088   44.176     3    0.147   44.323 Allocation Failure   No GC               
 22.08   0.00 100.00  67.04  81.78  52.14  22088   44.176     3    0.147   44.323 Allocation Failure   No GC 
```
参数说明：

|显示列名|具体描述|
|:---:|:---:|


## -gcnew：(统计新生代的情况)
示例：每5秒一次 打印3次 显示进程号为1601的显示JVM内存中三代（young,old,perm）对象的使用和占用大小
```shell script
[root@iZ94sbzk035Z ~]# jstat -gcnew 1601 5000 3
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT  
1216.0 1216.0    0.0  276.7 15  15  608.0   9920.0   8430.5  22089   44.178
1216.0 1216.0    0.0  276.7 15  15  608.0   9920.0   9491.0  22089   44.178
1216.0 1216.0    0.0  276.7 15  15  608.0   9920.0   9492.5  22089   44.178
```
参数说明：

|显示列名|具体描述|
|:---:|:---:|
|S0C|年轻代中第一个survivor（幸存区）的容量 (KB)|
|S1C|年轻代中第二个survivor（幸存区）的容量 (KB)|
|S0U|年轻代中第一个survivor（幸存区）目前已使用空间 (KB)|
|S1U|年轻代中第二个survivor（幸存区）目前已使用空间 (KB)|
|TT|持有次数限制|
|MTT|最大持有次数限制|
|DSS||
|EC|年轻代中Eden（伊甸园）的容量 (KB)|
|EU|年轻代中Eden（伊甸园）目前已使用空间 (KB)|
|YGC|从应用程序启动到采样时年轻代中gc次数|
|YGCT|从应用程序启动到采样时年轻代中gc所用时间(s)|

## -gcnewcapacity(统计新生代 heap容量)
示例：每5秒一次 打印3次 显示进程号为1601的显示年轻代对象的信息及其占用量
```shell script
[root@iZ94sbzk035Z ~]# jstat -gcnewcapacity 1601 5000 3
  NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC 
    8192.0    16384.0    12352.0   1600.0   1216.0   1600.0   1216.0    13184.0     9920.0 22119     3
    8192.0    16384.0    12352.0   1600.0   1216.0   1600.0   1216.0    13184.0     9920.0 22119     3
    8192.0    16384.0    12352.0   1600.0   1216.0   1600.0   1216.0    13184.0     9920.0 22119     3

```
参数说明：

|显示列名|具体描述|
|:---:|:---:|
|NGCMN|年轻代(young)中初始化(最小)的大小(KB)|
|NGCMX|年轻代(young)的最大容量 (KB)|
|NGC|年轻代(young)中当前的容量 (KB)|
|S0CMX|年轻代中第一个survivor（幸存区）的最大容量 (KB)|
|S0C|年轻代中第一个survivor（幸存区）的容量 (KB)|
|S1CMX|年轻代中第二个survivor（幸存区）的最大容量 (KB)||
|S1C|年轻代中第二个survivor（幸存区）的容量 (KB)|
|ECMX|年轻代中Eden（伊甸园）的最大容量 (KB)|
|EC|年轻代中Eden（伊甸园）的容量 (KB)|
|YGC|从应用程序启动到采样时年轻代中gc次数|
|FGC|从应用程序启动到采样时old代(全gc)gc次数|


## -gcold(统计老年代的情况)
示例：每5秒一次 打印3次 显示进程号为1601的显示old代对象的信息
```shell script
[root@iZ94sbzk035Z ~]# jstat -gcold 1601 5000 3
   MC       MU      CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT   
 24960.0  20411.5   5248.0   2736.0     12292.0      8243.2  22120     3    0.147   44.381
 24960.0  20411.5   5248.0   2736.0     12292.0      8243.2  22120     3    0.147   44.381
 24960.0  20411.5   5248.0   2736.0     12292.0      8243.2  22120     3    0.147   44.381

```
参数说明：

|显示列名|具体描述|
|:---:|:---:|
|MC||
|MU||
|CCSC||
|CCSU||
|OC|Old代的容量 (KB)|
|OU|Old代目前已使用空间 (KB)|
|YGC|从应用程序启动到采样时年轻代中gc次数|
|FGC|从应用程序启动到采样时old代(全gc)gc次数|
|FGCT|从应用程序启动到采样时old代(全gc)gc所用时间(s)|
|GCT|从应用程序启动到采样时gc用的总时间(s)|

## -gcoldcapacity(统计老年代 heap容量)
示例：每5秒一次 打印3次 显示进程号为1601的显示old代对象的信息及其占用量。
```shell script
[root@iZ94sbzk035Z ~]# jstat -gcoldcapacity 1601 5000 3
   OGCMN       OGCMX        OGC         OC       YGC   FGC    FGCT     GCT   
     8192.0     16384.0     12292.0     12292.0 22122     3    0.147   44.385
     8192.0     16384.0     12292.0     12292.0 22122     3    0.147   44.385
     8192.0     16384.0     12292.0     12292.0 22122     3    0.147   44.385
```
参数说明：

|显示列名|具体描述|
|:---:|:---:|
|OGCMN|old代中初始化(最小)的大小 (KB)|
|OGCMX|old代的最大容量(KB)|
|OGC|old代当前新生成的容量 (KB)|
|OC|Old代的容量 (KB)|
|YGC|从应用程序启动到采样时年轻代中gc次数|
|FGC|从应用程序启动到采样时old代(全gc)gc次数|
|FGCT|从应用程序启动到采样时old代(全gc)gc所用时间(s)|
|GCT|从应用程序启动到采样时gc用的总时间(s)|
