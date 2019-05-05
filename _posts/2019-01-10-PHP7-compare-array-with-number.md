---
layout: post
title: PHP7中数组和整型的比较
description: PHP7中数组和整型的比较
tags: [C, PHP, code]

comments: true
share: true
---

<b>起因</b>   
在`V2EX`看到这帖子，平时是没试过这么玩，出于好奇翻了一下源码   
![](//i.loli.net/2019/05/05/5cce453303d22.jpg)

<b>分析</b>   
以 `[] > 123;` 为例   
![](//i.loli.net/2019/05/05/5cce453358c7a.jpg)   
①略过词法分析语法分析AST生成，可以看到 op1 > op2 的操作等同于 op2 < op1，调用函数 is_smaller_function(result, op2, op1)

![](//i.loli.net/2019/05/05/5cce453368eba.jpg)   
②接着调用 compare_function， 这里的 op1 == 123，op2 == []   

![](//i.loli.net/2019/05/05/5cce45336703e.jpg)   
③ compare_function 其实就是各种类型的比较判断，引用类型转换继而比较blablabla，省略这些代码，因为当我们 op2 == IS_ARRAY 的时候都不成立，最终到达上图的判断，可以看到op1的值已经被忽略，所以无论是 123 还是 PHP_MAX_INT，result = -1 接着返回成功。回到图2，比较结果变成了 result = -1 < 0; 恒为 true。  

<b>结论</b>   
数组 > 任意数字类型 == true; 其他类型的比较也在compare_function找到，有兴趣的翻翻吧。   

<b>参考</b>
* PHP7.2-SRC