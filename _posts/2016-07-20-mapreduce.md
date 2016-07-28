---
layout: post
title: 大数据时代的MapReduce
description: "大数据时代的MapReduce"
tags: [Bigdata]
categories: [Bigdata]
---
 
 
![1](/images/mapreduce/3.ipg) 
  

大名鼎鼎的[MapReduce:Simplified Data Processing on Large Clusters](http://research.google.com/archive/mapreduce.html "MapReduce:Simplified Data Processing on Large Clusters")文章就开门见山地说明了MapReduce是干嘛用的  > MapReduce is a programming model and associated implementation for processing and generating large data sets.

谷歌在创业之初，提出了一个从海量文档中做倒排索引的聪明方法--Map-Reduce（映射-归约），正是它，协调若干万台电脑，并行计算，完成了倒排表的构建与维护，使谷歌在求多求快的竞争中立于不败之地。  

MapReduce是由Google提出的一种软件架构，用于大规模数据的并行计算。Map和Reduce这两个概念，是从函数式编程语言中借鉴过来的。正如Google MapReduce Paper中所描述，MapReduce是这样一个过程：输入是Key/Value对A,用户指定一个Map函数来处理A，得到一个中间结果Key/Value集合B，再由用户指定的Reduce函数来把B中相同Key的Value归并到一起，计算得到最终的结果集合C，这就是MapReduce的基本原理，可以简单的表达为：  
map (k1, v1) -> list (k2, v2)  
reduce (k2, list(v2)) -> list (v2)  



如果用简单的话来描述MapReduce的话就是如下的句子：  

>We want to count all the books in the library. You count up shelf #1, I count up shelf #2. That's map. The more people we get, the faster it goes. 我们要数图书馆中的所有书。你数1号书架，我数2号书架。这就是 “Map”。我们人越多，数书就更快。
Now we get together and add our individual counts. That's reduce. 现在我们到一起，把所有人的统计数加在一起。这就是 “Reduce”。


MapReduce特性:

* hide messy details of parallelization
* fault-tolerance
* data distribution
* load balancing


这些按照时间顺序包括：输入分片（input split）、map阶段、combiner阶段、shuffle阶段和reduce阶段。

Map:
对数据进行切片，处理错误的数据

Combiner Function: combiner是一个可选的功能，mapreduce允许combiner在数据进行传输之前去整合数据，combiner功能在每台运行map的机器上执行，在combiner处理完数据之后，数据再被reduce所处理。

Reduce:指定并发的Reduce（归约）函数

![1](/images/mapreduce/1)




Execution overview:

![2](/images/mapreduce/2.png)



关于拓展性:

MapReduce computation processes many terabytes of data on thousands of machines.




