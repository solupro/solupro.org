---
layout: post
title: PHP - 使disable_function支持通配符
description: 通过extension使disable_function支持通配符
tags: [C, PHP, code]

comments: true
share: true
---
很久很久以前，翻`php.ini`的时候看到了一堆“同族”的函数 

>; This directive allows you to disable certain functions for security reasons.
; It receives a comma-delimited list of function names. This directive is
; *NOT* affected by whether Safe Mode is turned On or Off.
; http://php.net/disable-functions
disable_functions = pcntl_alarm,pcntl_fork,pcntl_waitpid,pcntl_wait,pcntl_wifexited,
pcntl_wifstopped,pcntl_wifsignaled,pcntl_wexitstatus,pcntl_wtermsig,pcntl_wstopsig,
pcntl_signal,pcntl_signal_dispatch,pcntl_get_last_error,pcntl_strerror,pcntl_sigprocmask,
pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_exec,pcntl_getpriority,pcntl_setpriority

当时就想要是支持通配符那么直接写成`pcntl_*`这样就简便多了。想法是有了，但是不知道怎么实现好。偶然的机会看到了《[浅谈从PHP内核层面防范PHP WebShell](http://www.80vul.com/webzine_0x05/0x07%20%E6%B5%85%E8%B0%88%E4%BB%8EPHP%E5%86%85%E6%A0%B8%E5%B1%82%E9%9D%A2%E9%98%B2%E8%8C%83PHP%20WebShell.html)》这文章，当中提到`zend_disable_function`这个函数，于是感觉先前的通配符想法可以实现了。

说一下简单的思路吧：在`php.ini`读取配置，遍历函数表，正则匹配函数然后删除掉，注册一个同名函数以便给前端提示。

<!-- more -->

先用C模拟一下实现吧，代码如下

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pcre.h>

#define OVERCCOUNT 30
#define MAX_REGEX_COUNT 50 //最大支持规则数量

char *replace_start(char *src) { //替换通配符*号
    static char buffer[4096];
    char *p, *str;
    char *orig = "*";
    char *rep = "(\\w+)";
    
    str = (char *)malloc(4096);
    p = strstr(src, orig);
    if (p == src) {
        sprintf(str, "%s%s", src, "$");
    } else {
        sprintf(str, "%s%s%s", "^", src, "$");
    }
    if (!(p = strstr(str, orig))) {
        return str;
    }
    strncpy(buffer, str, p-str); // Copy characters from 'str' start to 'orig' st$
    buffer[p-str] = '\0';

    sprintf(buffer+(p-str), "%s%s", rep, p+strlen(orig));
    free(str);
    return buffer;
}

int matchpattern(char *src, char *pattern) {
    pcre *re;
    const char *error;
    int erroffset;
    int ovector[OVERCCOUNT];
    int rc;
    re = pcre_compile(pattern, PCRE_CASELESS|PCRE_DOTALL, &error, &erroffset, NULL);
    if (re == NULL)
        return 0;
    rc = pcre_exec(re, NULL, src, strlen(src), 0, 0, ovector, OVERCCOUNT);
    free(re);
    return rc;
}

int main(int argc, char **argv)
{
    char *function_table[] = \
        {"array_diff", "array_pop", "array_shift", "var_dump", "time", \
         "date", "str_replace", "strstr", "test", "abc_str"};
    char *ini = "array_*, *str test";

    char *s, *p;
    char *delim = ", ";//这里支持,号和空格来分割规则
    char *regex_list[MAX_REGEX_COUNT] = {0};

    int i = 0;
    s = strndup(ini, strlen(ini));
    p = strtok(s, delim);
    if (p) {
        do {
            p = replace_start(p);
            regex_list[i] = strndup(p, strlen(p));
            i++;
        } while ((p = strtok(NULL, delim)));
    }
   
    int match = -1, k;
    char *func, *regex;
    for (i = 0; i < 10; i++) {
        func = function_table[i];
        for (k = 0; k < MAX_REGEX_COUNT; k++) {
            regex = regex_list[k];
            if (!regex) break;
            //printf("regex:%s\n", regex);
            match = matchpattern(func, regex);
            if (match >= 0) {
                printf("function:%s() are disabled!!\n", func);
            }
        }
    }
   
    //free memory
    for(i = 0; i < MAX_REGEX_COUNT; i++) {
        regex = regex_list[i];
        if (regex) {
            free(regex);
            regex_list[i] = NULL;
        }
    }
    free(s);
    s = NULL;
    return 0;    
}
{% endhighlight %}

因为要使用正则，我在这里选择了`pcre`库，于是我们编译的时候要带上`-lpcre`。运行看看我们的效果。
![](//static.oschina.net/uploads/space/2013/0204/115147_uZQT_111529.png)
嗯，好像还不错的样子。接下来就是关键了，怎么改编成PHP扩展

至于怎么快速创建一个PHP扩展的就不介绍了，可以参考《[快速开发一个PHP扩展](http://blog.csdn.net/heiyeshuwu/article/details/3453854)》，我在这里新建了一个叫`solutest`的扩展。接着我们把上面的函数（main函数对应的改一下名字，我这里改为 static void remove_function()）贴到`solutest.c`（文件名对应你创建时候输入的名字）里面，对应的内存操作函数可以换成由php内核提供的e*系列函数，malloc->emalloc, free->efree ...还有一点是用e*系列申请的内存才用`efree`来释放，囧在这里吃过亏。（详细参考《PHP扩展开发与内核应用》- [内存管理](http://www.walu.cc/phpbook/3.1.md)）。然后在 `PHP_MINIT_FUNCTION` 里面调用我们的 `remove_function`，为什么选择 PHP_MINIT_FUNCTION ？或者你可以尝试在 `PHP_RINIT_FUNCTION` 调用 （参考《PHP扩展开发与内核应用》- [PHP启动与终结](http://www.walu.cc/phpbook/1.2.md)）。编译看看效果，别忘了需要`pcre`库的支持，所以要加上 pcre.h 后，然后编辑 Makefile 在EXTRA_LIBS 加上 `-lpcre`。

OK，make && sudo make install，接着编辑`php.ini`加上我们的扩展（我测试环境是nginx + php-fpm,对应php.ini在 /etc/php5/fpm/php.ini，如果不确定你加载的配置文件路径可以查看phpinfo的Loaded Configuration File）

>[solutest]   
extension=solutest.so

`sudo /etc/init.d/php5-fpm restart `  
我们重启fpm看看效果(如果apache环境直接重启apache服务器即可)
![](//static.oschina.net/uploads/space/2013/0204/151750_Nt7k_111529.png)
嗯～跑起来了。

怎么获取系统的函数呢？我们可以参考一下`zend_disable_function`的实现

{% highlight c %}
//file:"Zend/zend_API.c" line:2524
ZEND_API int zend_disable_function(char *function_name, uint function_name_length TSRMLS_DC)
{
	if (zend_hash_del(CG(function_table), function_name, function_name_length+1)==FAILURE) {
		return FAILURE;
	}
	disabled_function[0].fname = function_name;
	return zend_register_functions(NULL, disabled_function, CG(function_table), MODULE_PERSISTENT TSRMLS_CC);
}
{% endhighlight %}

嗯，从函数我们可以知道`CG(function_table)`保存了我们要的函数表，而且它是一个 HashTable 结构，我们可以通过 `zend_hash_del` 删除函数表内某个函数。跟进去 `zend_hash_del`函数看看

{% highlight c %}
//file:"Zend/zend_hash.h" line:154
#define zend_hash_del(ht, arKey, nKeyLength) \
		zend_hash_del_key_or_index(ht, arKey, nKeyLength, 0, HASH_DEL_KEY)
{% endhighlight %}

是一个宏，继续展开深入在 `file:"Zend/zend_hash.c" line:486`，函数有点就不贴了，可以看出是对HashTable的遍历和一些链表删除的操作，还有得到一个重要信息是函数名保存在了`Bucket`的`arKey`。以下是HashTbale的定义 

{% highlight c %}
//file:"Zend/zend_hash.h" line:52
struct _hashtable;

typedef struct bucket {
	ulong h;						/* Used for numeric indexing */
	uint nKeyLength;
	void *pData;
	void *pDataPtr;
	struct bucket *pListNext;
	struct bucket *pListLast;
	struct bucket *pNext;
	struct bucket *pLast;
	const char *arKey;
} Bucket;

typedef struct _hashtable {
	uint nTableSize;
	uint nTableMask;
	uint nNumOfElements;
	ulong nNextFreeElement;
	Bucket *pInternalPointer;	/* Used for element traversal */
	Bucket *pListHead;
	Bucket *pListTail;
	Bucket **arBuckets;
	dtor_func_t pDestructor;
	zend_bool persistent;
	unsigned char nApplyCount;
	zend_bool bApplyProtection;
#if ZEND_DEBUG
	int inconsistent;
#endif
} HashTable;
{% endhighlight %}

详细的解释可以参考《深入理解PHP内核》- [PHP哈希表实现](http://www.php-internal.com/book/?p=chapt03/03-01-02-hashtable-in-php)

好吧，依葫芦画瓢，尝试遍历一下`function_table`。把 `remove_function`函数对应修改为

{% highlight c %}
static void remove_function() {
#ifdef ZEND_SIGNALS
    TSRMLS_FETCH();
#endif

    char *ini = "array_*, *str test";

    char *s, *p;
    char *delim = ", ";//这里支持,号和空格来分割规则
    char *regex_list[MAX_REGEX_COUNT] = {0};

    int i = 0;
    s = estrndup(ini, strlen(ini));
    p = strtok(s, delim);
    if (p) {
        do {
            //p = replace_str(p, "*", "(\\w+)");
            p = replace_start(p);
            regex_list[i] = estrndup(p, strlen(p));
            i++;
        } while ((p = strtok(NULL, delim)));
    }
   
    int match = -1, k;
    char *regex;
    HashTable ht_func, *pht_func;
    Bucket *pBk;
    //拷贝一份CG(function_table)进行操作
    zend_hash_init(&ht_func, zend_hash_num_elements(CG(function_table)), NULL, NULL, 0);
    zend_hash_copy(&ht_func, CG(function_table), NULL, NULL, sizeof(zval*));
    pht_func = &ht_func;

    for (pBk = pht_func->pListHead; pBk != NULL; pBk = pBk->pListNext) {
 		printf("%s()\n", pBk->arKey);
    }

    //free memory
    zend_hash_destroy(&ht_func); //销毁HashTable
    pht_func = NULL;
    for(i = 0; i < MAX_REGEX_COUNT; i++) {
        regex = regex_list[i];
        if (regex) {
            efree(regex);
            regex_list[i] = NULL;
        }
    }
    efree(s);
    s = NULL;   
}
{% endhighlight %}

保存以后又是一轮的  make && sudo make install。sudo /etc/init.d/php5-fpm restart，刷啦啦的一大片，吓坏了吧，保存下来看看有多少。
![](//static.oschina.net/uploads/space/2013/0204/160012_pMwU_111529.png)
应该差不多了吧，后面有...省略号是不是buffer什么的满了所以还没输出完呢？？？

OK，下面是重点了，删除对应的函数。其实我们抄一下`zend_disable_function`就OK了，有同学会问为什么不直接调用`zend_disable_function`，别急，下面我会说道。再次修改我们的`remove_function`函数，这次修改便利的循环体和 `char *ini` 就好

{% highlight c %}
char *ini = "array_p*,"; //使用array族函数测试
{% endhighlight %}

{% highlight c %}
for (pBk = pht_func->pListHead; pBk != NULL; pBk = pBk->pListNext) {
    for (k = 0; k < MAX_REGEX_COUNT; k++) {
        regex = regex_list[k];
        if (!regex) break;
        //regex = "^array_p(\\w+)";
        match = matchpattern(pBk->arKey, regex);
        if (match >= 0) {
            printf("function:%s are disabled!!\n", pBk->arKey);
            //zend_disable_function(func, sizeof(func));
            if (zend_hash_del(CG(function_table), pBk->arKey, strlen(pBk->arKey)+1) == FAILURE) {
                printf("disable %s error\n", pBk->arKey);
            };
            disabled_function[0].fname = pBk->arKey;
            zend_register_functions(NULL, disabled_function, CG(function_table), MODULE_PERSISTENT TSRMLS_CC);
        }
    }
}
{% endhighlight %}

因为把系统的函数删除了，不知请者调用会产生一个php函数不存在的错误，脚本也会停止运行，于是需要注册一个同名的函数回去，而这个函数什么也不做，输出提示就好。那么我们需要在 `remove_function` 函数之前定义函数入口和提示函数

{% highlight c %}
PHP_FUNCTION(print_disabed_info)
{  
    //I don't know why I can't use get_active_function_name in here
    // Maybe "EG"
    zend_error(E_WARNING, "*** function has been disabled! (°Д°≡°д°)ｴｯ!?"); //get_active_function_name(TSRMLS_C)
}

static zend_function_entry disabled_function[] = {
    PHP_FALIAS(display_disabled_function, print_disabed_info, NULL)
    PHP_FE_END
};
{% endhighlight %}

估计有同学吐槽为什么用***代替了显示的函数名，这就是为什么我不调用`zend_disable_function`的原因。当时卡在这里很久，一直段错误，后来无意中注释了 get_active_function_name(TSRMLS_C) 就跑起来了╯-__-)╯ ╩╩，求告知(2014-05-12 update 问题已解决，下面会给出代码地址)。。和上面一个编译重启服务器什么的，然后看效果，因为我们配置写的是`array_p*`，所以一下函数被禁用了。（测试完以后记得关闭输出）
![](//static.oschina.net/uploads/space/2013/0204/162346_lRR8_111529.png)

然后随便写个脚本，调用一下array_pop函数什么的，然后执行之。
![](//static.oschina.net/uploads/space/2013/0204/162548_I77d_111529.png)

It's work!! :)

呼，不知不觉写了这么长了，也懒得分两篇了。接下来把读取`php.ini`配置代码写上就完成了。其实这部分工作在扩展自动生成的代码已经有了，只要稍微加工一下就好。

{% highlight c %}
/* 
  	Declare any global variables you may need between the BEGIN
	and END macros here:     
*/
ZEND_BEGIN_MODULE_GLOBALS(solutest)
	char *disable_functions;
ZEND_END_MODULE_GLOBALS(solutest)
{% endhighlight %}

`php_solutest.h` 大概47行左右的样子，去掉注释加入我们的`disable_functions`变量

{% highlight c %}
/* If you declare any globals in php_solutest.h uncomment this:*/
ZEND_DECLARE_MODULE_GLOBALS(solutest)
{% endhighlight %}

`solutest.c`30行左右，去掉注释

{% highlight c %}
/* Remove comments and fill if you need to have entries in php.ini*/
PHP_INI_BEGIN()
    STD_PHP_INI_ENTRY("solutest.disable_functions", "", PHP_INI_ALL, OnUpdateString, disable_functions, zend_solutest_globals, solutest_globals)
PHP_INI_END()
{% endhighlight %}

`solutest.c` 71行左右，去掉注释，修改为我们的变量

{% highlight c %}
/* If you have INI entries, uncomment these lines */
REGISTER_INI_ENTRIES();
{% endhighlight %}

`PHP_MINIT_FUNCTION`函数里面，去掉注释

{% highlight c %}
/* Remove comments if you have entries in php.ini */
DISPLAY_INI_ENTRIES();
{% endhighlight %}

`PHP_MINFO_FUNCTION` 函数里面，去掉注释

然后编辑你的`php.ini`文件，加入配置

>[solutest]  
extension=solutest.so  
solutest.disable_functions = array_p*,

编译重启服务器，然后浏览`phpinfo`会发现我们的配置已经被读取了。
![](//static.oschina.net/uploads/space/2013/0204/165144_uZ1e_111529.png)

最后把我们的配置利用上，可以通过`SOLUEXT_G(disable_functions)`宏来访问，对应修改 `remove_function` 函数。去掉 `char *ini` 因为已经不需要了，配置从`php.ini` 读取，然后修改 s 

{% highlight c %}
s = estrndup(SOLUTEST_G(disable_functions), strlen(SOLUTEST_G(disable_functions)));
{% endhighlight %}

OK，保存编译重启服务器测试。
![](//static.oschina.net/uploads/space/2013/0204/170137_r1sY_111529.png)

:)预期的效果达到了。打完收工。

PS:此扩展是本人YY的产物，没有经过严格测试，请勿在生产机上使用。
  
Github: [https://github.com/solupro/rdfunc](https://github.com/solupro/rdfunc)

###参考资料：
---
* 《[浅谈从PHP内核层面防范PHP WebShell](http://www.80vul.com/webzine_0x05/0x07%20%E6%B5%85%E8%B0%88%E4%BB%8EPHP%E5%86%85%E6%A0%B8%E5%B1%82%E9%9D%A2%E9%98%B2%E8%8C%83PHP%20WebShell.html)》
* 《[PHP扩展开发及内核应用](http://www.walu.cc/phpbook/index.md)》
* 《[鸟哥博客](http://www.laruence.com/)》
* 《[快速开发一个PHP扩展](http://blog.csdn.net/heiyeshuwu/article/details/3453854)》
* 《[深入理解PHP内核](http://www.php-internal.com/)》
