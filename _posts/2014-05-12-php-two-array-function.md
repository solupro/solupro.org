---
layout: post
title: PHP 实用的两个array函数
description: 感觉比较实用的array函数，熟悉HashTable操作
tags: [C, PHP, code]

comments: true
share: true
---
感觉比较实用的两个array函数，为了熟悉一下`HashTable`的操作就写成了extension的形式

### array_insert
---
--对应index插入值得的值  
array array_insert ( array $input , int $index , mixed $value )

示例:

{% highlight php %}
<?php
$arr = ['a' => 1, 'b' => 2, 'c' => 3, 4];
var_dump(array_insert($arr, -1, ['foo', 'bar']));
/* output */
array (size=5)
  'a' => int 1
  'b' => int 2
  'c' => int 3
  0 => 
    array (size=2)
      0 => string 'foo' (length=3)
      1 => string 'bar' (length=3)
  1 => int 4
{% endhighlight %}

<!-- more -->

{%  highlight c %}
PHP_FUNCTION(array_insert)
{
    zval     *input,        /* Input array */
            **value,        
            **entry;        /* An array entry */
    long     position;         
    int      num_in,        /* Number of elements in the input array */
             pos;           /* Current position in the array */
    char *string_key;
    uint string_key_len;
    ulong num_key;
    HashPosition hpos;
 
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "alZ", &input, &position, &value) == FAILURE) {
        return;
    }
 
    /* Get number of entries in the input hash */
    num_in = zend_hash_num_elements(Z_ARRVAL_P(input));
 
    if (position < 0 && (position = (num_in + position)) < 0) {
            position = 0;
    } else if ( position > num_in) {
        position = num_in;        
    }
 
    /* Initialize returned array */
    array_init_size(return_value, num_in + 1);
    
    pos = 0;
    zend_hash_internal_pointer_reset_ex(Z_ARRVAL_P(input), &hpos);
    while (zend_hash_get_current_data_ex(Z_ARRVAL_P(input), (void **)&entry, &hpos) == SUCCESS) {
        
        if (pos == position) {
            zval_add_ref(value);
            zend_hash_next_index_insert(Z_ARRVAL_P(return_value), value, sizeof(zval *), NULL);
        }
        
        zval_add_ref(entry);
        switch (zend_hash_get_current_key_ex(Z_ARRVAL_P(input), &string_key, &string_key_len, &num_key, 0, &hpos)) {
            case HASH_KEY_IS_STRING:
                zend_hash_update(Z_ARRVAL_P(return_value), string_key, string_key_len, entry, sizeof(zval *), NULL);
                break;
 
            case HASH_KEY_IS_LONG:
                zend_hash_next_index_insert(Z_ARRVAL_P(return_value), entry, sizeof(zval *), NULL);
                break;
        }
        pos++;
        zend_hash_move_forward_ex(Z_ARRVAL_P(input), &hpos);
    }
    
    if (position == num_in) {
        zval_add_ref(value);
        zend_hash_next_index_insert(Z_ARRVAL_P(return_value), value, sizeof(zval *), NULL);
    }
}
{% endhighlight %}


### array_column_recursive
---
--返回数组中指定的一列,递归多维数组   
array array_column_recursive ( array $input , mixed $column_key [, mixed $index_key ] )

其实这个函数是`PHP5.5`添加的，嗯，我的`PHP5.4`还没有，而且我稍微修改成递归多维数组。官方文档:[http://cn2.php.net/manual/zh/function.array-column.php](http://cn2.php.net/manual/zh/function.array-column.php)

示例:

{% highlight php %}
<?php
$arr = [
    id => 9527,
    [
        'id' => 2135,
        'first_name' => 'John',
        'last_name' => 'Doe',
    ],
    [
        'id' => 3245,
        'first_name' => 'Sally',
        'last_name' => 'Smith',
        [
            'id' => [
                'id' => 4321,
                'first_name' => 'Foo',
                'last_name' => 'Bar',
            ],
        ]
    ],
    [
        'id' => 5342,
        'first_name' => 'Jane',
        'last_name' => 'Jones',
    ],
    [
        'id' => 5623,
        'first_name' => 'Peter',
        'last_name' => 'Doe',
        [
            'id' => 1234,
            'first_name' => 'Solu',
            'last_name' => 'Scarlet',
        ],
    ],
    'first_name' => 'Flandre',
    'last_name' => 'Scarlet',
];

$arr = array_column_recursive($arr, 'first_name', 'id');
var_dump($arr);

/* output */
array (size=7)
  2135 => string 'John' (length=4)
  4321 => string 'Foo' (length=3)
  3245 => string 'Sally' (length=5)
  5342 => string 'Jane' (length=4)
  1234 => string 'Solu' (length=4)
  5623 => string 'Peter' (length=5)
  9527 => string 'Flandre' (length=7)
  
{% endhighlight %}

{%  highlight c %}
/* array_column_param_helper
 * Specialized conversion rules for array_column() function
 */
static inline
zend_bool array_column_param_helper(zval **param,
                                    const char *name TSRMLS_DC) {
    switch (Z_TYPE_PP(param)) {
        case IS_DOUBLE:
            convert_to_long_ex(param);
            /* fallthrough */
        case IS_LONG:
            return 1;
            
        case IS_OBJECT:
            convert_to_string_ex(param);
            /* fallthrough */
        case IS_STRING:
            return 1;
            
        default:
            php_error_docref(NULL TSRMLS_CC, E_WARNING, "The %s key should be either a string or an integer", name);
            return 0;
    }
}

static void do_column_recursive(zval *return_value, HashTable *arr_hash, zval **zcolumn, zval ** zkey)
{
    zval **data;
    HashPosition pointer;
    char *string_key;
    uint string_key_len;
    ulong num_key;
    zval **zcolval = NULL, **zkeyval = NULL;
    
    for (zend_hash_internal_pointer_reset_ex(arr_hash, &pointer);
         zend_hash_get_current_data_ex(arr_hash, (void**)&data, &pointer) == SUCCESS;
         zend_hash_move_forward_ex(arr_hash, &pointer)) {      
        
        if (Z_TYPE_PP(data) == IS_ARRAY) {
            do_column_recursive(return_value, Z_ARRVAL_PP(data), zcolumn, zkey);
            continue;
        }
        
        switch (zend_hash_get_current_key_ex(arr_hash, &string_key, &string_key_len, &num_key, 0, &pointer)) {
            case HASH_KEY_IS_STRING:
                if ((Z_TYPE_PP(zcolumn) == IS_STRING) && !strcmp(string_key, Z_STRVAL_PP(zcolumn))) {
                    zcolval = data;
                }
                if ((Z_TYPE_PP(zkey) == IS_STRING) && !strcmp(string_key, Z_STRVAL_PP(zkey))) {
                    zkeyval = data;
                }
                break;
            case HASH_KEY_IS_LONG:
                if ((Z_TYPE_PP(zcolumn) == IS_LONG) && num_key == Z_LVAL_PP(zcolumn)) {
                    zcolval = data;
                }
                if ((Z_TYPE_PP(zkey) == IS_LONG) && num_key == Z_LVAL_PP(zkey)) {
                    zkeyval = data;
                }
                break;
        }
        
    }
    
    if (zcolval) {
        Z_ADDREF_PP(zcolval);
        if (zkeyval && Z_TYPE_PP(zkeyval) == IS_STRING) {
            add_assoc_zval(return_value, Z_STRVAL_PP(zkeyval), *zcolval);
        } else if (zkeyval && Z_TYPE_PP(zkeyval) == IS_LONG) {
            add_index_zval(return_value, Z_LVAL_PP(zkeyval), *zcolval);
        } else if (zkeyval && Z_TYPE_PP(zkeyval) == IS_OBJECT) {
            SEPARATE_ZVAL(zkeyval);
            convert_to_string(*zkeyval);
            add_assoc_zval(return_value, Z_STRVAL_PP(zkeyval), *zcolval);
        } else {
            add_next_index_zval(return_value, *zcolval);
        }
    }
}

/* proto array array_column_recursive(array input, mixed column_key[,mixed index_key])
 Return the values from a single column in the input array, identified by the
 value_key and optionally indexed by the index_key */
PHP_FUNCTION(array_column_recursive)
{
    zval **zcolumn = NULL, **zkey = NULL;
    HashTable *arr_hash;
    
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "hZ!|Z!", &arr_hash, &zcolumn, &zkey) == FAILURE) {
        return;
    }
    
    if ((zcolumn && !array_column_param_helper(zcolumn, "column" TSRMLS_CC)) ||
        (zkey && !array_column_param_helper(zkey, "index" TSRMLS_CC))) {
        RETURN_FALSE;
    }
    
    array_init(return_value);
    do_column_recursive(return_value, arr_hash, zcolumn, zkey);
}
{% endhighlight %}

---
<s>开始直接引用gist的，发现这代码配色和gist搭配无法忍受还是算了</s>