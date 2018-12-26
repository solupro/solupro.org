---
layout: post
title: PHP __compare 魔术方法的实现
description: __compare 魔术方法的实现
tags: [C, PHP, code]

comments: true
share: true
---
之前吐槽过PHP为什么没`__compare`魔术方法《[PHP __compare?](//solupro.org/php-__compare/)》，可能开发组觉得没有必要吧，毕竟对象默认的比较一般情况已经够用了。 于是乎怀着no zuo no die心情尝试去实现一下，发现难度比预想要小。但由于拖延症的原因这篇文拖到现在才写，还有一方面就是修改的地方比较多和杂乱。

先看看效果吧！

{% highlight php %}
<?php
//默认情况
class Foo
{
    private $v = [];

    public function __construct(array $v) {
        $this->v = $v;
    }
}

$o1 = new Foo([1, 2, 3]);
$o2 = new Foo([2, 1, 4]);
var_dump($o1 > $o2);
/* output */
bool(false)

//添加 __compare
class Foo
{
    private $v = [];

    public function __construct(array $v) {
        $this->v = $v;
    }

    public function __compare($o) {
        return $this->v[1] > $o->v[1];
    }
}

$o1 = new Foo([1, 2, 3]);
$o2 = new Foo([2, 1, 4]);
var_dump($o1 > $o2);
/* output */
bool(true)
{% endhighlight %}

可以看出，$o1, $o2的比较行为已经被`__compare`改变

<!-- more -->

先看对象比较的实现吧，这里假设我们是有`__compare`这个魔术方法的。当两个对象进行比较时会调用`zend_std_compare_objects`这个函数，然后让函数检测对象是否注册了`__compare`，如果有就优先调用，很简单吧。

{% highlight c %}
/* PHP5.4.27 Zend/zend_object_handlers.c +1357 */
static int zend_std_compare_objects(zval *o1, zval *o2 TSRMLS_DC) /
{
	zend_object *zobj1, *zobj2;

	zobj1 = Z_OBJ_P(o1);
	zobj2 = Z_OBJ_P(o2);

	if (zobj1->ce != zobj2->ce) {
		return 1; /* different classes */
	}
    
    if (zobj1->ce->__compare) {
        zval *rv;
        rv = zend_std_call_compare(o2, o1 TSRMLS_CC);
        return Z_LVAL_P(rv) ? -1 : (Z_LVAL_P(rv) == 0 ? 1 : 0);
    }
    
	if (!zobj1->properties && !zobj2->properties) {
		int i;
......
{% endhighlight %}

这里要注意的是`zend_std_compare_objects`是谁触发的，也就是说`o1`到底是谁。在这里`o1`上面PHP代码的`$o2`，`o2`才是对应PHP的`$o1`。记得Python的`object.__cmp__`调用的是第一个对象，PHP有点不一样。因为觉得别扭所以在调用`zend_std_call_compare(o2, o1 TSRMLS_CC)`的时候我有意把参数对调了一下，当然清楚这点以后你也可以不必这么做。

还有一点就是`zend_std_compare_objects`的返回值, 大于 -> -1, 等于 -> 1, 小于 -> 0 <s>应该没记错吧</s>。`zend_std_call_compare`函数返回值是一个 `zval *` ，return 的时候需要稍微处理一下 `return Z_LVAL_P(rv) ? -1 : (Z_LVAL_P(rv) == 0 ? 1 : 0);`

{% highlight c %}
/* PHP5.4.27 Zend/zend_object_handlers.c +216 */
static zval *zend_std_call_compare(zval *object, zval *object2 TSRMLS_DC)
{
    zval *retval = NULL;
	zend_class_entry *ce = Z_OBJCE_P(object);
    
	/* __compare handler is called with one argument:
     other object
     */
    
	SEPARATE_ARG_IF_REF(object2);
    
	zend_call_method_with_1_params(&object, ce, &ce->__compare, ZEND_COMPARE_FUNC_NAME, &retval, object2);
    
	zval_ptr_dtor(&object2);
    return retval;
}
{% endhighlight %}

`zend_std_call_compare`就没什么好说了，直接`zend_call_method_with_1_params`调用已注册的`__compare`就可以了。以上就完成了对象对比的逻辑。

扯点题外话，有人可能会问我怎么知道调用了`zend_std_compare_objects`函数，我的思路是这样的。先了解PHP运行过程，引用鸟哥博客

>1.Scanning(Lexing) ,将PHP代码转换为语言片段(Tokens)   
2.Parsing, 将Tokens转换成简单而有意义的表达式  
3.Compilation, 将表达式编译成Opocdes  
4.Execution, 顺次执行Opcodes，每次一条，从而实现PHP脚本的功能。  

既然是对象，那就从`new`关键字开始，`Zend/zend_language_scanner.l`得到`T_NEW`token，`Zend/zend_language_parser.y` 观察猜测调用了`zend_do_begin_new_object`，以此为关键字在`Zend`目录搜索，`Zend/zend_compile.c +5287`发现opcode是`ZEND_NEW`，继续以此搜索，`Zend/zend_vm_def.h +3349`调用(以下文件名就不展开了,[lxr](http://lxr.php.net/)跟进就好)object_init_ex -> _object_init_ex -> _object_and_properties_init -> zend_objects_new -> 关键的一步`retval.handlers = &std_object_handlers;`, 看过上篇的应该知道`retval.handlers`是`zend_object_handlers`结构类常用操作的方法集合，而`std_object_handlers`就是默认的方法集，`zend_std_compare_objects`就包含在里面。

添加`__compare`魔术方法的过程比较杂乱枯燥，入手点就是仿照已有的魔术方法，比如你可以在[lxr](http://lxr.php.net/)搜索`__isset`。这是我当时添加代码时的笔记，以防下面讲漏，先贴一下。[http://note.youdao.com/share/?id=12a38d65c426d8ca35bbaa7ff7aff99f&type=note](http://note.youdao.com/share/?id=12a38d65c426d8ca35bbaa7ff7aff99f&type=note)

####zend.h
---

{% highlight c %}
/* PHP5.4.27 Zend/zend.h +495 */
    union _zend_function *__compare;
{% endhighlight %}

先往`_zend_class_entry`结构添加`__compare`这样我们的类就具有这个方法了

####zend_compile.h
---

{% highlight c %}
/* PHP5.4.27 Zend/zend_compile.h +826 */
#define ZEND_COMPARE_FUNC_NAME      "__compare"
{% endhighlight %}

这里定义`ZEND_COMPARE_FUNC_NAME`宏

{% highlight c %}
/* PHP5.4.27 Zend/zend_compile.h +1598 */
 else if ((name_len == sizeof(ZEND_COMPARE_FUNC_NAME)-1) && (!memcmp(lcname, ZEND_COMPARE_FUNC_NAME, sizeof(ZEND_COMPARE_FUNC_NAME)-1))) {
				if (fn_flags & ((ZEND_ACC_PPP_MASK | ZEND_ACC_STATIC) ^ ZEND_ACC_PUBLIC)) {
					zend_error(E_WARNING, "The magic method __compare() must have public visibility and cannot be static");
				}
			}
{% endhighlight %}

对`__compare`方法前缀判断（接口部分）

{% highlight c %}
/* PHP5.4.27 Zend/zend_compile.h +1652 */
 else if ((name_len == sizeof(ZEND_COMPARE_FUNC_NAME)-1) && (!memcmp(lcname, ZEND_COMPARE_FUNC_NAME, sizeof(ZEND_COMPARE_FUNC_NAME)-1))) {
				if (fn_flags & ((ZEND_ACC_PPP_MASK | ZEND_ACC_STATIC) ^ ZEND_ACC_PUBLIC)) {
					zend_error(E_WARNING, "The magic method __compare() must have public visibility and cannot be static");
				}
				CG(active_class_entry)->__compare = (zend_function *) CG(active_op_array);
			}
{% endhighlight %}

对`__compare`方法前缀判断（类部分）并`保存` (user class??)

{% highlight c %}
/* PHP5.4.27 Zend/zend_compile.h +2848 */
    if (!ce->__compare) {
		ce->__compare = ce->parent->__compare;
	}
{% endhighlight %}

如果当前没实现`__compare`方法将继承父类

{% highlight c %}
/* PHP5.4.27 Zend/zend_compile.h +3676 */
 else if (!strncmp(mname, ZEND_COMPARE_FUNC_NAME, mname_len)) {
		ce->__compare = fe;
	}
{% endhighlight %}

保存`__compare`方法(internal class??)

{% highlight c %}
/* PHP5.4.27 Zend/zend_compile.h +6633 */
ce->__compare = NULL;
{% endhighlight %}

这里是`nullify_handlers`处理，嗯，一段是`unticked_class_declaration_statement:` 调用的，我也没看懂，unticked user class初始化的时候把方法集设置为`NULL` ??? unticked_class_declaration_statement 是什么，囧，总之先把代码加上就对了。

####zend_API.h
---

{% highlight c %}
/* PHP5.4.27 Zend/zend_API.h +191 */
		class_container.__compare = NULL;                       \
{% endhighlight %}

这个宏好像是只提供给`zend_disable_class`使用，暂且设置为`NULL`吧，对我们用户层代码也不会有影响

####zend_API.C

{% highlight c %}
/* PHP5.4.27 Zend/zend_API.h +191 */
	zend_function *ctor = NULL, *dtor = NULL, *clone = NULL, *__get = NULL, *__set = NULL, *__unset = NULL, *__isset = NULL, *__call = NULL, *__callstatic = NULL, *__tostring = NULL, *__compare = NULL;              
{% endhighlight %}

添加`__compare`声明

{% highlight c %}
/* PHP5.4.27 Zend/zend_API.h +2115 */
	else if ((fname_len == sizeof(ZEND_COMPARE_FUNC_NAME)-1) && !memcmp(lowercase_name, ZEND_COMPARE_FUNC_NAME, sizeof(ZEND_COMPARE_FUNC_NAME))) {
                __compare = reg_function;
            } 
{% endhighlight %}

赋值`__compare`，这也是一个internal才会调用的方法，粗略看了下闭包什么的会用到

{% highlight c %}
/* PHP5.4.27 Zend/zend_API.h +2155 */
        scope->__compare = __compare; 
{% endhighlight %}

{% highlight c %}
/* PHP5.4.27 Zend/zend_API.h +2184 */
        if (__compare) {
			if (__compare->common.fn_flags & ZEND_ACC_STATIC) {
				zend_error(error_type, "Method %s::%s() cannot be static", scope->name, __compare->common.function_name);
			}
			__compare->common.fn_flags &= ~ZEND_ACC_ALLOW_STATIC;
		} 
{% endhighlight %}

对前缀的判断

####编译
----

以上代码都添加完毕以后到你的PHP源码目录 `./configure` && `make` 千万别<s>make install</s>除非你打算替换你系统的PHP。如果没有意外编译完成以后在 `your_php_src_path/sapi/sli`会生成一个`php`可执行文件，测试可以通过`your_php_src_path/sapi/sli/php -f phpfile.php`
