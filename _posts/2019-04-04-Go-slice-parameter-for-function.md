---
layout: post
title: Go语言中slice作为函数参数
description: Go语言中slice作为函数参数
tags: [Go, slice, code]

comments: true
share: true
---

<b>Go语言函数参数是值传递</b>
>from: https://golang.org/ref/spec#Calls   
>After they are evaluated, the parameters of the call are passed by value to the function and the called function begins execution. 

一个简单的例子
{% highlight go %}
a := make([]int, 1, 2); // len=1, cap=2
fmt.Println(a) // print: [0]
foo(a)
fmt.Println(a) // print: [0]

func foo(s []int)  {
    s = append(s, 12)
}
{% endhighlight %}
函数`foo`对slice进行了append操作，但是并没改变它的值

再看一个例子
{% highlight go %}
a := make([]int, 1, 2); // len=1, cap=2
fmt.Println(a) // print: [0]
foo(a)
fmt.Println(a) // print: [1]

func foo(s []int)  {
    s[0] = 1
}
{% endhighlight %}
函数`foo`修改了slice元素影响到了外部变量，说好的传值呢？！
![](//i.loli.net/2019/05/05/5cce44d392365.jpg)

<!-- more -->

看看slice的底层实现
{% highlight go %}
// src/runtime/slice.go#L13
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
{% endhighlight %}
其实是一个struct，slice的指针指向第一个字段的底层数组元素的地址，使用`%p`打印的也是这个数组的地址，
以下代码我们验证这个观点。
{% highlight go %}
a := make([]int, 1, 2)
p := (*uintptr)(unsafe.Pointer(&a))
fmt.Printf("a point: %p, a addr:%p \n %p, 0x%x\n", &a, a, p, *p)
// print a point: 0xc0000b82e0, a addr:0xc0000be280 
//       0xc0000b82e0, 0xc0000be280
{% endhighlight %}

array是存储数据的数组指针，len是长度，cap是内容。我们大胆猜测
参数传递是值，参数s变量是a变量的拷贝，它们字段值是相同的。
{% highlight go %}
a := make([]int, 1, 2)
fmt.Printf("out1: %v, array addr:%p, slice addr:%p %d, %d\n", a, a, &a, cap(a), len(a))

foo(a)

fmt.Printf("out2: %v, array addr:%p, slice addr:%p %d, %d\n", a, a, &a, cap(a), len(a))

func foo(s []int)  {
    fmt.Printf("in1: %v, array addr:%p, slice addr:%p %d, %d\n", s, s, &s, cap(s), len(s))
    s = append(s, 12)
    fmt.Printf("in2: %v, array addr:%p, slice addr:%p %d, %d\n", s, s, &s, cap(s), len(s))
}
/*
out1: [0], array addr:0xc000092290, slice addr:0xc00008a360 2, 1
in1: [0], array addr:0xc000092290, slice addr:0xc00008a3c0 2, 1
in2: [0 12], array addr:0xc000092290, slice addr:0xc00008a3c0 2, 2
out2: [0], array addr:0xc000092290, slice addr:0xc00008a360 2, 1
*/
{% endhighlight %}
`out1`和`in1`对比，除了slice的地址改变其他值都和外部的相对，验证了以上观点。

上面例子foo函数对slice进行了append操作，因为初始化cap=2，len=1，所以cap够用没有申请新的array，因此操作的还是同一个array。`in2`的输出数字12已经append到了array，并且len变成了2，但因为参数s变量是外部a变量的拷贝，因此`out2`的len依然是1，输出的内容也没有12。再次大胆猜测因为len的缘故导致12没被输出呢，尝试修改a变量的len。

{% highlight go %}
a := make([]int, 1, 2)
fmt.Printf("out1: %v, %d, %d\n", a, cap(a), len(a))

foo(a)
fmt.Printf("out2: %v, %d, %d\n", a, cap(a), len(a))

// 取得slice地址 + 指针长度 = len地址
lenAddr := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&a)) + unsafe.Sizeof(uintptr(1))));
*lenAddr = 2 // 修改len的值
fmt.Printf("out3: %v, %d, %d\n", a, cap(a), len(a))

func foo(s []int)  {
    s = append(s, 12)
}

/*
out1: [0], 2, 1
out2: [0], 2, 1
out3: [0 12], 2, 2
*/
{% endhighlight %}
顺利的把12也打印出来了。

<b>总结</b>   
slice当作参数传递的时候，形参是实参slice的拷贝，array地址，len，cap均相等；    
函数内对slice进行修改，没有因为容量不足而触发array重新分配，会影响到实参。    
以上是我对Go语言slice的一些看法，可能有误望指出。

<b>参考</b>
* 《Go程序设计语言》
* [Package unsafe](https://golang.org/pkg/unsafe/)