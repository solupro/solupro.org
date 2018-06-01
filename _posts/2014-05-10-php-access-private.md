---
layout: post
title: PHP 一种访问私有属性的方法
description: 按鸟哥博客的原话就是“PHP通过这种比较ugly但是简单高效的方法, 实现了对属性访问权限的标识.知道了这个, 我们就可以干一些不合常理的事请, 比如访问对象的私有/保护属性”
tags: [C, PHP, code]

comments: true
share: true
---
在鸟哥《[深入理解PHP原理之对象(一)](http://www.laruence.com/2010/05/18/1482.html)》看到一段挺有意思的代码

>“PHP通过这种比较ugly但是简单高效的方法, 实现了对属性访问权限的标识.知道了, 我们就可以干一些不合常理的事请, 比如访问对象的私有/保护属性”

{% highlight php %}
<?php
class Foo {
    private $_name = "laruence";
    protected $_age = 28;
}
$foo = new Foo();
$arr = (array) $foo;
var_dump($arr["\0Foo\0_name"]);
var_dump($arr["\0*\0_age"]);
//output:
string(8) "laruence"
int(28)
{% endhighlight %}

 至于这算不算BUG本文就不议论了，有兴趣的可以看看这里 (见:  [Bug #44273 access to private and protected class variables allowed when casting to array](https://bugs.php.net/bug.php?id=44273) ):  下面有相关议论

 <!-- more -->
 
那为什么对象转成数组以后能通过构建特殊的key去访问呢？直接看关键代码吧! (已下代码PHP版本为:  5.4.27)

{% highlight c %}
/* Zend/zend_API.c +3361*/
ZEND_API int zend_declare_property_ex(zend_class_entry *ce, const char *name, int name_length, zval *property, int access_type, const char *doc_comment, int doc_comment_len TSRMLS_DC)
{
.....
    switch (access_type & ZEND_ACC_PPP_MASK) {
        case ZEND_ACC_PRIVATE: {
                char *priv_name;
                int priv_name_length;

                zend_mangle_property_name(&priv_name, &priv_name_length, ce->name, ce->name_length, name, name_length, ce->type & ZEND_INTERNAL_CLASS);
                property_info.name = priv_name;
                property_info.name_length = priv_name_length;
            }
            break;
        case ZEND_ACC_PROTECTED: {
                char *prot_name;
                int prot_name_length;

                zend_mangle_property_name(&prot_name, &prot_name_length, "*", 1, name, name_length, ce->type & ZEND_INTERNAL_CLASS);
                property_info.name = prot_name;
                property_info.name_length = prot_name_length;
            }
            break;
        case ZEND_ACC_PUBLIC:
            if (IS_INTERNED(name)) {
                property_info.name = (char*)name;
            } else {
                property_info.name = ce->type & ZEND_INTERNAL_CLASS ? zend_strndup(name, name_length) : estrndup(name, name_length);
            }
            property_info.name_length = name_length;
            break;
    }
.....
	zend_hash_quick_update(&ce->properties_info, name, name_length+1, h, &property_info, sizeof(zend_property_info), NULL);

	return SUCCESS;
}
{% endhighlight %}

  可以看到当属性是private和protected时候调用`zend_mangle_property_name`来构造`property_info.name` (转换后的key)，而它们第3，4个参数也不一样，private传入的是`类名`，而protected传入的是`*`字符。在看看`zend_mangle_property_name`函数的实现就一目了然了。

{% highlight c %}
/* Zend/zend_compile.c +5037 */
ZEND_API void zend_mangle_property_name(char **dest, int *dest_length, const char *src1, int src1_length, const char *src2, int src2_length, int internal)
{
	char *prop_name;
	int prop_name_length;

	prop_name_length = 1 + src1_length + 1 + src2_length;
	prop_name = pemalloc(prop_name_length + 1, internal);
	prop_name[0] = '\0';
	memcpy(prop_name + 1, src1, src1_length+1);
	memcpy(prop_name + 1 + src1_length + 1, src2, src2_length+1);

	*dest = prop_name;
	*dest_length = prop_name_length;
}
{% endhighlight %}

 所以对象的private属性我们可以强制转换为array后通过`$obj["\0类名\0属性名"]`来访问，而protected则是`$obj["\0*\0属性名"]`

至于流程可以通过`zend_do_declare_property`关键字到 [http://lxr.php.net/](http://lxr.php.net/) 搜索, 大概就是
  
* zend_do_declare_property
* zend_declare_property_ex
* zend_mangle_property_name

嗯，好像也没写什么的样子，一些细节比如property_info被zend_hash_quick_update, 类型的转换都没理解清楚。