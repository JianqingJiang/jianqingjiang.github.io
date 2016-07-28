---
layout: post
title: 大数据时代的MapReduce
description: "大数据时代的MapReduce"
tags: [Bigdata]
categories: [Bigdata]
---
 
 
![3](/images/mapreduce/3.jpg) 
  

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



关于拓展性,文章中有这么一句话：  

>MapReduce computation processes many terabytes of data on thousands of machines.


##   一些python的实现:

###  Map
我们先看map。map()函数接收两个参数，一个是函数，一个是序列，map将传入的函数依次作用到序列的每个元素，并把结果作为新的list返回。  

举例说明，比如我们有一个函数f(x)=x2，要把这个函数作用在一个list [1, 2, 3, 4, 5, 6, 7, 8, 9]上，就可以用map()实现如下：  



```
>>> def f(x):
...     return x * x
...
>>> map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
[1, 4, 9, 16, 25, 36, 49, 64, 81]
```

map()传入的第一个参数是f，即函数对象本身。  

你可能会想，不需要map()函数，写一个循环，也可以计算出结果：  

```
L = []
for n in [1, 2, 3, 4, 5, 6, 7, 8, 9]:
    L.append(f(n))
print L
```

的确可以，但是，从上面的循环代码，能一眼看明白“把f(x)作用在list的每一个元素并把结果生成一个新的list”吗？  

所以，map()作为高阶函数，事实上它把运算规则抽象了，因此，我们不但可以计算简单的f(x)=x2，还可以计算任意复杂的函数，比如，把这个list所有数字转为字符串：  

```
>>> map(str, [1, 2, 3, 4, 5, 6, 7, 8, 9])
['1', '2', '3', '4', '5', '6', '7', '8', '9']
```

只需要一行代码。

### Reduce

再看reduce的用法。reduce把一个函数作用在一个序列[x1, x2, x3...]上，这个函数必须接收两个参数，reduce把结果继续和序列的下一个元素做累积计算，其效果就是：  

reduce(f, [x1, x2, x3, x4]) = f(f(f(x1, x2), x3), x4)  
比方说对一个序列求和，就可以用reduce实现：  

```
>>> def add(x, y):
...     return x + y
...
>>> reduce(add, [1, 3, 5, 7, 9])
25
```

当然求和运算可以直接用Python内建函数sum()，没必要动用reduce。  

但是如果要把序列[1, 3, 5, 7, 9]变换成整数13579，reduce就可以派上用场：  

```
>>> def fn(x, y):
...     return x * 10 + y
...
>>> reduce(fn, [1, 3, 5, 7, 9])
13579
```

