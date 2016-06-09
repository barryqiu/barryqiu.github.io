---
title: hadoop  for 数据分析师
layout: post
categories: 大数据和高并发
---

hadoop的核心有两个东西：

*  HDFS
*  Map Reduce运算模型

# HDFS

## 什么是HDFS，有什么用？

hadoop集群的文件系统，说白了就是存储数据的地方，hadoop是一个集群，很多台机器，我们要用它来跑数就先得把数据给它，最常见的就是数据文件的格式，例如txt或者csv之类的，然后它运算完之后的结果肯定也得写到文件里面去（大数据的运算查询不可能把结果全部放在显示器上显示的，放不下也）。这么多台机器我们一台台去传数据回头再一台台去down数据肯定不现实，那么HDFS就是提供这个功能的，你把文件放到其中一台服务器，HDFS能按照运算需要把数据自动分发然后运算完成之后又能把数据汇总起来到一个机器上方便查看结果，所以HDFS对于hadoop来说很重要。

## 对于HDFS，一个数据分析师需要掌握什么？

HDFS主要就是用于管理数据文件，数据分析需要掌握基础的HDFS的命令，例如复制（cp），移动（mv），显示（ls）能够进行数据文件的操作即可，其原理什么的可以不用关心

# MR运算模型

## 什么是MR运算模型

Map Reduce简称MR，是一种分治法的思想的运算模型它是Google提出的一个软件架构，用于大规模数据集（大于1TB）的并行运算。具体实现是指定一个Map（映射）函数，用来把一组键值对映射成一组新的键值对，指定并发的Reduce（归纳）函数，用来保证所有映射的键值对中的每一个共享相同的键组，这里可以参考之前见过的单词统计的那个例子（word count）。
下面这张图可以比较友好的说明map-reduce的过程。

![](http://i.imgur.com/uJyC7uX.jpg)

上图中，分成了两个map进程，在进程内部进行统计，然后reduce统计结果。

再引用网上其他人用最简单的语言解释map reduce;

```
1、我们要数图书馆中的所有书。你数1号书架，我数2号书架。这就是“Map”。我们人越多，数书就更快。
2、现在我们到一起，把所有人的统计数加在一起。这就是“Reduce”。
```

## hadoop中的MR

hadoop 就是用MR模型来组织运算的，在hadoop中，首先会有调度者选了一些机器（通常数目比较多）让他们去做map，待所有的map结束后，hadoop会将中间的资料进行整理和排序（这里通常会使用hash），整理排序的结果就是把相同的KEY放在一起，然后hadoop的调度者再选取一些机器去进行reduce，最后把结果统计到一起。

![](http://i.imgur.com/ow25Um1.jpg)

## 我们需要做什么

hadoop帮我们做好了选取机器map，统计整理中间结果，再选取机器reduce，那么我们作为hadoop的使用者，需要做的就是写好map和reduce的方法提供给hadoop就可以了，下面以word count为例子讲解一下map和reduce函数。

Map 函数

```r
f <- file(description="stdin")
words <- scan(f, character(0), quiet=T)
tab = table(words)

for(i in 1:length(tab)){
  write.table(cbind(names(tab)[i], tab[[i]]),
              stdout(),row.names=F,col.names=F,quote=F,sep="\t")
}
```

Reduce 函数

```
f <- file(description="stdin")
input <- read.table(f,sep="\t",header=F,stringsAsFactors=F,na.strings = "\\N")

for(i in unique(input[,1])){
  write.table(cbind(i, sum(input[input[,1]==i,2])),
              stdout(),row.names=F,col.names=F,quote=F,sep="\t")
}
```


# 写在最后

R语言，数据分析师常用的语言，那么它能不能和Hadoop结合在一起呢？答案是absolutely yes！

["R语言为Hadoop注入统计血脉"](http://blog.fens.me/r-hadoop-intro/)

如果你看到这儿的，那一定是真爱，下面最后再讲解一款神器：

[SparkR：数据科学家的新利器](http://www.csdn.net/article/2015-10-23/2826010)

Spark是比Hadoop更牛逼的大数据运算集群，跟Hadoop的主要区别在于：
1、Spark支持更加丰富的运算模型，而Hadoop仅仅可以执行MR
2、Spark的所有运算结果是全内存的，Hadoop的每一步中间结果都会写到文件里面然后下一步再从文件中读取，我们都知道磁盘读取和内存读取速度上巨大的差异，所以Sprak比Hadoop快的多！


更多内容网上都有！