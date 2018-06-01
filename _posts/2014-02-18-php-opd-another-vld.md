---
layout: post
title: opd-山寨版vld
description: 使用PHP Embed SAPI实现Opcodes查看器
tags: [C, PHP, code]

comments: true
share: true
---
###起因
---
又是很久很久以前（大概1年多了）看到鸟哥的《[使用PHP Embed SAPI实现Opcodes查看器](http://www.laruence.com/2008/09/23/539.html)》，然后在Ubuntu上跟着折腾了起来，最后勉强也是跑起来了。最近找回了这份代码，打算在新笔记本上编译，然后问题就接踵而来。

###Mac上编译PHP embed sapi
---
按`Ubuntu上`的套路，直接

>$ cd php-src/  
$ ./configure --enable-embed  
$ make && make install

编译出`libphp5.so`, 但是Mac上不认这一套，后来Google到《[在mac上开启php的embed模式](http://shuoshi.me/?p=23)》才知道要编译成`libphp5.dylib`。其实按照文章我打上patch执行make提示没有相关规则，继续Google后《[在mac上开启php的embed模式](http://chengzhubo.com/post/2013-07-16/40051671070)》（别吐槽为什么名字一样，其实不是同一篇）。`autoconf`以后重新configure

>$ cd php-src/   
$ ./configure --enable-embed   
$ make libphp5.dylib && make install   

到这里动态链接库就编译好了。

<!-- more -->

###PHP相关struct的改变
---
到源码目录执行make的时候杯具又发生了
![](http://static.oschina.net/uploads/space/2014/0218/105937_x2Yb_111529.png)

赶紧翻源码看了一下(我系统PHP是brew安装的5.4.24，相应的我下载了5.4.19的源码，但是鸟哥文章中用的是5.3 alpha2 ) 

{% highlight c %}
typedef struct _znode { /* used only during compilation */ 
	int op_type;
	union {
		znode_op op;
		zval constant; /* replaced by literal/zv */
		zend_op_array *op_array;
	} u;
	zend_uint EA;      /* extended attributes */
} znode;
{% endhighlight %}

已经不在有`var`这个成员。后来下载了最新的`vld`源码，看到对应的处理宏发现5.4以后`_znode`结构都没在用了

{% highlight c %}
#if PHP_VERSION_ID >= 50399
# define OPD_ZNODE znode_op
# define OPD_ZNODE_ELEM(node,var) node.var
# define OPD_TYPE(t) t##_type
# define OPD_EXTENDED_VALUE(o) extended_value
#else
# define OPD_ZNODE znode
# define OPD_ZNODE_ELEM(node,var) node.u.var
# define OPD_TYPE(t) t.op_type
# define OPD_EXTENDED_VALUE(o) o.u.EA.type
#endif
{% endhighlight %}

接着对着vld修修改改也终于跑起来了，效果大概这样

{% highlight php %}
function foo($str) {
    echo $str;
}

$str = "hello world\n";
foo($str);
{% endhighlight %}

![](http://static.oschina.net/uploads/space/2014/0218/111141_0iNg_111529.png)

功能上当然不及`vld`，比如没有把函数，类的内部opcode dump出来什么的。
这只是个`PHP embed`模式的一个例子，发挥你的想象力应该还会有很多好玩的东西。

源码地址:[https://github.com/solupro/opd](https://github.com/solupro/opd)