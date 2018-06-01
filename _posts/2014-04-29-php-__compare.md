---
layout: post
title: PHP __compare?
description: 在看《PHP之道》时某代码引起的一些思考
tags: [C, PHP, code]

comments: true
share: true
---
在看《[PHP之道](http://wulijun.github.io/php-the-right-way/)》时有这么一段代码

{% highlight php %}
<?php
$raw = '22. 11. 1968';
$start = \DateTime::createFromFormat('d. m. Y', $raw);
echo 'State date: ' . $start->format('m/d/Y') . "\n";

$end = clone $start;
$end->add(new \DateInterval('P1M6D'));

$diff = $end->diff($start);
echo "Difference: " . $diff->format('%m month, %d days (total: %a days)') . "\n";

if($start < $end) {
    echo "Start is before end!\n";
}
{% endhighlight %}

`$start < $end` 引起了我的注意，印象中并没有`__compare`之类的魔术方法，翻了下手册确实没找到，也没相关实现，那为什么这里可以直接比较呢？因为这是built-in class，所以需要翻一下源码。(已下代码版本是 5.4.27)

<!-- more -->

以`compare`为关键字在`ext/date/php_date.c`(DateTime在这实现)搜索，得到这么一个函数

{% highlight c %}
static int date_object_compare_date(zval *d1, zval *d2 TSRMLS_DC)
{
    if (Z_TYPE_P(d1) == IS_OBJECT && Z_TYPE_P(d2) == IS_OBJECT &&
        instanceof_function(Z_OBJCE_P(d1), date_ce_date TSRMLS_CC) &&
        instanceof_function(Z_OBJCE_P(d2), date_ce_date TSRMLS_CC)) {
        php_date_obj *o1 = zend_object_store_get_object(d1 TSRMLS_CC);
        php_date_obj *o2 = zend_object_store_get_object(d2 TSRMLS_CC);

        if (!o1->time || !o2->time) {
            php_error_docref(NULL TSRMLS_CC, E_WARNING, "Trying to compare an incomplete DateTime object");
            return 1;
        }
        if (!o1->time->sse_uptodate) {
            timelib_update_ts(o1->time, o1->time->tz_info);
        }
        if (!o2->time->sse_uptodate) {
            timelib_update_ts(o2->time, o2->time->tz_info);
        }
        
        return (o1->time->sse == o2->time->sse) ? 0 : ((o1->time->sse < o2->time->sse) ? -1 : 1);
    }    
    
    return 1;
}
{% endhighlight %}

可以看出这就是`__compare`的实现部分，通过函数名继续搜索，来到line:1950

{% highlight c %}
static void date_register_classes(TSRMLS_D)
{
	zend_class_entry ce_date, ce_timezone, ce_interval, ce_period;

	INIT_CLASS_ENTRY(ce_date, "DateTime", date_funcs_date);
	ce_date.create_object = date_object_new_date;
	date_ce_date = zend_register_internal_class_ex(&ce_date, NULL, NULL TSRMLS_CC);
	memcpy(&date_object_handlers_date, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
	date_object_handlers_date.clone_obj = date_object_clone_date;
	
	date_object_handlers_date.compare_objects = date_object_compare_date;
	
	date_object_handlers_date.get_properties = date_object_get_properties;
	date_object_handlers_date.get_gc = date_object_get_gc;
	.....
{% endhighlight %}

`date_object_handlers_date` 是一个 `zend_object_handlers`，结构是这样的

{% highlight c %}
struct _zend_object_handlers {
	/* general object functions */
	zend_object_add_ref_t					add_ref;
	zend_object_del_ref_t					del_ref;
	zend_object_clone_obj_t					clone_obj;
	/* individual object functions */
	zend_object_read_property_t				read_property;
	zend_object_write_property_t			write_property;
	zend_object_read_dimension_t			read_dimension;
	zend_object_write_dimension_t			write_dimension;
	zend_object_get_property_ptr_ptr_t		get_property_ptr_ptr;
	zend_object_get_t						get;
	zend_object_set_t						set;
	zend_object_has_property_t				has_property;
	zend_object_unset_property_t			unset_property;
	zend_object_has_dimension_t				has_dimension;
	zend_object_unset_dimension_t			unset_dimension;
	zend_object_get_properties_t			get_properties;
	zend_object_get_method_t				get_method;
	zend_object_call_method_t				call_method;
	zend_object_get_constructor_t			get_constructor;
	zend_object_get_class_entry_t			get_class_entry;
	zend_object_get_class_name_t			get_class_name;
	zend_object_compare_t					compare_objects;
	zend_object_cast_t						cast_object;
	zend_object_count_elements_t			count_elements;
	zend_object_get_debug_info_t			get_debug_info;
	zend_object_get_closure_t				get_closure;
	zend_object_get_gc_t					get_gc;
};
{% endhighlight %}

所以只有实现`zend_object_handlers.compare_objects`才具备直接比较的功能，但这并没有暴露给用户层。

####相关资料:
* [Objects comparison in PHP](http://stackoverflow.com/questions/22528491/objects-comparison-in-php)

---

<i>2014-05-08 update 已实现添加`__compare`魔术方法，整理以后近期写篇博客记录一下</i>