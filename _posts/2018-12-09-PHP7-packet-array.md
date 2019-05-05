---
layout: post
title: PHP7中的packed array
description: PHP7中数组的一些改进
tags: [C, PHP, code]

comments: true
share: true
---
上次提到PHP7的数组分为packed array 和 hash array，区别在于packed array取值不需要通过slot获得arData的下标，它的key对应的就是下标，所以取值速度上更有优势和比较省内存。

packed array要满足的条件是key必须是递增的正整型。
{% highlight php %}
<?php
$arr = [1, 2, 3]; // 是
$arr = [ 1 => 1, 3 => 3]; // 是(跨下标的有例外)
$arr = [-1 => 1, 0 => 2]; // 不是
$arr = [2 => 1, 1 => 2]; // 不是

{% endhighlight %}

<b>转换为packed array的操作</b>   
shuffle，array_keys等函数最终都会调用 zend_hash_to_packed 转换为packed array。


<b>转换为hash array的操作</b>   
1. 往packed array添加string key的操作
2. 添加的key是整型但是之前这个key被unset过（Z_TYPE == IS_UNDEF 状态）
3. 添加的key 大于nTableSize（数组容量）并且 (key >> 1) > nTableSize || (nTableSize >> 1) > nNumOfElements 这里😂太拗口我就直接贴代码好了

关于②PHP数组unset操作并不会立马删除指定的bucket，而且把它标识为`IS_UNDEF`的状态，等待rehash或者扩容的时候再处理（这是后话，有机会再填坑吧）。  
比如有一个packed array 
{% highlight php %}
<?php
$arr = [1, 2, 3];
unset($arr[1]);
$arr[1] = 4; 
// output
$arr == [0 => 1, 2 => 3, 1 => 4]
{% endhighlight %}
这时候就已经转化成了hash array

关于③先要各个字段之间的关系是，数组容量(nTableSize，初始化数组容量是8) = 已使用(nNumUsed) + 未使用， 已使用(nNumUsed) = 有效(nNumOfElements，我们count数组得到的就是这个大小) + 无效(unset的) 。结合上面的描述得到 $arr = [0 => 1, 8 => 2]; 是一个hash array。

![](//i.loli.net/2019/05/05/5cce45be335b7.jpg)

打印出这时候的HashTable可以看到，u.flag = 26 & HASH_FLAG_PACKED(4) == 0 已经不是packed array了，虽然我们第二个元素下标是8超过了容量nTableSize，但是这时候并没有扩容，而是转为hash array使得内存使用边得更紧凑。

参考：
* 《PHP 7底层设计与源码实现》
* PHP7.2-SRC
* nikic 博客