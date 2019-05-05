---
layout: post
title: PHP7中数组的一些改进
description: PHP7中数组的一些改进
tags: [C, PHP, code]

comments: true
share: true
---
PHP的数组是一个 `有序字典` ，特性是有序、可通过键查找。
为了两个特性，PHP5通过两个双向链实现，一个双链解决全局有序，一个双链解决键hash冲突。
简单概括PHP7这方面的改进就是抛弃链结构改为数组索引。

下图是初始化的空packed array结构（packed array 和 hash array以后有机会在写吧）

![packed array](//i.loli.net/2019/05/05/5cce45d8a4d49.jpg)
	￼￼

这里有三个概念HashTable, solt, bucket。HashTable数组本体，slot存储索引，bucket实际数据的存储结构。图中的 *arData 指向的是一段连续的内存，起点位置是bucket的开始地址，往前（负索引）是slot的地址，这块内存同时存放了solt和bucket结构。

取值的流程是：key通过hash算法再和掩码nTableMask(负数的nTableSize)做位或运算得到slot的索引，slot取出bucket数组的下标，通过下标就能定位到对应的bucket。 （上图slot值为-1是因为packed array没有使用slot）

如何保证有序性？bucket数组本身是有序，添加数据的时候也是按照顺序加入。如何解决hash冲突？当冲突的时候slot记录新的下标值，新加入的bucket val.u2.next 记录了上一个bucket的下标。

参考：《PHP 7底层设计与源码实现》