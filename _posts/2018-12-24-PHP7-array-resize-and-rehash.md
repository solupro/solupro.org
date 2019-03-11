---
layout: post
title: PHP7数组扩容和rehash
description: PHP7数组扩容和rehash
tags: [C, PHP, code]

comments: true
share: true
---
<b>TL;DR</b>  
<b>数组初始化容量为8，最大是0x80000000，每次扩容为原容量的2倍。</b>

![](//ww1.sinaimg.cn/large/65fcc0d7gy1fz1nxgy4ajj20ww0lwn25.jpg)

<b>扩容</b>  
数组插入新元素的时候会调用`ZEND_HASH_IF_FULL_DO_RESIZE`判断已使用容量是否大于等于数组容量（ht->nNumUsed >= ht->nTableSize）;  
条件成立再判断已使用是否大于有效元素加上有效元素除以32（ht->nNumUsed > ht->nNumOfElements + (ht->nNumOfElements >> 5)），符合这条件的先进行`rehash`整理，不符合条件的判断数组容量是否小于HT_MAX_SIZE（0x80000000）;  
如果是则容量扩大至原来的2倍，uint32_t nSize = ht->nTableSize + ht->nTableSize; 在内存池申请 nSize* sizeof(Bucket) + nSize* sizeof(uint32_t)大小的新数据地址;  
设置新的nTableSize和nTableMask，修改ht->arData指向新地址，把旧Bucket数据拷贝到新地址，释放旧地址内存，执行`zend_hash_rehash`。

<b>rehash</b>  
首先调用HT_HASH_RESET重置所有索引为-1；  
如果数组所有元素都是有效元素则按arData数组顺序重新分配索引，原索引存在元素值zval.u2.next；  
否则循环遍历bucket至nNumUsed，去除IS_UNDEF的元素，更新nNumUsed。


<b>参考</b>
* 《PHP 7底层设计与源码实现》
* PHP7.2-SRC