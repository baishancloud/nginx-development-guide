NGINX开发指南
===========

* [译者序](#译者序)
* [简介](#简介)
    * [代码结构](#代码结构)
    * [头文件](#头文件)
    * [整数](#整数)
    * [常用返回值](#常用返回值)
    * [错误处理](#错误处理)
* [字符串](#字符串)
    * [概述](#概述)
    * [格式化](#格式化)
    * [数值转换](#数值转换)
    * [正则表达式](#正则表达式)
* [时间](#时间)
* [容器](#容器)
    * [数组](#数组)
    * [列表](#列表)
    * [队列](#队列)
    * [红黑树](#红黑树)
    * [哈希](#哈希)
* [内存管理](#内存管理)
    * [堆](#堆)
    * [内存池](#内存池)
    * [共享内存](#共享内存)
* [日志](#日志)
* [周期](#周期)
* [缓冲](#缓冲)
* [网络](#网络)
    * [连接](#连接)
* [事件](#事件)
    * [事件](#事件)
    * [I/O事件](#I/O事件)
    * [定时器事件](#定时器事件)
    * [延迟事件](#延迟事件)
    * [遍历事件](#遍历事件)
* [进程](#进程)
* [线程](#线程)
* [模块](#模块)
    * [添加新模块](#添加新模块)
    * [核心模块](#核心模块)
    * [配置指令](#配置指令)
* [HTTP](#HTTP)
    * [连接](#连接)
    * [请求](#请求)
    * [配置](#配置)
    * [阶段](#阶段)
    * [变量](#变量)
    * [复杂值](#复杂值)
    * [请求重定向](#请求重定向)
    * [子请求](#子请求)
    * [请求结束](#请求结束)
    * [请求体](#请求体)
    * [响应](#响应)
    * [响应体](#响应体)
    * [响应体过滤](#响应体过滤)
    * [构建过滤模块](#构建过滤模块)
    * [缓冲复用](#缓冲复用)
    * [负载均衡](#负载均衡)
    
译者序
=====

本文档是nginx官方文档“Developer Guide”（[https://nginx.org/en/docs/dev/development_guide.html](https://nginx.org/en/docs/dev/development_guide.html)）的中文版本，由白山云（[http://www.baishancloud.com](http://www.baishancloud.com/zh/)）NGINX开发团队负责翻译。官方文档是HTML页面发布的，我们翻译的时候转成了Markdown，以方便编辑。同时也一并保留了英文的Markdown版本：[https://github.com/baishancloud/nginx-development-guide/blob/master/en.md](https://github.com/baishancloud/nginx-development-guide/blob/master/en.md)。希望此中文版文档能为广大的nginx以及开源爱好者提供入门指导，开发出优秀的nginx模块，回馈社区。本文的官方版本并没有全部完成，依然处于活跃更新的状态，中文版本会持续保持跟踪并持续更新。

简介
===

代码结构
-------
* auto — 编译脚本
* src
    * core — 基础数据结构和函数 — 字符串，数组，日志，内存池等
    * event — 事件机制核心模块
        * modules — 具体事件机制模块：epoll，kqueue，select等
    * http — HTTP核心模块和公共代码
        * modules — 其他HTTP模块
        * v2 — HTTP/2模块
    * mail — 邮件协议模块
    * os — 平台相关代码
        * unix
        * win32
    * stream — 流模块

头文件
-----
每个nginx文件都应该在开头包含如下两个头文件：

```
#include <ngx_config.h>
#include <ngx_core.h>
```

除此之外，HTTP相关的代码还要包含：

```
#include <ngx_http.h>
```

邮件模块的代码应该包含：

```
#include <ngx_mail.h>
```

Stream模块的代码应该包含：

```
#include <ngx_stream.h>
```

整数
----
一般情况下，nginx代码使用如下两个整数类型：ngx_int_t和ngx_uint_t，分别用typedef定义成了intptr_t和uintptr_t。

常用返回值
--------
nginx中的大多数函数使用如下类型的返回值：

* NGX_OK — 处理成功
* NGX_ERROR — 处理失败
* NGX_AGAIN — 处理未完成，函数需要被再次调用
* NGX_DECLINED — 处理被拒绝，例如相关功能在配置文件中被关闭。不要将此当成错误。
* NGX_BUSY — 资源不可用
* NGX_DONE — 处理完成或者在他处继续处理。也可以作为处理成功使用。
* NGX_ABORT — 函数终止。也可以作为处理出错的返回值。

错误处理
-------
为了获取最近一次系统错误码，nginx提供了ngx_errno宏。该宏被映射到了POSIX平台的errno变量上，而在Windows平台中，则变为对GetLastError()的函数调用。为了获取最近一次socket错误码，nginx提供了ngx_socket_errno宏。同样，在POSIX平台上该宏被映射为errno变量，而在Windows环境中则是对WSAGetLastError()进行调用。考虑到对性能的影响，ngx_errno和ngx_socket_errno不应该被连续访问。如果有连续、频繁访问的需要，则应该将错误码的值存储到类型为ngx_err_t的本地变量中，然后使用本地变量进行访问。如果需要设置错误码，可以使用ngx_set_errno(errno)和ngx_set_socket_errno(errno)这两个宏。

ngx_errno和ngx_socket_errno变量可以在调用日志相关函数ngx_log_error()和ngx_log_debugX()的时候使用，这样具体的错误文本就会被添加到日志输出中。

一个使用ngx_errno的例子：

```
void
ngx_my_kill(ngx_pid_t pid, ngx_log_t *log, int signo)
{
    ngx_err_t  err;

    if (kill(pid, signo) == -1) {
        err = ngx_errno;

        ngx_log_error(NGX_LOG_ALERT, log, err, "kill(%P, %d) failed", pid, signo);

        if (err == NGX_ESRCH) {
            return 2;
        }

        return 1;
    }

    return 0;
}
```

字符串
=====

概述
----
nginx使用无符号的char类型指针来表示C字符串：u_char *。

nginx字符串类型ngx_str_t的定义如下所示：

```
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

结构体成员len存放字符串的长度，成员data指向字符串本身数据。在ngx_str_t中存放的字符串，对于超出len长度的部分可以是NULL结尾（'\0'——译者注），也可以不是。在大多数情况是不以NULL结尾的。然而，在nginx的某些代码中（例如解析配置的时候），ngx_str_t中的字符串是以NULL结尾的吗，这种情况会使得字符串比较变得更加简单，也使得使用系统调用的时候更加容易。

nginx提供了一系列关于字符串处理的函数。它们在src/core/ngx_string.h文件中定义。其中的一部分就是对C库中字符串函数的封装：

* ngx_strcmp()
* ngx_strncmp()
* ngx_strstr()
* ngx_strlen()
* ngx_strchr()
* ngx_memcmp()
* ngx_memset()
* ngx_memcpy()
* ngx_memmove()

还有一些nginx特有的字符串函数：

* ngx_memzero() 内存清0
* ngx_cpymem() 和ngx_memcpy()行为类似，不同的是该函数返回的是copy后的最终目的地址，这在需要连续拼接多个字符串的场景下很方便。
* ngx_movemem() 和ngx_memmove()的行为类似，不同的是该函数返回的是move后的最终目的地址。
* ngx_strlchr() 在字符串中查找一个特定字符，字符串由两个指针界定。

最后是一些大小写转换和字符串比较的函数：

* ngx_tolower()
* ngx_toupper()
* ngx_strlow()
* ngx_strcasecmp()
* ngx_strncasecmp()

格式化
-----
nginx提供了一些格式化字符串的函数。以下这些函数支持nginx特有的类型：

* ngx_sprintf(buf, fmt, ...)
* ngx_snprintf(buf, max, fmt, ...)
* ngx_slpintf(buf, last, fmt, ...)
* ngx_vslprint(buf, last, fmt, args)
* ngx_vsnprint(buf, max, fmt, args)

这些函数支持的全部格式化选项定义在src/core/ngx_string.c文件中，以下是其中的一部分：

```
%O — off_t
%T — time_t
%z — size_t
%i — ngx_int_t
%p — void *
%V — ngx_str_t *
%s — u_char * (null-terminated)
%*s — size_t + u_char *
```

'u'修饰符将类型指明为无符号，'X'和'x'则将输出转换为16禁止。

例如：

```
u_char     buf[NGX_INT_T_LEN];
size_t     len;
ngx_int_t  n;

/* set n here */

len = ngx_sprintf(buf, "%ui", n) — buf;
```


数值转换
-------
nginx实现了若干用于数值转换的函数：

* ngx_atoi(line, n) — 将一个指定长度的字符串转换为一个正整数，类型为ngx_int_t。出错返回NGX_ERROR。
* ngx_atosz(line, n) — 同上，转换类型为ssize_t
* ngx_atoof(line, n) — 同上，转换类型为off_t
* ngx_atotm(line, n) — 同上，转换类型为time_t
* ngx_atofp(line, n, point) — 将一个固定长度的定点小数字符串转换为ngx_int_t类型的正整数。转换结果会左移point指定的10进制位数。字符串中的定点小数不能含有多过point参数指定的小数位。出错返回NGX_ERROR。举例：ngx_atofp("10.5", 4, 2) 返回1050
* ngx_hextoi(line, n) — 将表示16进制正整数的字符串转换为ngx_int_t类型的整数。出错返回NGX_ERROR。

正则表达式
--------
nginx中的正则表达式接口是对PCRE库的封装。相关的头文件是src/core/ngx_regex.h。

要使用正则表达式进行字符串匹配，首先需要对正则表达式进行编译，这通常是在配置解析阶段处理的。需要注意的是，因为PCRE的支持是可选的，因此所有使用正则相关接口的代码都需要用NGX_PCRE括起来：

```
#if (NGX_PCRE)
ngx_regex_t          *re;
ngx_regex_compile_t   rc;

u_char                errstr[NGX_MAX_CONF_ERRSTR];

ngx_str_t  value = ngx_string("message (\\d\\d\\d).*Codeword is '(?<cw>\\w+)'");

ngx_memzero(&rc, sizeof(ngx_regex_compile_t));

rc.pattern = value;
rc.pool = cf->pool;
rc.err.len = NGX_MAX_CONF_ERRSTR;
rc.err.data = errstr;
/* rc.options are passed as is to pcre_compile() */

if (ngx_regex_compile(&rc) != NGX_OK) {
    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "%V", &rc.err);
    return NGX_CONF_ERROR;
}

re = rc.regex;
#endif
```

编译成功之后，结构体ngx_regex_compile_t的captures和named_captures成员分别会被填上正则表达式中全部以及命名捕获的数量。

然后，编译过的正则表达式就可以用来进行字符串匹配：

```
ngx_int_t  n;
int        captures[(1 + rc.captures) * 3];

ngx_str_t input = ngx_string("This is message 123. Codeword is 'foobar'.");

n = ngx_regex_exec(re, &input, captures, (1 + rc.captures) * 3);
if (n >= 0) {
    /* string matches expression */

} else if (n == NGX_REGEX_NO_MATCHED) {
    /* no match was found */

} else {
    /* some error */
    ngx_log_error(NGX_LOG_ALERT, log, 0, ngx_regex_exec_n " failed: %i", n);
}
```

ngx_regex_exec()的参数有：编译了的正则表达式re，待匹配的字符串s，可选的用于存放发现的捕获和其大小的整数数组。捕获数组的大小必须是3的倍数，这是PCRE库的API要求的。在上面例子中，该数组的大小是通过总捕获数加上字符串自身来计算得出的。

现在，如果成功匹配，则可以对捕获进行访问：

```
u_char     *p;
size_t      size;
ngx_str_t   name, value;

/* all captures */
for (i = 0; i < n * 2; i += 2) {
    value.data = input.data + captures[i];
    value.len = captures[i + 1] — captures[i];
}

/* accessing named captures */

size = rc.name_size;
p = rc.names;

for (i = 0; i < rc.named_captures; i++, p += size) {

    /* capture name */
    name.data = &p[2];
    name.len = ngx_strlen(name.data);

    n = 2 * ((p[0] << 8) + p[1]);

    /* captured value */
    value.data = &input.data[captures[n]];
    value.len = captures[n + 1] — captures[n];
}
```

ngx_regex_exec_array()函数接受ngx_regex_elt_t元素的数组（其实就是多个编译好的正则表达式以及对应的名字），一个待匹配字符串以及一个log。该函数会对待匹配字符串逐一应用数组中的正则表达式，直到匹配成功或者无一匹配。存在成功的匹配则返回NGX_OK，否则返回NGX_DECLINED，出错返回NGX_ERROR。

时间
====

结构体 ngx_time_t 将GMT格式的时间表示分割成秒和毫秒：

```
typedef struct {
    time_t      sec;
    ngx_uint_t  msec;
    ngx_int_t   gmtoff;
} ngx_time_t;
```

ngx_tm_t 是 struct tm 的一个别名，用在 UNIX 平台和Windows上的SYSTEMTIME。

为了获取当前时间，通常只需要访问一个可用的全局变量，表示所需格式的缓存时间值。ngx_current_msec 变量保存着自Epoch以来的毫秒数，并截成ngx_msec_t。

以下是可用的字符串表示：

* ngx_cached_err_log_time — 用在 error log: "1970/09/28 12:00:00"
* ngx_cached_http_log_time — 用在 HTTP access log: "28/Sep/1970:12:00:00 +0600"
* ngx_cached_syslog_time — 用在 syslog: "Sep 28 12:00:00"
* ngx_cached_http_time — 用在 HTTP headers: "Mon, 28 Sep 1970 06:00:00 GMT"
* ngx_cached_http_log_iso8601 — ISO 8601 标准格式: "1970-09-28T12:00:00+06:00"

宏 ngx_time() 和 ngx_timeofday() 返回当前时间的秒，是访问缓存时间值的首选方式。

为了明确获取时间，可以使用ngx_gettimeofday()，它会更新参数（指向struct timeval）。当nginx从系统调用回到事件循环体时，时间总是会更新。如果想立即更新时间，调用 ngx_time_update() 或 ngx_time_sigsafe_up date() （如果在信号处理上下文需要用到）。

以下函数将 time_t 转换成可分解的时间表示形式，对于libc前缀的那些，可以使用 ngx_tm_t 或者 struct tm。

* ngx_gmtime(), ngx_libc_gmtime() — 结果时间是 UTC
* ngx_localtime(), ngx_libc_localtime() — 结果时间是相对时区

ngx_http_time(buf, time) 返回用于适合 HTTP headers（比如 "Mon, 28 Sep 1970 06:00:00 GMT"）的字符串表示。另一种可能转变通过 ngx_http_cookie_time(buf, time) 提供，用于生成适合HTTP cookies ("Thu, 3 1-Dec-37 23:55:55 GMT") 的格式。

容器
====

数组
----
表示nginx数组（array）的结构体ngx_array_t定义如下：

```
typedef struct {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
} ngx_array_t;
```

数组的元素可以通过elts成员获取。元素的个数存放在nelts成员里。size成员记录单个元素的大小，size成员是在数组初始化的时候设置的。

数组可以使用调用ngx_array_create(pool, n, size)来创建，其所需内存在提供的pool中。一个已经分配过内存的数组对象，可以调用ngx_array_init(array, pool, n, size)进行初始化。


```
ngx_array_t  *a, b;

/* create an array of strings with preallocated memory for 10 elements */
a = ngx_array_create(pool, 10, sizeof(ngx_str_t));

/* initialize string array for 10 elements */
ngx_array_init(&b, pool, 10, sizeof(ngx_str_t));
```

使用下面的函数向数组添加元素：

* ngx_array_push(a) 向数组末尾添加一个元素并返回其指针
* ngx_array_push_n(a, n) 向数组末尾添加n个元素并返回指向其中第一个元素的指针

如果现有内存无法满足新元素的需要，数组会分配新的内存并将现有元素复制过去。新分配的内存一般是原有内存的2倍大。

```
s = ngx_array_push(a);
ss = ngx_array_push_n(&b, 3);
```

列表
----
nginx中的列表（List）由一系列的数组组成，并为可能插入大量item进行了优化。列表类型定义如下：

```
typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
```

实际的item存放在列表部件结构中，定义如下：

```
typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
```

使用之前，列表必须通过ngx_list_init(list, pool, n, size)初始化，或者通过ngx_list_create(pool, n, size)创建。两个方式都需要指定单一条目的大小以及每个列表部件中item的数量。ngx_list_push(list)函数用来向列表添加一个item。遍历item是通过直接访问列表成员实现的，参考以下示例：

```
ngx_str_t        *v;
ngx_uint_t        i;
ngx_list_t       *list;
ngx_list_part_t  *part;

list = ngx_list_create(pool, 100, sizeof(ngx_str_t));
if (list == NULL) { /* error */ }

/* add items to the list */

v = ngx_list_push(list);
if (v == NULL) { /* error */ }
ngx_str_set(v, "foo");

v = ngx_list_push(list);
if (v == NULL) { /* error */ }
ngx_str_set(v, "bar");

/* iterate over the list */

part = &list->part;
v = part->elts;

for (i = 0; /* void */; i++) {

    if (i >= part->nelts) {
        if (part->next == NULL) {
            break;
        }

        part = part->next;
        v = part->elts;
        i = 0;
    }

    ngx_do_smth(&v[i]);
}
```

nginx中列表的主要用途是处理HTTP中输入和输出的头部。

列表不支持删除item。然而，如果需要的话，可以将item标识成missing而不是真正的删除他们。例如，HTTP的输出头部——以ngx_table_elt_t对象存储——可以通过将ngx_table_elt_t结构的hash成员设置成0来将其标识为missing。这样一来，该HTTP头部就不会被遍历到。

队列
----

nginx里的队列是一个双向链表，每个节点定义如下：

```
typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

头部队列节点没有连接任何数据。使用之前，列表头部要先调用 ngx_queue_init(q) 以初始化。队列支持如下操作：

* ngx_queue_insert_head(h, x), ngx_queue_insert_tail(h, x) — 插入新节点
* ngx_queue_remove(x) — 删除队列节点
* ngx_queue_split(h, q, n) — 从a节点切割，队列尾部起将变成新的独立的队列
* ngx_queue_add(h, n) — 将队列n加到队列h
* ngx_queue_head(h), ngx_queue_last(h) — 返回首或尾队列节点
* ngx_queue_sentinel(h) - 返回队列哨兵用来结束迭代
* ngx_queue_data(q, type, link) — 返回指向queue的data字段的起始地址，根据它的queue字段的偏移量

例子：

```
typedef struct {
    ngx_str_t    value;
    ngx_queue_t  queue;
} ngx_foo_t;

ngx_foo_t    *f;
ngx_queue_t   values;

ngx_queue_init(&values);

f = ngx_palloc(pool, sizeof(ngx_foo_t));
if (f == NULL) { /* error */ }
ngx_str_set(&f->value, "foo");

ngx_queue_insert_tail(&values, f);

/* insert more nodes here */

for (q = ngx_queue_head(&values);
     q != ngx_queue_sentinel(&values);
     q = ngx_queue_next(q))
{
    f = ngx_queue_data(q, ngx_foo_t, queue);

    ngx_do_smth(&f->value);
}
```

红黑树
------

头文件 src/core/ngx_rbtree.h 提供了访问红黑树的定义。

```
typedef struct {
    ngx_rbtree_t       rbtree;
    ngx_rbtree_node_t  sentinel;

    /* custom per-tree data here */
} my_tree_t;

typedef struct {
    ngx_rbtree_node_t  rbnode;

    /* custom per-node data */
    foo_t              val;
} my_node_t;
```

为了处理整个树，需要两个节点：root 和 sentinel。通常他们被添加到某些自定义的结构中，这样就能将数据组织到树中，其中包含指向数据的边接。

初始化树：

```
my_tree_t  root;

ngx_rbtree_init(&root.rbtree, &root.sentinel, insert_value_function);
```

The insert_value_function is a function that is responsible for traversing the tree and inserting new values into correct place. For example, the ngx_str_rbtree_insert_value functions is designed to deal with ngx_str_t type.

```
void ngx_str_rbtree_insert_value(ngx_rbtree_node_t *temp,
                                 ngx_rbtree_node_t *node,
                                 ngx_rbtree_node_t *sentinel)
```

第一个参数是树中插入的节点，第二个是新创建的用来添加的节点，最后一个是树的sentinel。

遍历非常简单明了，用下面的轮询函数模式作为演示。

```
my_node_t *
my_rbtree_lookup(ngx_rbtree_t *rbtree, foo_t *val, uint32_t hash)
{
    ngx_int_t           rc;
    my_node_t          *n;
    ngx_rbtree_node_t  *node, *sentinel;

    node = rbtree->root;
    sentinel = rbtree->sentinel;

    while (node != sentinel) {

        n = (my_node_t *) node;

        if (hash != node->key) {
            node = (hash < node->key) ? node->left : node->right;
            continue;
        }

        rc = compare(val, node->val);

        if (rc < 0) {
            node = node->left;
            continue;
        }

        if (rc > 0) {
            node = node->right;
            continue;
        }

        return n;
    }

    return NULL;
}
```

compare() 是一个返回较小，相等或较大的经典函数。为了更快的查找，并且避免比较太大的对象，整型的hash字段就派上用场了。

为了添加节点到树，需要分配新节点，初始化它，然后调用 ngx_rbtree_insert()：

```
    my_node_t          *my_node;
    ngx_rbtree_node_t  *node;

    my_node = ngx_palloc(...);
    init_custom_data(&my_node->val);

    node = &my_node->rbnode;
    node->key = create_key(my_node->val);

    ngx_rbtree_insert(&root->rbtree, node);
```

to remove a node:

```
ngx_rbtree_delete(&root->rbtree, node);
```

哈希
----

唏希表定义在 src/core/ngx_hash.h，支持精确和通配符匹配。后者需要额外的处理，放在下面的章节专门描述。

初始化哈希时，我们需要提前知道元素的个数，以便nginx能更好的优化哈希表。max_size 和 bucket_size 这两参数需要配置。细节详见官方提供的文档。通常这两参数会做成用户可配置的。哈希初始化的设置放在ngx_hash_init_t类型的存储中。而哈希表本身的类型是 ngx_hash_t。

```
ngx_hash_t       foo_hash;
ngx_hash_init_t  hash;

hash.hash = &foo_hash;
hash.key = ngx_hash_key;
hash.max_size = 512;
hash.bucket_size = ngx_align(64, ngx_cacheline_size);
hash.name = "foo_hash";
hash.pool = cf->pool;
hash.temp_pool = cf->temp_pool;
```

key是一个指向能根据字符串创建整型的函数的指针。nginx提供了两个通用的函数：ngx_hash_key(data, len) 和 ngx_hash_key_lc(data, len)。后者将字符串转为小写，这需要这个字符串是可写的。如果不想这样，NGX_HASH_READONLY_KEY 标记可以传给这个函数，然后初始化数组键（见下文）。

哈希keys保存在ngx_hash_keys_arrays_t里，然后通过 ngx_hash_keys_array_init(arr, type) 初始化。

```
ngx_hash_keys_arrays_t  foo_keys;

foo_keys.pool = cf->pool;
foo_keys.temp_pool = cf->temp_pool;

ngx_hash_keys_array_init(&foo_keys, NGX_HASH_SMALL);
```

第二个参数可以是NGX_HASH_SMALL或者NGX_HASH_LARGE，用于控制哈希表的预分配。如果你想hash包含更多的无素，请用NGX_HASH_LARGE。

ngx_hash_add_key(keys_array, key, value, flags) 函数用于将key添加到hash keys array：

```
ngx_str_t k1 = ngx_string("key1");
ngx_str_t k2 = ngx_string("key2");

ngx_hash_add_key(&foo_keys, &k1, &my_data_ptr_1, NGX_HASH_READONLY_KEY);
ngx_hash_add_key(&foo_keys, &k2, &my_data_ptr_2, NGX_HASH_READONLY_KEY);
```

现在就可能通过调用 ngx_hash_init(hinit, key_names, nelts) 来完成hash表的创建：

```
ngx_hash_init(&hash, foo_keys.keys.elts, foo_keys.keys.nelts);
```

这样是有可能错误的，如果max_size或者bucket_size不足够大的话。当hash创建了之后， ngx_hash_find(hash, key, name, len) 函数可用来查找无素：

```
my_data_t   *data;
ngx_uint_t   key;

key = ngx_hash_key(k1.data, k1.len);

data = ngx_hash_find(&foo_hash, key, k1.data, k1.len);
if (data == NULL) {
    /* key not found */
}
```

通配符匹配
----------

为了创建能运行通配符的hash，需要用 ngx_hash_combined_t 类型。它包含了上面提到的hash类型，还有两个额外的keys arrays：dns_wc_head 和 dns_wc_tail。它的基本的初始化类似于普通hash。

```
ngx_hash_init_t      hash
ngx_hash_combined_t  foo_hash;

hash.hash = &foo_hash.hash;
hash.key = ...;
```

可以使用 NGX_HASH_WILDCARD_KEY 标记来添加通配符的key。

```
/* k1 = ".example.org"; */
/* k2 = "foo.*";        */
ngx_hash_add_key(&foo_keys, &k1, &data1, NGX_HASH_WILDCARD_KEY);
ngx_hash_add_key(&foo_keys, &k2, &data2, NGX_HASH_WILDCARD_KEY);
```

这个函数重新组织通配符和添加keys到对应的数组。详细用法和匹配算法参考map模块。

根据添加keys的内容，你可能需要初始化三个keys arrays：一个用于前面提到的精确数组，另外两个用于从头或尾的模糊匹配：

```
if (foo_keys.dns_wc_head.nelts) {

    ngx_qsort(foo_keys.dns_wc_head.elts,
              (size_t) foo_keys.dns_wc_head.nelts,
              sizeof(ngx_hash_key_t),
              cmp_dns_wildcards);

    hash.hash = NULL;
    hash.temp_pool = pool;

    if (ngx_hash_wildcard_init(&hash, foo_keys.dns_wc_head.elts,
                               foo_keys.dns_wc_head.nelts)
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    foo_hash.wc_head = (ngx_hash_wildcard_t *) hash.hash;
}
```

keys 数组需要先排序，然后初始化后的结果必须添加到合并hash。dns_wc_tail 也是类似的操作。

查找合并hash通过 ngx_hash_find_combined(chash, key, name, len)：

```
/* key = "bar.example.org"; — will match ".example.org" */
/* key = "foo.example.com"; — will match "foo.*"        */

hkey = ngx_hash_key(key.data, key.len);
res = ngx_hash_find_combined(&foo_hash, hkey, key.data, key.len);
```

内存管理
========

堆
--

nginx提供以下的函数用于从系统堆分配内存：

* ngx_alloc(size, log) — 从系统堆分配内存。这个封装了malloc()，并且带有log。分配错误和调试信息都会记录到log。
* ngx_calloc(size, log) — 和 ngx_alloc() 一样，但是将分配后的内存填充为0。
* ngx_memalign(alignment, size, log) — 从系统堆分配可对齐的内存。如果平台提供了posix_memalign()，就用它做为封装。否则返回调用传递最大对齐值参数的ngx_alloc()。
* ngx_free(p) — 释放内存。这是free()的封装。

内存池
------

大部份nginx分配使用内存池完成。在内存池分配的内存会在内存池销毁时自动释放。这样就提供了更好的分配性能，并且控制内存变的更简单。

内存池是通过在内部连续的内存块分配对象的。当一个块满时，新的块会被分配并且加入到该池的内存块列表。当块装不了一个大的分配时，分配会交给系统，然后返回指向存到该内存池，以后以后释放。

nginx 内存池类型为 ngx_pool_t。支持以下操作：

* ngx_create_pool(size, log) — 根据块大小创建内存池。返回pool对象也是在里面内存池里创建的。
* ngx_destroy_pool(pool) — 销毁整个内存池，包括pool对象自己。
* ngx_palloc(pool, size) — 从内存池分配对齐的内存。
* ngx_pcalloc(pool, size) — 从内存池分配对齐的内存并且置为0。
* ngx_pnalloc(pool, size) — 从内存池分配没对齐的内存。大部份用于分配字符串。
* ngx_pfree(pool, p) — 释放前面内存池中分配的内存。只对那些由系统分配的内存才会释放。

```
u_char      *p;
ngx_str_t   *s;
ngx_pool_t  *pool;

pool = ngx_create_pool(1024, log);
if (pool == NULL) { /* error */ }

s = ngx_palloc(pool, sizeof(ngx_str_t));
if (s == NULL) { /* error */ }
ngx_str_set(s, "foo");

p = ngx_pnalloc(pool, 3);
if (p == NULL) { /* error */ }
ngx_memcpy(p, "foo", 3);
```

因为链 ngx_chain_t 在nginx经常使用，所以nginx内存池提供了一种方式来复用它们。ngx_pool_t 的 chain 字段保留了原先已经分配的列表用来复用。 为了有效分配内存池中的chain，应当使用 ngx_alloc_chain_link(pool) 函数。该函数查找内存池中空闲的chain，只有当为空时才分配一个新的。使用ngx_free_chain(pool, cl) 可以回收chain。 

cleanup handler可以注册在pool里。cleanup handler 是一个带有参数的回调，在内存池销毁时调用。内存池通常在特定的nginx对象（比如HTTP请求），并且在对象的生命周期结束时销毁，以释放对象自己。注册内存池cleanup可以方便地释放资源，关闭文件描述符，或者做最后的关联在对象上的数据的调整。

通过调用ngx_pool_cleanup_add(pool, size)注册pool cleanup，它将返回 ngx_pool_cleanup_t 类型的指针，调用者会设置它。size 参数用分配cleanup上下文的大小。

```
ngx_pool_cleanup_t  *cln;

cln = ngx_pool_cleanup_add(pool, 0);
if (cln == NULL) { /* error */ }

cln->handler = ngx_my_cleanup;
cln->data = "foo";

...

static void
ngx_my_cleanup(void *data)
{
    u_char  *msg = data;

    ngx_do_smth(msg);
}
```

共享内存
--------

nginx用共享内存在进程之间共享公共的数据。函数 ngx_shared_memory_add(cf, name, size, tag) 添加新的共享内存实体到cycle。该函数接收 name 和 zone的大小。每个共享内存必须有唯一的名称。如果提供的名称存在，并且tag值也匹配，则会复用旧的zone实体。tag不匹配会被认为错误。通常模块地址会被当作tag的值，这样在模块里就能通过name来复用共享内存。

以下是 ngx_shm_zone_t 的字段：

* init — 初始化回调函数，在实际的共享内存映射后调用。
* data — data 上下文，传递给初始化回调函数。
* noreuse — 村记。禁止复用从旧的cycle里的共享内存。
* tag — 共享内存tag。
* shm — 类型为 ngx_shm_t 的特定平台对象，有以下几个字段：
    * addr — 映射的共享内存地址，初始为NULL
    * size — 共享内存大小
    * name — 共享内存名称
    * log — 共享内存log
    * exists — 标记。表示共享内存继承自主进程 (Windows特定)

共享内存zone实体会在ngx_init_cycle()解析配置后在映射到实际的内存。对POSIX系统，mmap() 系统调用用来创建匿名共享映射。对Windows，使用CreateFileMapping()/MapViewOfFileEx()对。

nginx提供了 ngx_slab_pool_t 来分配共享内存。对每个zone，slab pool会自动创建用来分配内存。这个池在共享zone的开头，并且通过表达式 (ngx_slab_pool_t *) shm_zone->shm.addr 访问。共享内存的分配通过调用 ngx_slab_alloc(pool, size)/ngx_slab_c alloc(pool, size) 函数完成，内存通过调用 ngx_slab_free(pool, p) 释放。

slab pool 将共享zone分成多个页。每个页被用于分配同样大小的对象。大小推荐为2的次方，并且不小于8。其它值被四舍五入。对每个页，bitmask被用来表示哪些块是已经使用的和哪些是空闲的。对大小超过半页（通常是2048字节），将按完整的页大小分配。

为了保护数据不会并发访问，需要有 ngx_slab_pool_t 的 mutex 字段。mutex 在分配和释放内存里被使用。然后它也可以用来保护其它分配自共享内存的数据。调用 ngx_shmtx_lock(&shpool->mutex) 锁住，调用 ngx_shmtx_unlock(&shpool->mutex) 解锁。

```
ngx_str_t        name;
ngx_foo_ctx_t   *ctx;
ngx_shm_zone_t  *shm_zone;

ngx_str_set(&name, "foo");

/* allocate shared zone context */
ctx = ngx_pcalloc(cf->pool, sizeof(ngx_foo_ctx_t));
if (ctx == NULL) {
    /* error */
}

/* add an entry for 65k shared zone */
shm_zone = ngx_shared_memory_add(cf, &name, 65536, &ngx_foo_module);
if (shm_zone == NULL) {
    /* error */
}

/* register init callback and context */
shm_zone->init = ngx_foo_init_zone;
shm_zone->data = ctx;


...

static ngx_int_t
ngx_foo_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_foo_ctx_t  *octx = data;

    size_t            len;
    ngx_foo_ctx_t    *ctx;
    ngx_slab_pool_t  *shpool;

    value = shm_zone->data;

    if (octx) {
        /* reusing a shared zone from old cycle */
        ctx->value = octx->value;
        return NGX_OK;
    }

    shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        /* initialize shared zone context in Windows nginx worker */
        ctx->value = shpool->data;
        return NGX_OK;
    }

    /* initialize shared zone */

    ctx->value = ngx_slab_alloc(shpool, sizeof(ngx_uint_t));
    if (ctx->value == NULL) {
        return NGX_ERROR;
    }

    shpool->data = ctx->value;

    return NGX_OK;
}
```

日志
=======

nginx用ngx_log_t对象记录日志。nginx的日志提供以下几种方式：

* stderr — 记录到标准错误输出
* file — 记录到文件
* syslog — 记录到syslog
* memory — 记录到内部内存用于开发的目的。这块内存可以在debugger时访问。

一个日志实例可以是一个日志对象链接，每个通过next连接起来。每个消息都被写到所有的日志对象。

每个日志对象有错误级别，用于限制消息写到它自己。以下是nginx提供的几种错误级别：

* NGX_LOG_EMERG
* NGX_LOG_ALERT
* NGX_LOG_CRIT
* NGX_LOG_ERR
* NGX_LOG_WARN
* NGX_LOG_NOTICE
* NGX_LOG_INFO
* NGX_LOG_DEBUG

对于调试日志，有以下几种选项：

* NGX_LOG_DEBUG_CORE
* NGX_LOG_DEBUG_ALLOC
* NGX_LOG_DEBUG_MUTEX
* NGX_LOG_DEBUG_EVENT
* NGX_LOG_DEBUG_HTTP
* NGX_LOG_DEBUG_MAIL
* NGX_LOG_DEBUG_STREAM

通常而言，日志是通过error_log指令创建的，并且在各个阶段都有效，cycle, 配置解析, 客户端连接和其它。

nginx提供以下的日志宏：

* ngx_log_error(level, log, err, fmt, ...) — 记录错误
* ngx_log_debug0(level, log, err, fmt), ngx_log_debug1(level, log, err, fmt, arg1) etc — 调试日志，提供最多8个可格式化的参数。

周期
=====

cycle 对象保持了nginx的运行时上文，由指定的配置创建。cycle的类型是 ngx_cycle_t。在配置重新加载后，新的cycle将从新版的配置创建，而旧的cycle通常在新的成功创建之后删除。目前活动的cycle保存在 ngx_cycle 这个全局变量并且继承自新启动的nginx进程。

cycle 是通过ngx_init_cycle()这个函数创建的。这个函数接收老的cycle作为参数。它用于定位配置并且尽可能多的继承旧的cycle以达到平滑过度。当nginx启动时，模拟的cycle被创建，然后被根据配置的正常cycle替换。

以下是cycle的一些字段：

* pool — cycle内存池。每个新的cycle都会创建一个内存池。
* log — cycle日志。初始时这个log继承自旧的cycle。当读完配置后，它会指向new_log。
* new_log - cycle日志。由配置创建。会根据最外层范围的error_log指令的设置而变化。
* connections, connections_n — 每个工作进程有一个类型为 ngx_connection_t 的数组 connections，由 event 模块在进程初始化时创建。connections的数目由worker_connections指令指定。
* files, files_n — 将文件描述符映射到nginx连接的数组。这个映射由event模块使用，具有NGX_USE_FD_EVENT标记（当前是poll和devpoll）。
* conf_ctx — 模块配置数组。这些配置在读nginx配置文件时创建和填充。
* modules, modules_n — 类型为 ngx_module_t 的模块数组，包括静态和通过当前配置加载的动态模块。
* listening — 类型为 ngx_listening_t 的监听socket数组。监听对象通常由不同模块的监听指令通过调用ngx_create_listening()函数添加。nginx基于这些监听对象创建监听socket。
* paths - 类型为 ngx_path_t 的数组。paths由那些想操作指定目录的模块通过调用ngx_add_path()添加。nginx读完配置之后，如果这些目录不存在，nginx会创建它们。些外，两个handler会被加到每个path：
    * path loader — 只会在nginx启动或配置加载60秒后执行一次。通常读取目录并将数据保存在共享内存里，这个handler由名为“nginx cache loader”的进程调用。
    * path manager — 定期执行。通常移走目录中旧的文件并将变化重新反映到共享内存里。这个handler由名为“nginx cache manager”的进程调用。
* open_files — 类型为ngx_open_file_t的列表。一个open file对象通过调用ngx_conf_open_file()创建。nginx读完配置后会根据open_files列表打开文件，并且将文件描述符保存在各自的open file对象的fd字段。这些文件会以追加模块打开，并且如果不存在时创建。nginx的工作进程在收到reopen信号（通常是USR1）后会重新打开被打开。这种情况下fd会变成新的文件描述符。目前这些打开的文件被用于日志。
* shared_memory — 共享内存zone的列表，通过调用ngx_shared_memory_add()函数添加。在所有nginx进程里，共享内存zone会映射到同样的地址，以共享所有的数据，比如HTTP缓存的in-memory树。

缓冲
======

nginx对 input/output 操作提供了类型为 ngx_buf_t 的buffer。它通常用于保存写入到目的的或从源读的数据。buffer可以将数据指向内存或文件。
 技术上来讲同时指向这两种也是可能的。缓冲区的内存是单独创建的，并且不会关联到 ngx_buf_t 这个结构体。

ngx_buf_t 结构体有以下字段：

* start, end — 内存块的边界，分配给这个缓冲区。
* pos, last — 内存缓冲区边界，一般在 start .. end 以内。
* file_pos, file_last — 文件缓冲区边界。相对文件开头的偏移量。
* tag — 唯一值。用于区分buffer，由不同的模块创建，通常是为了buffer复用。
* file — file对象。
* temporary — 临时标记。意味着这个buffer指向可写的内存。
* memory — 内存标记。表示这个buffer指向只读的内存。
* in_file — 文件村记。表示该当前buffer指向文件的数据。
* flush — 清空标记。表示应该清空这个buffer之前的所有数据。
* recycled — 可回收标记。表示该buffer可以被回收，而且应该尽快的使用。
* sync — 同步标记。表示这个buffer不带任何数据或特殊的像flush, last_buf这样的。一般这样的buffer在nginx里会被认为是错误的，这个标记允许略过错误检查。
* last_buf — 标记。表示这个buffer是输出的最后一个。
* last_in_chain — 标记。表示在(子)请求没有更多有数据的buffer了。
* shadow — 指向另一个buffer，与当前的buffer有关。通常当前buffer使用这个shadow buffer的数据。一量当前buffer使用完了，这个shadow buffer也应该被标记为已使用。
* last_shadow — 标记。表示当前buffer是最后一个buffer，并且指向特殊的shadow buffer。
* temp_file — 标记。表示这个buffer是临时文件。

输入输出 buffer 连接在一个链里。链是定义为下的一系列 ngx_chain_t 。

```
typedef struct ngx_chain_s  ngx_chain_t;

struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};
```

每个链保存着它的buffer，并且指向下一个链。

使用buffer和chain例子：

```
ngx_chain_t *
ngx_get_my_chain(ngx_pool_t *pool)
{
    ngx_buf_t    *b;
    ngx_chain_t  *out, *cl, **ll;

    /* first buf */
    cl = ngx_alloc_chain_link(pool);
    if (cl == NULL) { /* error */ }

    b = ngx_calloc_buf(pool);
    if (b == NULL) { /* error */ }

    b->start = (u_char *) "foo";
    b->pos = b->start;
    b->end = b->start + 3;
    b->last = b->end;
    b->memory = 1; /* read-only memory */

    cl->buf = b;
    out = cl;
    ll = &cl->next;

    /* second buf */
    cl = ngx_alloc_chain_link(pool);
    if (cl == NULL) { /* error */ }

    b = ngx_create_temp_buf(pool, 3);
    if (b == NULL) { /* error */ }

    b->last = ngx_cpymem(b->last, "foo", 3);

    cl->buf = b;
    cl->next = NULL;
    *ll = cl;

    return out;
}
```

网络
==========

连接
----------

连接结构体 ngx_connection_t 是socket描述符的封装。有如下字段：

* fd — socket描述符
* data — 任意连接上下文。通常指向更高层次的对象，构建在连接的上面，比如HTTP请求或Stream会话。
* read, write — 连接的读写事件。
* recv, send, recv_chain, send_chain — 连接的I/O操作。
* pool — 连接池
* log — connection日志
* sockaddr, socklen, addr_text — 客户端的二进制和文格形式的地址。
* local_sockaddr, local_socklen — 本地二进制形式地址。初始时这些为空，通过函数 ngx_connection_local_sockaddr() 得到本地socket地址。
* proxy_protocol_addr, proxy_protocol_port - PROXY protocol 客户端地址和端口，如果为连接开启了 PROXY protocol。
* ssl — nginx 连接 SSL 上下文
* reusable — 可复用村记。
* close — 关闭标记。表示连接是可复用的，并且应该被关闭。

nginx connection可以透传SSL层。这种情况下connection ssl字段指向一个ngx_ssl_connection_t结构体，保留着这个连接的SSL相关的数据，包括 SSL_CTX 和 SSL。处理函数 recv, send, recv_chain, send_chain 被设置成对应的SSL函数。

每个进程的connection数量被限制为 worker_connections 的值。所有的connection结构体会提前创建并且保存在cycle的connections这个字段里。通过 ngx_get_connection(s, log) 获得一个connection结构体。该函数接收socket描述符并且会在connection结构体里作封装。

国为每个进程有connection数的限制，nginx提供了一个抢占connection的方式。通过 ngx_reusable_connection(c, reusable) 允许或禁止connection的复用。调用 ngx_reusable_connection(c, 1) 设置reuse标记并且将connection加入 cycle 的 reusable_connections_queue。每当 ngx_get_connection() 发现 cycle 的 free_connections 无可用的 connection 时，它会调用 ngx_drain_connections() 以释放一定数量的可复用connection。对每个这样的 connection，关闭标记被设置并且读handler被调用以便通过调用ngx_close_connection(c)释放connection，然后将它设置为可复用。连接处于可复用状态下，调用ngx_reusable_connection(c, 0)可以取消复用。举个nginx里connection可复用的例子，在接收客户端的数据之前，HTTP客户端的connection会被标记为可复用。


事件
======

事件
----

事件对象 ngx_event_t 在nginx里提供了一种特定事件发生时能被通知的方式。

以下是 ngx_event_t 的一些字段：

* data — 任意的事件上下文，用于事件处理。通常指向 connection，使其绑定到事件。
* handler — 回调函数。当事件发生时调用。
* write — 写标记。表示这是一个写事件。用于区分读写事件。
* active — 活跃标记。表示该事件收到I/O通知后已经注册，一般来自像 epoll, kqueue, poll 这样的通知机制。
* ready — 就绪标记。表示这个事件接收到I/O通知。
* delayed — 延迟标记。意味着I/O由于限速而延迟。
* timer — 红黑树节点。用于添加进超时红黑树。
* timer_set — 定时器设置标记。意味着这个事件定时器被设置，但还未过期。
* timedout — 超时标记。意味着这个事件已经过期。
* eof — 读结束标记。表示读结束。
* pending_eof — 结束挂起标记。表示结束是在socket上挂起的，虽然可能还有一些数据可用。这个标记通过 epoll EPOLLRDHUP 事件 or kqueue EV_EOF 标记传递。
* error — 错误标记。意思当读或写时发生了错误。
* cancelable — 可取消标记。表示当nginx工作进程退出时，即使该事件没过期也能被立即调用。它提供了一种方式用来完成特定动作，比如清空日志文件。
* posted — 队列加入标记。意味这个事件已经加入了队列。
* queue — 队列节点。用于加到队列。

I/O事件
-------

每个通过调用ngx_get_connection()的 connection 有两个事件：c->read 和 c->write。这两事件用于接受可读写socket的通知。所有的这些事件都是边缘触发模式，意味着只有socket的状态变化时它们才会触发。举个例子，假设只读了部份数据，当有更多的数据到达时，nginx不会重新发读通知。即使底层的I/O通知机制本质上是水平触发的（poll, select等等），nginx将会把它们转成边缘触发。为了将不同平台的事件通知机制统一起来，当处理I/O socket通知或任何I/O操作后，必须调用ngx_handle_read_event(rev, flags) and ngx_handle_write_event(wev, lowat) 这两函数。通常这两函数在读或写事件处理结束后调用一次。

定时器事件
----------

事件可以被设置以通知超时过期。ngx_add_timer(ev, timer) 函数设置事件的超时时间，ngx_del_timer(ev) 删除前面设置的超时。当前为所有事件设置的超时都存放在一个全局的超时红黑树 ngx_event_timer_rbtree。这个树key的类型是 ngx_msec_t，值是从1970年1月1日算起的过期时间。这个树结构提供了快速的插入和删除，以及访问那些最小的超时。后者被nginx用于查找等待I/O事件的时间以及之后的过期事件。

已加事件
-------------

添加事件意味着它的handler会在某个时间点的事件遍历时调用。加入事件对简化代码和防止栈溢出是一个好的实践。加入的事件放在一个队列里。宏 ngx_post_event(ev, q) 加入事件到 post queue，ngx_delete_posted_event(ev) 从它所加入的队列中删除事件。通常事件加到 ngx_posted_events 这个队 列。 这个事件在稍后事件遍历中被事件(在所有的I/O和定时器事件已经处理后)。 ngx_event_process_posted() 函数用来处理事件队列。这个函数一直处理到列队为空，这意味着在当前的事件遍历过程中可以加更多的事件。

例子：

```
void
ngx_my_connection_read(ngx_connection_t *c)
{
    ngx_event_t  *rev;

    rev = c->read;

    ngx_add_timer(rev, 1000);

    rev->handler = ngx_my_read_handler;

    ngx_my_read(rev);
}


void
ngx_my_read_handler(ngx_event_t *rev)
{
    ssize_t            n;
    ngx_connection_t  *c;
    u_char             buf[256];

    if (rev->timedout) { /* timeout expired */ }

    c = rev->data;

    while (rev->ready) {
        n = c->recv(c, buf, sizeof(buf));

        if (n == NGX_AGAIN) {
            break;
        }

        if (n == NGX_ERROR) { /* error */ }

        /* process buf */
    }

    if (ngx_handle_read_event(rev, 0) != NGX_OK) { /* error */ }
}
```

遍历事件
----------

所有做I/O处理的nginx进程都有一个事件遍历。唯一没有I/O的进程是master进程，因为它花大部份时间在sigsuspend()上面，以等待信号的到达。事件遍历由 ngx_process_events_and_timers 函数实现。只要进程存在，这个函数就会一直重复的调用。它有以下几个阶段：

* 找出通过调用 ngx_event_find_timer() 的最小超时时间。该函数找到最左边的定时器树节点，并且返回该节点的到期毫秒数。
* 处理I/O事件。通过nginx配置选出对应的事件通知机制，然后处理。这个handler会一直待待至有I/O事件发生，或者最小的超时时间。对每个发生的读写事件，它的ready标记会被设置，它的handler会被调用。对Linux而言，通常会使用 ngx_epoll_process_events() 来调用 epoll_wait() 以等待I/O发生。
* 通过调用 ngx_event_expire_timers() 处理过期事件。这个定时器树会从最左侧的节点向右历遍，直到找到没有过期到期的超时。对每个超时的节点，timedout 标记会被设置，timer_set 会被重量置，并且事件handler会被调用。
* 通过调用 ngx_event_process_posted() 处理已加事件。这个函数一直重复删除和处理队列里的第一个无素，直到队列为空。

所有这些nginx进程也处理信号。信号handler只是设置了在 ngx_process_events_and_timers() 调用之后的全局变量。

进程
=========

nginx有好几种进程类型。当前进程的类型保存在ngx_process这个全局变量。

* NGX_PROCESS_MASTER — 主进程运行ngx_master_process_cycle()这个函数。主进程不能有任何的I/O，并且只对信号响应。它读取配置，创建cycle，启动和控制子进程。

* NGX_PROCESS_WORKER — 工作进程运行ngx_worker_process_cycle()函数。工作进程由子进程创建，处理客户端连接。他们同样也响应来自主进程的信号。

* NGX_PROCESS_SINGLE — 单进程只存在于master_process模式模式的情况下。生命周期函数是ngx_single_process_cycle()。这个进程创建生命周期并且处理客户端连接。

* NGX_PROCESS_HELPER — 目前只有两种help进程：cache manager 和 cache loader. 它们共用同样的生命周期函数ngx_cache_manager_process_cycle()。

所有的nginx处理如下信号：

* NGX_SHUTDOWN_SIGNAL (SIGQUIT) — 优雅结束。收到此信号后主进程发送 shutdown 信号给所有的子进程。当没有任何子进程时，主进程释放生命周期内存池然后结束。工作进程收到此信号后，关闭所有的监听端口然后一直等到超时树为空，最后释放生命周期内存池并且结束。cache 管理进程收到这个信号后立马退出。收到信号后 ngx_quit 设置为0，然后在处理完成后立马重置。ngx_exiting 在工作进程处理退出状态时设置为1。

* NGX_TERMINATE_SIGNAL (SIGTERM) - 终止。. 收到此信号后主进程发送 terminate 信号给所有的子进程。如果子进程1秒内没结束，它们会通过SIGKILL 信号被杀掉。当没有任何子进程时，主进程释放生命周期内存池然后结束。工作进程或cache管理进程释放生命周期内存池并且结束。ngx_terminate 在收到结信号后设置为1.

* NGX_NOACCEPT_SIGNAL (SIGWINCH) - 优雅结束工作进程。

* NGX_RECONFIGURE_SIGNAL (SIGHUP) - 配置热加载。 收到此信号后主进程根据配置文件创建新的cycle。如果这个新的cycle被成功的创建了，旧的cycle会被删除并且启动新的子进程。同时旧进程会被到 shutdown 信号。在单进程模式下，nginx 同样创建新的cycle，但是旧的会一直保留到所有跟它关联的连接都结束了。工作进程和helper进程忽略这种信号。

* NGX_REOPEN_SIGNAL (SIGUSR1) — 重新打开文件。主进程发送这个信号给工作进程。工作进程重新打开来自cycle的open_files。

* NGX_CHANGEBIN_SIGNAL (SIGUSR2) — 更新可执行程序。主进程启动新的可执行程序，将所有的监听文件描述符传给它。这些列表是通过环境变量“NGINX” 传递的，描述符值以分号分隔。新的nginx实例读这个变量然后将socket描述符添加到自己的初始cycle。其它进程忽略这种信号。

虽然nginx工作进程可以接受和处理POSIX信号，但是主进程却不通过调用标准kill()给工作进程和help进程发送信号。nginx通过内部进程间通道发送消息。即使这样，目前nginx也只是从主进程给工作进程发送消息。这些消息携带同样的信号。这些通过是socketpairs，其对端在不同的进程。

当运行可执行程序，可以通过-s参数指定几种值。分别是 stop, quit, reopen, reload。它们被转化成信号 NGX_TERMINATE_SIGNAL, NGX_SHUTDOWN_SIGNAL, NGX_REOPEN_SIGNAL 和 NGX_RECONFIGURE_SIGNAL 并且被发送给nginx主进程，通过从nginx pid文件获取进程id。

线程
====

可以将可能阻塞nginx工作进程的任务移到一个独立的线程。举例，nginx可以配置成使用线程来执行文件I/O操作。另一个例子是使用不具有异步接口的库，不能按通常方式用于nginx。请记住，线程接口是现有异步处理客户端连接的一种补充，而不是一种替代。

为了处理异步，可以使用以下原生pthread的封装：

```
typedef pthread_mutex_t  ngx_thread_mutex_t;

ngx_int_t ngx_thread_mutex_create(ngx_thread_mutex_t *mtx, ngx_log_t *log);
ngx_int_t ngx_thread_mutex_destroy(ngx_thread_mutex_t *mtx, ngx_log_t *log);
ngx_int_t ngx_thread_mutex_lock(ngx_thread_mutex_t *mtx, ngx_log_t *log);
ngx_int_t ngx_thread_mutex_unlock(ngx_thread_mutex_t *mtx, ngx_log_t *log);

typedef pthread_cond_t  ngx_thread_cond_t;

ngx_int_t ngx_thread_cond_create(ngx_thread_cond_t *cond, ngx_log_t *log);
ngx_int_t ngx_thread_cond_destroy(ngx_thread_cond_t *cond, ngx_log_t *log);
ngx_int_t ngx_thread_cond_signal(ngx_thread_cond_t *cond, ngx_log_t *log);
ngx_int_t ngx_thread_cond_wait(ngx_thread_cond_t *cond, ngx_thread_mutex_t *mtx,
    ngx_log_t *log);
```

nginx实现了线程池策略，而不是为每个任务创建一个线程。可以配置多个线程池用于不同的目的（举例，在不同的碰盘组上执行I/O）。每个线程池在启动时创建，并且包含一定数目的线程用来处理一个任务队列。当任务完成时，预定的handler就会被调用。

头文件 src/core/ngx_thread_pool.h 包含了对应的定义：

```
struct ngx_thread_task_s {
    ngx_thread_task_t   *next;
    ngx_uint_t           id;
    void                *ctx;
    void               (*handler)(void *data, ngx_log_t *log);
    ngx_event_t          event;
};

typedef struct ngx_thread_pool_s  ngx_thread_pool_t;

ngx_thread_pool_t *ngx_thread_pool_add(ngx_conf_t *cf, ngx_str_t *name);
ngx_thread_pool_t *ngx_thread_pool_get(ngx_cycle_t *cycle, ngx_str_t *name);

ngx_thread_task_t *ngx_thread_task_alloc(ngx_pool_t *pool, size_t size);
ngx_int_t ngx_thread_task_post(ngx_thread_pool_t *tp, ngx_thread_task_t *task);
```

在配置阶段，一个模块通过调用ngx_thread_pool_add(cf, name)获取线程池引用，以便使用线程。这个函数要么创建新的线程池，要么返回name对应存在的创建池引用。

在运行阶段，用ngx_thread_task_post(tp, task)函数将任务添加进tp线程池的队列。结构体ngx_thread_task_t包含了所有信息，用来执行线程里的用户函数，传递参数和建立完成时的处理handler。

```
typedef struct {
    int    foo;
} my_thread_ctx_t;


static void
my_thread_func(void *data, ngx_log_t *log)
{
    my_thread_ctx_t *ctx = data;

    /* this function is executed in a separate thread */
}


static void
my_thread_completion(ngx_event_t *ev)
{
    my_thread_ctx_t *ctx = ev->data;

    /* executed in nginx event loop */
}


ngx_int_t
my_task_offload(my_conf_t *conf)
{
    my_thread_ctx_t    *ctx;
    ngx_thread_task_t  *task;

    task = ngx_thread_task_alloc(conf->pool, sizeof(my_thread_ctx_t));
    if (task == NULL) {
        return NGX_ERROR;
    }

    ctx = task->ctx;

    ctx->foo = 42;

    task->handler = my_thread_func;
    task->event.handler = my_thread_completion;
    task->event.data = ctx;

    if (ngx_thread_task_post(conf->thread_pool, task) != NGX_OK) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

模块
=======

添加新模块
--------
标准nginx模块位于独立的目录，至少包含两个文件：config和包含模块源码的文件。config包含需要跟nginx整合的信息，比如：
```
ngx_module_type=CORE
ngx_module_name=ngx_foo_module
ngx_module_srcs="$ngx_addon_dir/ngx_foo_module.c"

. auto/module

ngx_addon_name=$ngx_module_name
```

这是个POSIX shell脚本，它能设置（或访问）以下变量：

* ngx_module_type — 模块类型。可选值包括 CORE, HTTP, HTTP_FILTER, HTTP_INIT_FILTER, HTTP_AUX_FILTER, MAIL, STREAM, or MISC
* ngx_module_name — 模块名称。可以用空格分隔并且单个源文件可以构造多个模块。如果是动态模块，第一个名称将作为二制进文件的名称。这些名称必须跟模块里面的能匹配。
* ngx_addon_name — 该模块在控制台的输出文本。
* ngx_module_srcs — 编译该模块时用到的源文件列表，用空格分隔。$ngx_addon_dir 变量可用作替代符，表示模块的当前路径。
* ngx_module_incs — 用于构建该模块的包含路径。
* ngx_module_deps — 模块依赖头文件列表。
* ngx_module_libs — 模块用到的链接库列表。 举个例子，libpthread 可以这样被链接 ngx_module_libs=-lpthread。这些宏可以直接在nginx里使用： LIBXSLT, LIBGD, GEOIP, PCRE, OPENSSL, MD5, SHA1, ZLIB, and PERL
* ngx_module_link — 模块链接形式，DYNAMIC表示动态模块，ADDON表示静态模块，其它根据不同的值会执行不同的操作。
* ngx_module_order — 模块顺序，设置模块的加载顺序在 HTTP_FILTER 和 HTTP_AUX_FILTER 类型的模块中是很有用的。模块按反序加载。
  在列表底部附近的 ngx_http_copy_filter_module 是最先被执行的。它读数据给其它的filter使用。在列表头部附近的ngx_http_write_filter_module 输出数据，并且是最后执行的。

  选项格式是这样的：当前模块名称紧接着用空格分隔的模块列表，这些列表位置靠前，但执行是靠后。这个模块将被插入在这个列表最后一个模块的前面。
  
  对filter模块默认是“ngx_http_copy_filter”，这样该模块被插入在copy filter之前，执行也就是copy filter的后面。对其它类型模块默认值为空。

模块通过使用 --add-module=/path/to/module 表示静态编译，--add-dynamic-module=/path/to/module 表示动态编译。

核心模块
-------

模块是nginx的构建方式，nginx的大部份功能也被实现成模块。模块源文件必须包含类型为 ngx_module_t 的全局变量，定义为：

```
struct ngx_module_s {

    /* private part is omitted */

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    /* stubs for future extensions are omitted */
};
```

省略私有部分包含模块版本，签名和预定义的宏 NGX_MODULE_V1。

每个模块包含有私有数据的上下文，特定的配置指令，命令数组，还有可能在特写被调用的函数钩子。模块的生命周期由下面这些组成：

* 配置指令处理函数在master进程解析配置文件时被调用。
* init_module 在master进程成功解析配置后调用。
* master进程创建了worker进程，然后调用这些worker进程各自的 init_process。
* 当一个工作进程收到来自master的shutdown命令后 exit_process 被调用。
* master进程在退出前调用 exit_master。

init_module 可能会被调用多次，如果master进程做了配置的reload。

init_master, init_thread and exit_thread 目前是没有实现的；线程在nginx里用于补充处理IO功能，而init_master看起来不是必须的。

type定义了模块类型，有以下几种：

* NGX_CORE_MODULE
* NGX_EVENT_MODULE
* NGX_HTTP_MODULE
* NGX_MAIL_MODULE
* NGX_STREAM_MODULE

NGX_CORE_MODULE 是最基础和通用的，处于最低层次的类型。其它类型都依赖在它上面，并且提供更方便的方式去处理各自领域的问题，比如事件和http请求。

核心模块有 ngx_core_module, ngx_errlog_module, ngx_regex_module, ngx_thread_pool_module, ngx_openssl_module，当然 http, stream, mail and event 也是。核心模块的上下文定义如下：

```
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```

name只是用于方便识别的模块字符串名称，create_conf 和 init_conf 指向创建和初始模块对应的配置结构体。对核心模块，create_conf在解析配置之前被调用， init_conf 在配置成功解析后调用。典型的 create_conf 函数分配空间用于配置，并且设置默认值。init_conf 处理已知配置，然后执行合理的校验和完成配置初始化。

举个例子，很简单的模块 ngx_foo_module 是这样的：

```
/*
 * Copyright (C) Author.
 */


#include <ngx_config.h>
#include <ngx_core.h>


typedef struct {
    ngx_flag_t  enable;
} ngx_foo_conf_t;


static void *ngx_foo_create_conf(ngx_cycle_t *cycle);
static char *ngx_foo_init_conf(ngx_cycle_t *cycle, void *conf);

static char *ngx_foo_enable(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_enable_post = { ngx_foo_enable };


static ngx_command_t  ngx_foo_commands[] = {

    { ngx_string("foo_enabled"),
      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_foo_conf_t, enable),
      &ngx_foo_enable_post },

      ngx_null_command
};


static ngx_core_module_t  ngx_foo_module_ctx = {
    ngx_string("foo"),
    ngx_foo_create_conf,
    ngx_foo_init_conf
};


ngx_module_t  ngx_foo_module = {
    NGX_MODULE_V1,
    &ngx_foo_module_ctx,                   /* module context */
    ngx_foo_commands,                      /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};


static void *
ngx_foo_create_conf(ngx_cycle_t *cycle)
{
    ngx_foo_conf_t  *fcf;

    fcf = ngx_pcalloc(cycle->pool, sizeof(ngx_foo_conf_t));
    if (fcf == NULL) {
        return NULL;
    }

    fcf->enable = NGX_CONF_UNSET;

    return fcf;
}


static char *
ngx_foo_init_conf(ngx_cycle_t *cycle, void *conf)
{
    ngx_foo_conf_t *fcf = conf;

    ngx_conf_init_value(fcf->enable, 0);

    return NGX_CONF_OK;
}


static char *
ngx_foo_enable(ngx_conf_t *cf, void *post, void *data)
{
    ngx_flag_t  *fp = data;

    if (*fp == 0) {
        return NGX_CONF_OK;
    }

    ngx_log_error(NGX_LOG_NOTICE, cf->log, 0, "Foo Module is enabled");

    return NGX_CONF_OK;
}
```

配置指令
------------------------

ngx_command_t 表示一个配置指令。每个模块包含一组指令，每个指令的格式表示了如何处理参数和解析时调用的函数。

```
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```

指令数组以 “ngx_null_command” 结束。name 是指令名称，体现在配置文件中，比如 “worker_processes” or “listen”。type 是bit组合，表示参数个数，指令类型和其它对应的属性。参数的标记为：

* NGX_CONF_NOARGS — 没有参数
* NGX_CONF_1MORE — 至少一个参数
* NGX_CONF_2MORE — 至少两个参数
* NGX_CONF_TAKE1..7 — 明确的1..7个参数
* NGX_CONF_TAKE12, 13, 23, 123, 1234 — 一个或两个参数，一个或参数，依此类推。

指令类型：

* NGX_CONF_BLOCK — 表示是一个块，比如它可能用 { } 包含其它指令，或自己实现的解析以处理包含的内容，比如 map 指领。
* NGX_CONF_FLAG — 表示是个boolean的标记，“on” 或者 “off”。

指令的上下文定义了配置的位置，并且关联到对应的存储配置的地方。

* NGX_MAIN_CONF — 上层配置
* NGX_HTTP_MAIN_CONF — http 块
* NGX_HTTP_SRV_CONF — http server 块
* NGX_HTTP_LOC_CONF — http location 块
* NGX_HTTP_UPS_CONF — http upstream 块
* NGX_HTTP_SIF_CONF — http server “if” 块
* NGX_HTTP_LIF_CONF — http location “if” 块
* NGX_HTTP_LMT_CONF — http “limit_except” 块
* NGX_STREAM_MAIN_CONF — stream 块
* NGX_STREAM_SRV_CONF — stream server 块
* NGX_STREAM_UPS_CONF — stream upstream 块
* NGX_MAIL_MAIN_CONF — mail 块
* NGX_MAIL_SRV_CONF — mail server 块
* NGX_EVENT_CONF — event 块
* NGX_DIRECT_CONF — 没有层级的上下文，直接存储在模块的ctx

配置解析时根据这些标记，要么对放错位置的指令抛出错误，要么调用指令handler，这样即使相同的配置在不同的location也能存储到能区分的位置。

set字段定义了解析配置时调用的handler，并且将解析的值存放到对应的配置结构体。Nginx提供了一些方便的公共函数集：

* ngx_conf_set_flag_slot — 将 “on” or “off” 转化成 ngx_flag_t 类型的值 1 or 0
* ngx_conf_set_str_slot — 存储类型为 ngx_str_t 的值
* ngx_conf_set_str_array_slot — 追加元素为ngx_str_t的ngx_array_t一个新的值。array会自动创建，如果不存在的话。
* ngx_conf_set_keyval_slot — 追加元素为ngx_keyval_t的ngx_array_t一个新的值。第一个作为键，第二个作为值，如果不存在的话。
* ngx_conf_set_num_slot — 转化参数为 ngx_int_t 类型的值
* ngx_conf_set_size_slot — 转化参数为 size_t 类型的值
* ngx_conf_set_off_slot — 转化参数为 off_t 类型的值
* ngx_conf_set_msec_slot — 转化参数为 ngx_msec_t 类型的值
* ngx_conf_set_sec_slot — 转化参数为 time_t 类型的值
* ngx_conf_set_bufs_slot — 转化两个参数为 ngx_bufs_t，包含了 ngx_int_t 类型的 number 和 buffers的size
* ngx_conf_set_enum_slot — 转化参数为 ngx_uint_t 类型的值。这是个类似枚举的功能，可以传以 null-terminated 结尾的 ngx_conf_enum_t 数组给post字段，以设置对应的值。
* ngx_conf_set_bitmask_slot — 转化参数为 ngx_uint_t 类型的值。这是个类似枚举的功能，可以传以 null-terminated ngx_conf_bitmask_t 数组给post字段，以设置对应的值。 
* set_path_slot — 转化参数为 ngx_path_t 类型并且做必须的初始化。详情请看 proxy_temp_path 指令
* set_access_slot — 转化参数为文件权限mask。详情请看 proxy_store_access 指令。

conf字段定义了哪个上下文用来存储指令的值，或者用NULL表示不使用上下文。简单的核心模块不用配置上下文并且设置 NGX_DIRECT_CONF 标识。 在真实场景里，像http或stream的模块往往更复杂，配置可以在pre-server或者pre-location里，还有甚至是在 "if" 里的。这样的模块里，配置结构会更复杂，请到一些模块里看他们是如何管理各自的配置的。

* NGX_HTTP_MAIN_CONF_OFFSET — http 块配置
* NGX_HTTP_SRV_CONF_OFFSET — http 块配置
* NGX_HTTP_LOC_CONF_OFFSET — http 块配置
* NGX_STREAM_MAIN_CONF_OFFSET — stream 块配置
* NGX_STREAM_SRV_CONF_OFFSET — stream server 块配置
* NGX_MAIL_MAIN_CONF_OFFSET — mail 块配置
* NGX_MAIL_SRV_CONF_OFFSET — mail server 块配置

offset字段定义了存储该指令值的位置在配置结构体的偏移大小。典型的使用是调用 offsetof() 宏。

post字段包含双重意思：它可能在主handler完成后调用，或者传额外的数据给主handler。第一种情况 ngx_conf_post_t 需要初始化handler，举个例子：

```
static char *ngx_do_foo(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_post = { ngx_do_foo };
```

post函数参数是：ngx_conf_post_t它自己, data 来自主handler的参数。


HTTP
====

连接
----

每个HTTP客户端连接经历以下几个阶段：

* ngx_event_accept() 接受一个客户端TCP连接。这个函数在监听socket发生读通知时被调用。在这阶段创建新的 ngx_connecton_t 对象。这个对象封装了新接受的客户端socket。每个nginx监听会提供并传递给这个新的connection对象一个handler。比如 HTTP connection 是ngx_http_init_connection(c)。
* ngx_http_init_connection() 执行了HTTP connection的早期初始化。这个阶段为connection创建了一个 ngx_http_connection_t 对象，并且引用存放在 connection 的 data 字段。稍后会被替换成 HTTP request 对象。PROXY 协议的解析和 SSL 握手也发生在这个阶段。
* ngx_http_wait_request_handler() 是读事件handler，当客户端socket有数据可读时被调用。在这个阶段会创建 HTTP request 对象 ngx_http_request_t 并且设置到 connection 的 data 字段。
* ngx_http_process_request_line() 是读事件handler，用来读请求行。这个 handler 在 ngx_http_wait_request_handler() 里设置。数据被读进 connection 的 buffer。 buffer的大小初始值是指令 client_header_buffer_size。整个 client header 应该是适合这个buffer的。如果这个初始值不够时，会分配一个更大的buffer，它的大小为指令large_client_header_buffers的值。
* ngx_http_process_request_headers() 是读事件handler，在 ngx_http_process_request_line() 之后设置，被用来读请求头。
* ngx_http_core_run_phases() 当整个http请求头读完和解析后调用。这个函数运行从 NGX_HTTP_POST_READ_PHASE 到 NGX_HTTP_CONTENT_PHASE 的请求阶段。最后阶段产生响应内容并传给整个filter链。响应不一定要在这阶段发给客户端。它可能缓冲起来然后在最后阶段发送。
* ngx_http_finalize_request() 通常在请求已经产生了所有的输出或发生错误时调用。后者会查找合适的错误页面作为响应。如果响应没有完全的发送给客户端，HTTP写处理 ngx_http_writer() 会被激活以完成数据的发送。
* ngx_http_finalize_connection() 在响应完全发送给客户端后调用，然后销毁请求。如果客户端连接的keepalive功能启用了，ngx_http_set_keepalive() 会被调用，用来销毁当前请求并等待这个连接的下一个请求。否则，调用 ngx_http_close_request() 同时销毁请求和连接。

请求
----

对每个客户端HTTP请求创建一个ngx_http_request_t对象。以下是这个对象的一些字段：

* connection — 指向类型为 ngx_connection_t 的 connection 对象。多个请求可能同时指向同个连接 - 一个主请求和它的多个子请求。一个请求被删除后，新的请求可能会在同样的连接上被创建。

    注意：HTTP连接 ngx_connection_t 的 data 字段会指向这个请求。这种请求被认为是激活的，相反的其它该连接上的请求则不是。激活的请求被用来处理客户端事件，并且允许发送它的响应给客户端。通常每个请求会在某个时间点激活以发送它的数据。

* ctx — 一组HTTP模块的上下文。每个类型为 NGX_HTTP_MODULE 的模块在这个请求里可以存任意的东西（通常指向一个结构体）。值存放在模块ctx_index位置上对应ctx数组的地方。以下宏提供了获取和设置请求上下文的方便方式。
    * ngx_http_get_module_ctx(r, module) — 返回模块的上下文。
    * ngx_http_set_ctx(r, c, module) — 设置c为模块的上下文。
* main_conf, srv_conf, loc_conf — 当前请求的配置数组。配置存放在模块的ctx_index对应的位置。
* read_event_handler, write_event_handler - 请求的读写事件handler。通常，HTTP连接用 ngx_http_request_handler() 作为读写事件 handler。这个函数会调用当前激活请求的 read_event_handler 和 write_event_handler。
* cache — 用于缓存上游响应的缓存对象。
* upstream — 用于代理的上游对象。
* pool — 请求内存池。这个内存池在请求被删除后被销毁。这个请求对象本身也是从该内存池分配的。对需要活动在整个客户端连接生命周期的分配，应该使用 ngx_connection_t 的 内存池。
* header_in — 从请求头读的buffer。
* headers_in, headers_out — 输入和输出的 HTTP 头部对象。两个对象都包含类型为 ngx_list_t 的 headers 头部域，用来保存原始的头部列表。此外还有比较特别的单独字段，用来直接获取和设置，比如 content_length_n, status 等等。
* request_body — 客户端请求体对象。
* start_sec, start_msec — 请求创建时间点。用于跟踪请求时间。
* method, method_name — 客户端HTTP请求方法的数字和文本表示方式。方法的数字值定义在 src/http/ngx_http_request.h，有 NGX_HTTP_GET, NGX_HTTP_HEAD, NGX_HTTP_POST 等宏。
* http_protocol, http_version, http_major, http_minor - 客户端HTTP协议和版本的文本形式 (“HTTP/1.0”, “HTTP/1.1” 等)，数字形式 (NGX_HTTP_VERSION_10, NGX_HTTP_VERSION_11 等) 和主次版本号
* request_line, unparsed_uri — 客户端原始的请求行和URI。
* uri, args, exten — 当前请求的请求URI, 参数和文件扩展名。URI值可能由于规范跟客户端发送过来的原始URI不同。经过请求处理，这些值可能在内部重定向时发生改变。
* main — 指向主请求对象。创建这个创建用来处理HTTP请求，而那些子请求被创建用来执行主请求里的特定子任务。
* parent — 子请求指向的父请求。
* postponed — 依次要发送和创建的buffer和子请求列表。这个列表被用在 postpone filter 以提供连续的请求输出，它的各部份由子请求创建。
* post_subrequest — 不用于主请求。指向子请求完成会调用的具有上下文的handler。不用于主请求。
* posted_requests — 开始要执行或恢复的请求列表。通过调用请求的write_event_handler完成启动或恢复。通常这个handler会保留请求主函数，第一个运行请求阶段并且产生输出的。

    一个请求经常通过调用 ngx_http_post_request(r, NULL)加到posted_requests。这样会加到主请求的 posted_requests 列表里。函数会 ngx_http_run_posted_requests(c) 会运行所有的请求，这些添加在通过连接激活请求对应的主请求。这个函数应该在所有的事件处理中调用，这样能产生新的添加请求。通常在执行了请求的读写处理后调用。

* phase_handler — 当前请求阶段的索引。
* ncaptures, captures, captures_data — 请求最后一次正则匹配产生的正则capture。当处理一个请求时，有很多地方可以发生正则匹配：map 查找， server 通过 SNI 或 HTTP Host 查找，rewrite, proxy_redirect 等等。capture 在查找时产生并且保存这些字段里。字段 ncaptures 保存caputure的个数, captures 保存 capture 边界，captures_data 保存字符串，针对这些匹配到的正则和被用于精确的capture。每次正则匹配后，请求capture会重置并且保存新的值。
* count — 请求引用计数。这个字段只发生在主请求上。通过简单的 r->main->count++ 就可以递增。要通过 ngx_http_finalize_request(r, rc) 递减。创建子请求和运行读请求体处理都会增加这个计数。
* subrequests — 当前子请求的嵌套级别。每个子请求会让它的父请求的嵌套级别数减1。一旦这个值到达0就会发生错误，主请求的这个值定义为 NGX_HTTP_MAX_SUBREQUESTS 这个常量。
* uri_changes — 请求的URI剩余可改变数。一个请求可以改变它的URI的总次数限制为 NGX_HTTP_MAX_URI_CHANGES 这个常量。每次变化都会递减直到0。后者会导致错误发生。这些被认为是改变URI的操作是重写和内部重定向到普通或有命名的location。
* blocked — 请求上的阻塞次数。只要此值为非0,请求不会被终止。目前这个值会由于待处理AIO（POSIX AIO和线程操作）操作和缓存锁增加。
* buffered — 位，表示一些模块缓冲了请求产生的输出。一些filter都可以缓冲输出，比如 sub_filter 可以缓冲数据用来作部分字符串匹配，copy filter 因为缺少空闲的output_buffers缓冲数据，等等。只要这个值为非0，请求就不会终止，期望继续刷新。
* header_only — 标记。用于表示不需要输出请求体。举例，这个标记用于 HTTP HEAD 请求。
* keepalive — 标记。用于表示否支持客户端的持久连接。这个值根据 HTTP 版本和 头部 "Connection" 的值推算出。

* header_sent — 标记。表示请求的头部信息已经发送（不一定发到客户端）。
* internal — 标记。表示当前请求是内部的。要进入这种内部的状态，请求必须通过内部重定向或者是一个子请求。内部请求进入内部的location。
* allow_ranges — 标记。用于表示如果是HTTP Range的请求，可以发送部份响应给客户端。
* subrequest_ranges — 标记。用于表示处理子请求时，允许发送部分响应给客户端。
* single_range — 标记。表示只有一个连续的range能被发送给客户端。这个标记通常在发送数据流时设置，比如来自代理服务器，并且整个响应不是一次完成的。
* main_filter_need_in_memory, filter_need_in_memory — 标记。用于表示输出应该产生自内存，而非文件。这个被copy filter用来从文件buffer读数据，即使开了sendfile。两者的匹别在设置它们的filter模块的location。这些在postpone filter调用之前的filters，设置了filter_need_in_memory 表明当前请求的输出应该来自memory buffer。在之后调用的filter设置 main_filter_need_in_memory 表明主请求和子请求在发送输出时都要从读文件到内存里。
* filter_need_temporary — 表示请求输出应该产生自 temporary buffer，而且不能是只读的memory buffer或file buffer。这个用于那些可能直接改变要发送buffer输出的filter。

配置
-------------

每个HTTP模块都可以有三种类型的配置：

* Main配置。 此配置作用于整个http{}块，属于全局配置。此配置中存储了模块的全局配置。
* Server配置. 此配置作用于一个server{}块，用于存储模块server特有的配置。
* Location配置. 此配置作用于一个location{}块，if{}块或者limit_except()块，用于存储location相关的配置。

上述配置的结构体是在nginx的配置阶段，通过调用一系列函数来创建的。这些函数会为配置结构体分配内存，并进行初始化和合并操作。下面的例子演示了如何创建一个简单的location配置。该配置中只有一个无符号整形的配置项foo。

```
typedef struct {
    ngx_uint_t  foo;
} ngx_http_foo_loc_conf_t;


static ngx_http_module_t  ngx_http_foo_module_ctx = {
    NULL,                                  /* preconfiguration */
    NULL,                                  /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_foo_create_loc_conf,          /* create location configuration */
    ngx_http_foo_merge_loc_conf            /* merge location configuration */
};


static void *
ngx_http_foo_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_foo_loc_conf_t  *conf;

    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_foo_loc_conf_t));
    if (conf == NULL) {
        return NULL;
    }

    conf->foo = NGX_CONF_UNSET_UINT;

    return conf;
}


static char *
ngx_http_foo_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_foo_loc_conf_t *prev = parent;
    ngx_http_foo_loc_conf_t *conf = child;

    ngx_conf_merge_uint_value(conf->foo, prev->foo, 1);
}
```

在例子中可见，ngx_http_foo_create_loc_conf()函数创建了一个新的配置结构，ngx_http_foo_merge_loc_conf()函数则将配置和更高层次的配置进行合并。实际上，server和location的配置并不仅仅存在于server和location这两个配置层次中，而是为相应更高的配置层次全部进行创建。具体来说，server配置也会在main层次进行创建，而location配置同时会在main, server和location三个层次创建。这些配置使得server和location的配置出现在任何层次的nginx配置中成为了可能。最终各级配置会进行合并。为了在合并的时候识别出缺失的配置并进行忽略，nginx提供了一系列类似于NGX_CONF_UNSET和NGX_CONF_UNSET_UINT这样的宏。标准的nginx合并宏，比如ngx_conf_merge_value()和ngx_conf_merge_uint_value()，提供了一种更加方便的方法来对配置选项进行合并，此外如果在配置文件中没有显式的进行配置，上述合并宏还可以设置默认值。完整的合并宏请参考src/core/ngx_conf_file.h文件。

可以使用如下这些宏来再配置阶段访问HTTP模块的配置。它们的第一个参数都是ngx_conf_t类型的指针。

* ngx_http_conf_get_module_main_conf(cf, module)
* ngx_http_conf_get_module_srv_conf(cf, module)
* ngx_http_conf_get_module_loc_conf(cf, module)

下面的例子展示了nginx核心模块ngx_http_core_module的location配置的指针，并修改其content handler内容的过程。

```
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r);


static ngx_command_t  ngx_http_foo_commands[] = {

    { ngx_string("foo"),
      NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
      ngx_http_foo,
      0,
      0,
      NULL },

      ngx_null_command
};


static char *
ngx_http_foo(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf->handler = ngx_http_bar_handler;

    return NGX_CONF_OK;
}
```

在运行阶段，可以使用下面的这些宏来获取HTTP模块的配置。

* ngx_http_get_module_main_conf(r, module)
* ngx_http_get_module_srv_conf(r, module)
* ngx_http_get_module_loc_conf(r, module)

需要将指向表示HTTP请求的ngx_http_request_t结构体的指针传递给这些宏。对于一个请求，main配置从不会发生变化，server配置会在切换虚拟服务器配置后发生改变。请求的location配置会随着rewrite或者内部重定向而被多次改变。下面的例子展示了如何在运行阶段获取HTTP配置。

```
static ngx_int_t
ngx_http_foo_handler(ngx_http_request_t *r)
{
    ngx_http_foo_loc_conf_t  *flcf;

    flcf = ngx_http_get_module_loc_conf(r, ngx_http_foo_module);

    ...
}
```


阶段
----
每个HTTP请求都会经过一系列HTTP阶段（phase），其中每个阶段都会负责处理不同的功能。大部分阶段允许注册handler，这些阶段的handler会在请求到达这个阶段的时候被调用。很多标准nginx模块通过注册阶段handler的方式来实现在某个请求处理阶段被调用模块逻辑。下面是nginx HTTP阶段列表：

* NGX_HTTP_POST_READ_PHASE是最开始的一个阶段。ngx_http_realip_module模块在此注册了handler，这样一来就可以在其他模块被触发之前就替换掉客户端的IP地址。
* NGX_HTTP_SERVER_REWRITE_PHASE是用来执行server层面rewrite脚本的阶段。ngx_http_rewrite_module模块在这里注册handler。
* NGX_HTTP_FIND_CONFIG_PHASE — 基于请求URI来查找location的特殊阶段。这个阶段里不允许注册任何handler。该阶段只执行匹配location的动作。在进入到这个阶段之前，请求中的location被设置成了对应server中的默认location，任何模块获取请求的location配置，只会得到默认location的配置。这个阶段之后，请求将会得到新的location配置。
* NGX_HTTP_REWRITE_PHASE — 和NGX_HTTP_SERVER_REWRITE_PHASE阶段类似，不过是执行上个阶段新选择的location中的rewrite动作。
* NGX_HTTP_POST_REWRITE_PHASE — 用于将请求重定向到新location的特殊阶段，这种重定向会在URI被rewrite过的情况下发生。重定向是通过重新跳转回NGX_HTTP_FIND_CONFIG_PHASE阶段来实现的。该阶段不允许注册handler。
* NGX_HTTP_PREACCESS_PHASE — 这是一个可以注册不同类型handler的通用阶段，此时没有进行过访问控制检查。标准nginx模块ngx_http_limit_conn_module和ngx_http_limit_req_module在此阶段注册了handler。
* NGX_HTTP_ACCESS_PHASE — 对请求进行访问控制权限检查的阶段。ngx_http_access_module和ngx_http_auth_basic_module这些标准nginx模块在此阶段注册handler。如果使用satisfy指令进行相应的配置，则可以实现只要任意一个handler进行了放行，请求就可以继续后续的处理。
* NGX_HTTP_POST_ACCESS_PHASE — 对于satisfy设置为any时候的特殊阶段。如果某些access阶段的handler阻断了了访问且没有其他handler放行，则请求会被阻断。此阶段不允许注册任何handler。
* NGX_HTTP_TRY_FILES_PHASE — 实现try_file功能的特殊阶段。此阶段不允许注册任何handler。
* NGX_HTTP_CONTENT_PHASE — 用于生成HTTP应答的阶段。多个标准nginx模块在此阶段注册handler，例如ngx_http_index_module和ngx_http_static_module模块。所有注册的这些模块handler会被按照顺序调用直到其中的一个生成输出。也可以基于每个location单独设置content handler。如果ngx_http_core_module模块的location配置中的handler成员被设置，则在NGX_HTTP_CONTENT_PHASE阶段此handler会被调用，注册到此阶段的其他handler会被忽略。
* NGX_HTTP_LOG_PHASE用来对请求记录日志。当前，只有ngx_http_log_module模块在此阶段注册handler以便记录访问日志。Log阶段handler在每个请求结束，但还没被释放的时候被调用。

以下是使用preaccess阶段handler的例子：

```
static ngx_http_module_t  ngx_http_foo_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_foo_init,                     /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    NULL,                                  /* create location configuration */
    NULL                                   /* merge location configuration */
};


static ngx_int_t
ngx_http_foo_handler(ngx_http_request_t *r)
{
    ngx_str_t  *ua;

    ua = r->headers_in->user_agent;

    if (ua == NULL) {
        return NGX_DECLINED;
    }

    /* reject requests with "User-Agent: foo" */
    if (ua->value.len == 3 && ngx_strncmp(ua->value.data, "foo", 3) == 0) {
        return NGX_HTTP_FORBIDDEN;
    }

    return NGX_DECLINED;
}


static ngx_int_t
ngx_http_foo_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_foo_handler;

    return NGX_OK;
}
```

阶段的handler可以返回如下返回值：

* NGX_OK — 继续执行下个阶段
* NGX_DECLINED — 继续执行当前阶段的下一个handler。如果当前handler是本阶段的最后一个handler，则执行下个阶段。
* NGX_AGAIN, NGX_DONE — 挂起阶段处理直到事件发生。此场景可以用来处理异步I/O操作或者进行延迟处理。阶段处理应该通过对ngx_http_core_run_phases()函数的调用来恢复。
* 任何其他的返回值都被视为请求结束，尤其是HTTP应答码，这种情况下会以返回的HTTP应答码结束当前请求。

一些阶段对返回值的处理稍有不同。在content阶段，除了NGX_DECLINED之外的任何返回值都会被当成结束请求处理。对于location提供的content handler，任何返回值都会别当成结束状态码进行处理。在access阶段，如果使用了satisfy any模式，返回除了NGX_OK，NGX_DECLINED，NGX_AGAIN和NGX_DONE之外的值会被作为阻断处理。如果没有其他的access handler对请求放行或者通过一个返回码阻断，则前述导致阻断的返回值会被当成结束状态码。

变量
----

访问已有变量
----------

变量可以通过索引（即index，这是最常用的方式）或者名字（参考下文关于创建变量的章节）。索引是在配置阶段，当一个变量添加到配置中的时候创建。变量索引可以通过ngx_http_get_variable_index()函数获取：

```
ngx_str_t  name;  /* ngx_string("foo") */
ngx_int_t  index;

index = ngx_http_get_variable_index(cf, &name);
```

这里，cf变量是一个指向nginx配置的指针，name则指向变量名称字符串。该函数在执行出错时候返回NGX_ERROR，其他情况下典型的做法是将返回的索引存储在模块配置中以便后续使用。

所有的HTTP变量都是基于HTTP请求的上下文而计算的，其结果也是与HTTP请求相关并存储于其中。所有用于计算变量的函数的返回值都是ngx_http_variable_value_t类型，该类型代表了一个变量的值。

```
typedef ngx_variable_value_t  ngx_http_variable_value_t;

typedef struct {
    unsigned    len:28;

    unsigned    valid:1;
    unsigned    no_cacheable:1;
    unsigned    not_found:1;
    unsigned    escape:1;

    u_char     *data;
} ngx_variable_value_t;
```

说明：

* len — 值的长度
* data — 值本身
* valid — 值是有效的
* not_found — 变量没有找到，因此data和len成员无意义；例如，像尝试获取$arg_foo这种类型的变量的值，而请求中却没有名为foo的参数时，就可能发生这种情况。
* no_cacheable — 禁止缓存结果值
* escape — 由日志模块内部使用，用来标记在输出时需要进行转移的变量值

ngx_http_get_flushed_variable()和ngx_http_get_indexed_variable()函数用来获取变量值。它们拥有相同的接口 —— 一个HTTP请求r作为计算变量值的上下文以及一个index参数，用于指示哪个变量。以下是一个典型的用法：

```
ngx_http_variable_value_t  *v;

v = ngx_http_get_flushed_variable(r, index);

if (v == NULL || v->not_found) {
    /* we failed to get value or there is no such variable, handle it */
    return NGX_ERROR;
}

/* some meaningful value is found */
```

这两个函数的区别是，ngx_http_get_indexed_variable()返回缓存的变量值而ngx_http_get_flushed_variable()函数对于不可缓存的变量进行刷新处理。

有一些场景中需要处理那些在配置阶段还不知道名字的变量，这些变量无法通过使用索引来访问，例如SSI和Perl模块。对于这类场景，可以使用ngx_http_get_variable(r, name, key)函数。该函数通过变量名字和它的哈希key来查找变量。

创建变量
-------

ngx_http_add_variable()函数用来创建一个变量。其参数有：配置（注册变量的配置），变量名和用来控制变量行为的标记位：

* NGX_HTTP_VAR_CHANGEABLE  — 允许变量被重新定义；如果另外一个模块使用同样的名字定义变量，不会产生冲突。例如，这个特点允许用户使用set指令覆盖变量。
* NGX_HTTP_VAR_NOCACHEABLE  — 禁止缓存，在类似于$time_local这样的变量上使用。
* NGX_HTTP_VAR_NOHASH  — 标识这个变量只能通过索引访问，不允许通过变量名访问。这是一个小的优化，可以在类似于SSI或者Perl这样的模块中不需要此变量的时候使用。
* NGX_HTTP_VAR_PREFIX  — 该变量的名字是一个前缀。相关的handler必须实现额外的逻辑来获取指定的变量值。例如，所有"arg_"变量都被同一个handler处理，该handler在请求的参数中查找并返回对应的参数值。

此函数在失败时返回NULL，否则返回一个指向ngx_http_variable_t类型的指针：

```
struct ngx_http_variable_s {
    ngx_str_t                     name;
    ngx_http_set_variable_pt      set_handler;
    ngx_http_get_variable_pt      get_handler;
    uintptr_t                     data;
    ngx_uint_t                    flags;
    ngx_uint_t                    index;
};
```

get和set handler被用来获取以及设置变量的值，data成员会被传递给变量handler，index成员中存储的是分配的变量索引，用来引用变量。

通常，一个以null结尾的上述结构体数组会在模块中创建，并在preconfiguration阶段将数组中的变量添加到配置中：

```
static ngx_http_variable_t  ngx_http_foo_vars[] = {

    { ngx_string("foo_v1"), NULL, ngx_http_foo_v1_variable, NULL, 0, 0 },

    { ngx_null_string, NULL, NULL, 0, 0, 0 }
};

static ngx_int_t
ngx_http_foo_add_variables(ngx_conf_t *cf)
{
    ngx_http_variable_t  *var, *v;

    for (v = ngx_http_foo_vars; v->name.len; v++) {
        var = ngx_http_add_variable(cf, &v->name, v->flags);
        if (var == NULL) {
            return NGX_ERROR;
        }

        var->get_handler = v->get_handler;
        var->data = v->data;
    }

    return NGX_OK;
}
```

HTTP模块上下文中的preconfiguration成员会被赋值为这个函数，并在解析HTTP配置之前被调用，所以它可以处理这些变量。

get handler负责为某个请求计算变量的值，例如：

```
static ngx_int_t
ngx_http_variable_connection(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    u_char  *p;

    p = ngx_pnalloc(r->pool, NGX_ATOMIC_T_LEN);
    if (p == NULL) {
        return NGX_ERROR;
    }

    v->len = ngx_sprintf(p, "%uA", r->connection->number) - p;
    v->valid = 1;
    v->no_cacheable = 0;
    v->not_found = 0;
    v->data = p;

    return NGX_OK;
}
```

如果内部出现错误（比如分配内存失败）则返回NGX_ERROR，否则返回NGX_OK。变量计算结果的状态可以通过ngx_http_variable_value_t的flags成员的值来了解（参考前文相关描述）。

set handler允许设置变量所指向的属性。例如，$limit_rate变量的set handler修改了请求的limit_rate成员的值：

```
...
{ ngx_string("limit_rate"), ngx_http_variable_request_set_size,
  ngx_http_variable_request_get_size,
  offsetof(ngx_http_request_t, limit_rate),
  NGX_HTTP_VAR_CHANGEABLE|NGX_HTTP_VAR_NOCACHEABLE, 0 },
...

static void
ngx_http_variable_request_set_size(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    ssize_t    s, *sp;
    ngx_str_t  val;

    val.len = v->len;
    val.data = v->data;

    s = ngx_parse_size(&val);

    if (s == NGX_ERROR) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "invalid size \"%V\"", &val);
        return;
    }

    sp = (ssize_t *) ((char *) r + data);

    *sp = s;

    return;
}
```

复杂值
-------

复杂值提供了一种简单的方法来计算一个包含有文本、变量以及文本变量组合等情况的表达式的值。

由ngx_http_compile_complex_value所表示的复杂值在配置阶段被编译到ngx_http_complex_value_t类型中，该编译的结果在运行阶段可以被用来计算表达式的值。

```
ngx_str_t                         *value;
ngx_http_complex_value_t           cv;
ngx_http_compile_complex_value_t   ccv;

value = cf->args->elts; /* directive arguments */

ngx_memzero(&ccv, sizeof(ngx_http_compile_complex_value_t));

ccv.cf = cf;
ccv.value = &value[1];
ccv.complex_value = &cv;
ccv.zero = 1;
ccv.conf_prefix = 1;

if (ngx_http_compile_complex_value(&ccv) != NGX_OK) {
    return NGX_CONF_ERROR;
}
```

这里，ccv里包含了全部初始化复杂值cv所需的参数：

* cf — 配置指针
* value — 待解析的字符串 (输入)
* complex_value — 编译后的值 (输出)
* zero — 是否对结果进行0结尾处理
* conf_prefix — 是否将结果带上配置前缀（nginx当前查找配置的目录）
* root_prefix — 是否将结果带上根前缀（通常是nginx的安装目录）

zero标记位在需要把结果传递给要求0结尾字符串的库时，非常有用，而前缀相关的标记位在处理文件名时很方便。

对于正确的编译，可以从cv.lengths成员获取到表达式中是否存在变量的情况。如果为NULL，则表示表达式中只是纯文本，所以没有必要将其保存成一个复杂值，使用简单的字符串就可以了。

ngx_http_set_complex_value_slot()可以在声明指令的时候对复杂值进行初始化。

在运行阶段，复杂值可以使用ngx_http_complex_value()函数来计算：

```
ngx_str_t  res;

if (ngx_http_complex_value(r, &cv, &res) != NGX_OK) {
    return NGX_ERROR;
}
```

给定请求r和之前编译的cv，该函数会对表达式的值进行急计算并将结果存放在res变量中。

请求重定向
---------

HTTP请求总是通过ngx_http_request_t结构体的loc_conf成员来绑定到某个location上。这意味着在任意时刻，任何模块都可以通过调用ngx_http_get_module_loc_conf(r, module)来获取到location的配置。在HTTP请求的生命周期内，其location可能会改变多次。初始时，default server的default location会被分配给HTTP请求。一旦这个请求切换到了另外一个不同的server（比如通过HTTP的"Host"头，或者通过SSL的SNI扩展），该server的default location也同样会分配给这个请求。接下来在NGX_HTTP_FIND_CONFIG_PHASE阶段中会重新为请求选择location。在这个阶段里，location的选择是基于请求的URI，在此server中全部的非命名location中查找得来的。ngx_http_rewrite_module模块也可能在NGX_HTTP_REWRITE_PHASE阶段对请求的URI进行修改，这样的话请求会重新发送回NGX_HTTP_FIND_CONFIG_PHASE阶段使用新的URI进行location匹配。

也可以在任意时候通过对ngx_http_internal_redirect(r, uri, args)和ngx_http_named_location(r, name)函数进行调用来实现将请求重定向到一个新的location。

ngx_http_internal_redirect(r, uri, args)函数修改请求的URI并且将请求发送回NGX_HTTP_SERVER_REWRITE_PHASE阶段。之后请求被分配到server默认的location上，然后在NGX_HTTP_FIND_CONFIG_PHASE阶段根据请求新的URI来选择location。

下面是一个同时带有新的请求参数的内部重定向的例子。

```
ngx_int_t
ngx_http_foo_redirect(ngx_http_request_t *r)
{
    ngx_str_t  uri, args;

    ngx_str_set(&uri, "/foo");
    ngx_str_set(&args, "bar=1");

    return ngx_http_internal_redirect(r, &uri, &args);
}
```

The function ngx_http_named_location(r, name) redirects a request to a named location. The name of the location is passed as the argument. The location is looked up among all named locations of the current server, after which the requests switches to the NGX_HTTP_REWRITE_PHASE phase.

ngx_http_named_location(r, name)函数将请求重定向到一个命名location。目标location的名称通过参数传递，并在当前server中的全部命名location中查找，接着请求会被发送到NGX_HTTP_REWRITE_PHASE阶段。

下面是一个将请求重定向到命名location @foo的例子：

```
ngx_int_t
ngx_http_foo_named_redirect(ngx_http_request_t *r)
{
    ngx_str_t  name;

    ngx_str_set(&name, "foo");

    return ngx_http_named_location(r, &name);
}
```

当ngx_http_internal_redirect(r, uri, args)和ngx_http_named_location(r, name)这两个函数被调用时，nginx模块可能已经向HTTP请求的ctx成员中存储了一些上下文。这些上下文在请求发生location切换之后可能会变得不一致。为了避免这种不一致性，所有的请求上下文会被这两个函数清除。

被重定向以及被重写的请求成为了内部请求进而可以访问内部location。内部请求的internal标记位被设置为真。

子请求
-----

子请求主要用来将一个请求的输出合并到另外一个请求中，很可能和其他数据混合。一个子请求看起来就像是一个普通的请求，但是和其父请求共享某些数据。具体来说，所有和客户端输入相关的数据都是共享的，因为子请求不从客户端接收任何额外的数据。子请求的请求结构中的parent成员保存了指向其父请求的指针，如果是main request则此成员为空。成员main存储了指向一组请求中main请求的指针。

子请求从NGX_HTTP_SERVER_REWRITE_PHASE阶段开始。它经历的其他阶段和普通请求相同，并基于其URI来分配location。

子请求的输出头总是被忽略。子请求的输出体通过ngx_http_postpone_filter插入到父请求产生的数据中的合适位置。

子请求和活动请求的概念相关。一个请求r被认为是活动的，如果c->data == r，c是表示nginx和客户端连接的对象。在任意时候，只有一组请求中的活动请求才允许将其输出缓冲发送给客户端。一个非活动请求仍然可以将其数据发送到过滤链中，但是这些数据不会通过ngx_http_postpone_filter过滤并且数据会一直保留在这个过滤器中，直到请求变成活动状态。下面是一些关于请求活动性的规则：

* 开始时，main请求是活动的
* 一个活动请求的第一个子请求在被创建之后立刻变为活动的
* 如果活动请求的子请求队列上的下一个请求之前的数据都已经发送完，则ngx_http_postpone_filter会将此请求激活
* 当一个请求结束了，它的父请求变为活动请求

一个子请求是用过调用ngx_http_subrequest(r, uri, args, psr, ps, flags)函数来创建的，其中r是父请求，uri和args分别是子请求的URI和请求参数，psr是一个输出参数，含有新创建的子请求的引用，ps是一个回调函数，用来在子请求结束的时候通知父请求，flags是子请求的创建标记位。有如下标记位可以使用：

* NGX_HTTP_SUBREQUEST_IN_MEMORY - 子请求的输出不需要发送给客户端，而是在内存中保留。此标记位只对代理子请求有效。在子请求结束后，它的输出会以ngx_buf_t类型存放在r->upstream->buffer中。
* NGX_HTTP_SUBREQUEST_WAITED - 子请求的done标记位会被设置，即使当其结束时处于非活动状态。这个标记位被SSI过滤器使用。
* NGX_HTTP_SUBREQUEST_CLONE - 子请求作为父请求的克隆来创建。如此创建的子请求将继承父请求的location并从父请求所在的阶段继续执行。

下面的例子中创建了一个URI为"/foo"的子请求。

```
ngx_int_t            rc;
ngx_str_t            uri;
ngx_http_request_t  *sr;

...

ngx_str_set(&uri, "/foo");

rc = ngx_http_subrequest(r, &uri, NULL, &sr, NULL, 0);
if (rc == NGX_ERROR) {
    /* error */
}
```

这个例子是将当前请求进行克隆并为子请求设置了一个结束回调函数。

```
ngx_int_t
ngx_http_foo_clone(ngx_http_request_t *r)
{
    ngx_http_request_t          *sr;
    ngx_http_post_subrequest_t  *ps;

    ps = ngx_palloc(r->pool, sizeof(ngx_http_post_subrequest_t));
    if (ps == NULL) {
        return NGX_ERROR;
    }

    ps->handler = ngx_http_foo_subrequest_done;
    ps->data = "foo";

    return ngx_http_subrequest(r, &r->uri, &r->args, &sr, ps,
                               NGX_HTTP_SUBREQUEST_CLONE);
}


ngx_int_t
ngx_http_foo_subrequest_done(ngx_http_request_t *r, void *data, ngx_int_t rc)
{
    char  *msg = (char *) data;

    ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                  "done subrequest r:%p msg:%s rc:%i", r, msg, rc);

    return rc;
}
```

子请求通常在body过滤器中创建。在这种情况下，子请求的输出可以被当成任意的显式请求输出处理。这意味着子请求的输出会在其他全部先于子请求创建的显式缓冲之后，以及在除此之外的任何缓冲之前，发送给客户端。这个顺序对于大型的子请求层次结构也同样有效。下面演示了将一个子请求插入到所有请求数据缓冲之后，但是在拥有last_buf的最后一个缓冲之前的例子。

```
ngx_int_t
ngx_http_foo_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_int_t                   rc;
    ngx_buf_t                  *b;
    ngx_uint_t                  last;
    ngx_chain_t                *cl, out;
    ngx_http_request_t         *sr;
    ngx_http_foo_filter_ctx_t  *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_foo_filter_module);
    if (ctx == NULL) {
        return ngx_http_next_body_filter(r, in);
    }

    last = 0;

    for (cl = in; cl; cl = cl->next) {
        if (cl->buf->last_buf) {
            cl->buf->last_buf = 0;
            cl->buf->last_in_chain = 1;
            cl->buf->sync = 1;
            last = 1;
        }
    }

    /* Output explicit output buffers */

    rc = ngx_http_next_body_filter(r, in);

    if (rc == NGX_ERROR || !last) {
        return rc;
    }

    /*
     * Create the subrequest.  The output of the subrequest
     * will automatically be sent after all preceding buffers,
     * but before the last_buf buffer passed later in this function.
     */

    if (ngx_http_subrequest(r, ctx->uri, NULL, &sr, NULL, 0) != NGX_OK) {
        return NGX_ERROR;
    }

    ngx_http_set_ctx(r, NULL, ngx_http_foo_filter_module);


    /* Output the final buffer with the last_buf flag */

    b = ngx_calloc_buf(r->pool);
    if (b == NULL) {
        return NGX_ERROR;
    }

    b->last_buf = 1;

    out.buf = b;
    out.next = NULL;

    return ngx_http_output_filter(r, &out);
}
```

一个子请求也可以为了输出数据之外的目的而创建。例如，ngx_http_auth_request_module在NGX_HTTP_ACCESS_PHASE阶段创建了一个子请求。为了在这个阶段禁止任何输出，子请求的header_only标志被设置。这可以避免子请求的body被发送到客户端。子请求的header无论如何都是被忽略的。子请求的结果可以通过回调handler来分析处理。

请求结束
-------

一个HTTP请求通过调用ngx_http_finalize_request(r, rc)来完成其生命周期。这通常是content handler在向过滤链发送完全部输出数据后执行的。在这个时候，数据有可能还没有全部发送到客户端，而是其中一部分依然缓存在过滤链的某处。如果是这样，ngx_http_finalize_request(r, rc)函数会自动注册一个特殊的handlerngx_http_writer(r)来完成数据的发送。一个请求也可能是因为产生了某种错误或者因为标准的HTTP响应码需要被返回给客户端，而被终结。

ngx_http_finalize_request(r, rc)函数接受如下的rc参数值：

* NGX_DONE - 快速结束。减少请求引用计数并且如果为0的话就销毁请求。和客户端的连接可能会被继续复用。
* NGX_ERROR, NGX_HTTP_REQUEST_TIME_OUT (408), NGX_HTTP_CLIENT_CLOSED_REQUEST (499) - 错误结束。尽可能快结束请求并关闭客户端连接。
* NGX_HTTP_CREATED (201), NGX_HTTP_NO_CONTENT (204), 大于或等于 NGX_HTTP_SPECIAL_RESPONSE (300) - 特殊响应结束。对这些值nginx要么发送默认代号响应页面给客户端，要么根据error_page location执行内部重定向（如果配置了的话）。
* 其它值被认为是成功结束，并且可能激活请求writer完成发送响应体。一旦body发送完毕，请求计数就会递减。如果到达0,则该请求会被销毁，但是客户端可能因为其它请求继续被使用着。如果计数大于0, 则该请求内还有未完成的活动，它们将在后面被继续完成。

请求体
------

为处理客户端请求体，nginx提供了两个函数：ngx_http_read_client_request_body(r, post_handler) 和 ngx_http_discard_request_body(r)。每一个函数读请求体并且设到 request_body 字段。第二个函数指示nginx丢弃（读和忽略）请求体。每个请求必须调用它们其中的一个。通常，这个在content阶段完成。

读或丢弃客户端请求体不能在子请求里。这个需要在主请求里完成。当一个子请求创建时，如果父请求已经在前面读了请求体，则子请求会继承父的request_body以便使用。

函数 ngx_http_read_client_request_body(r, post_handler) 开始读请求体的处理。当请求体完全读取后，post_handler 回调函数会被调用以继续处理请求。如果没有请求体或已读，则回调函数会立即被调用。函数 ngx_http_read_client_request_body(r, post_handler) 分配类型为ngx_http_request_body_t的request_body字段。该对象的bufs字段将结果保留为buffer chain。请求体可以保存在内存buffer，如果client_body_buffer_size不足于容纳整个在内存的body时，则保存在文件buffer。

以下例子读客户端请求体并返回它的大小。

```
ngx_int_t
ngx_http_foo_content_handler(ngx_http_request_t *r)
{
    ngx_int_t  rc;

    rc = ngx_http_read_client_request_body(r, ngx_http_foo_init);

    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        /* error */
        return rc;
    }

    return NGX_DONE;
}


void
ngx_http_foo_init(ngx_http_request_t *r)
{
    off_t         len;
    ngx_buf_t    *b;
    ngx_int_t     rc;
    ngx_chain_t  *in, out;

    if (r->request_body == NULL) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    len = 0;

    for (in = r->request_body->bufs; in; in = in->next) {
        len += ngx_buf_size(in->buf);
    }

    b = ngx_create_temp_buf(r->pool, NGX_OFF_T_LEN);
    if (b == NULL) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    b->last = ngx_sprintf(b->pos, "%O", len);
    b->last_buf = (r == r->main) ? 1: 0;
    b->last_in_chain = 1;

    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = b->last - b->pos;

    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        ngx_http_finalize_request(r, rc);
        return;
    }

    out.buf = b;
    out.next = NULL;

    rc = ngx_http_output_filter(r, &out);

    ngx_http_finalize_request(r, rc);
}
```

以下请求的字段会影响请求体的读取方式。

* request_body_in_single_buf - 将请求体读到单一内存buffer。
* request_body_in_file_only - 始终将请求体读到文件，即使适合内存缓冲区。
* request_body_in_persistent_file - 创建后不删除该文件。这样的文件可以被移到其它目录。
* request_body_in_clean_file - 当请求结束时删除该文件。当文件希望被移到其它目录，但由于某种原因没移走，这时该字段就派上用场了。
* request_body_file_group_access - 启用文件组权限。默认情况文件以0600权限被创建。当该标记设置时，0660权限就被用上了。
* request_body_file_log_level - 记录文件错误的日志级别。
* request_body_no_buffering - 不缓冲的读请求体。

当设置request_body_no_buffering这个标记，读请求体的非缓冲模式就开启了。这种模式下，调用完 ngx_http_read_client_request_body()之后，bufs链可能只保留请求体的一部份。要继续读下个部分，应该调用ngx_http_read_unbuffered_request_body(r) 函数。返回值为 NGX_AGAIN 并且设置了标记reading_body表明还有更多的数据可读。如果调用该函数后 bufs 是 NULL，则说明此该没有数据可读。当请求体下个部份可用时，请求回调用函数 read_event_handler 回被调用。

响应
----

nginx里的HTTP响应是通过发送响应头和接着可选的响应体产生的。两者被传进filter链里并且最终写到客户端socket。一个nginx模块可以安装它的handler到header或body filter里，并且处理来自上一个handler的输出。

响应头
------

通过函数 ngx_http_send_header(r) 发送输出头。在调用这个函数之前，r->headers_out 必须包含所有被用来发送HTTP响应头的数据。r->headers_out的status字段通常是需要设置的。如果该响应状态码指示响应体应该接着头部，content_length_n 也可以设置。该值默认是-1，表示响应体大小是未知的。这种情况下，就会用到chunked传输。想输出任意的头部，需要加到头部列表里。

```
static ngx_int_t
ngx_http_foo_content_handler(ngx_http_request_t *r)
{
    ngx_int_t         rc;
    ngx_table_elt_t  *h;

    /* send header */

    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = 3;

    /* X-Foo: foo */

    h = ngx_list_push(&r->headers_out.headers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    h->hash = 1;
    ngx_str_set(&h->key, "X-Foo");
    ngx_str_set(&h->value, "foo");

    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }

    /* send body */

    ...
}
```

头部过滤
--------------

函数 ngx_http_send_header(r) 通过调用首个头部filter handler ngx_http_top_header_filter 执行头部filter链。它假设所有的header heandle会调用链里的下一个hanndler直到最后一个handler ngx_http_header_filter(r)。 这个最后的handler构造了基于 r->headers_out 的 HTTP 响应并且将它传给 ngx_http_writer_filter 以作输出。

要将一个handler添加到 header filter 链, 需要在配置阶段将它的地址保存在 ngx_http_top_header_filter 这个全局变量。前一个handler的地址通常保存在模块里的一个静态变量，并且在退出前由新加入的handler调用。

以下是个header filter模块的例子，对每个状态是200的输出都加个 "X-Foo: foo" 头部信息。

```
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>


static ngx_int_t ngx_http_foo_header_filter(ngx_http_request_t *r);
static ngx_int_t ngx_http_foo_header_filter_init(ngx_conf_t *cf);


static ngx_http_module_t  ngx_http_foo_header_filter_module_ctx = {
    NULL,                                   /* preconfiguration */
    ngx_http_foo_header_filter_init,        /* postconfiguration */

    NULL,                                   /* create main configuration */
    NULL,                                   /* init main configuration */

    NULL,                                   /* create server configuration */
    NULL,                                   /* merge server configuration */

    NULL,                                   /* create location configuration */
    NULL                                    /* merge location configuration */
};


ngx_module_t  ngx_http_foo_header_filter_module = {
    NGX_MODULE_V1,
    &ngx_http_foo_header_filter_module_ctx, /* module context */
    NULL,                                   /* module directives */
    NGX_HTTP_MODULE,                        /* module type */
    NULL,                                   /* init master */
    NULL,                                   /* init module */
    NULL,                                   /* init process */
    NULL,                                   /* init thread */
    NULL,                                   /* exit thread */
    NULL,                                   /* exit process */
    NULL,                                   /* exit master */
    NGX_MODULE_V1_PADDING
};


static ngx_http_output_header_filter_pt  ngx_http_next_header_filter;


static ngx_int_t
ngx_http_foo_header_filter(ngx_http_request_t *r)
{
    ngx_table_elt_t  *h;

    /*
     * The filter handler adds "X-Foo: foo" header
     * to every HTTP 200 response
     */

    if (r->headers_out.status != NGX_HTTP_OK) {
        return ngx_http_next_header_filter(r);
    }

    h = ngx_list_push(&r->headers_out.headers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    h->hash = 1;
    ngx_str_set(&h->key, "X-Foo");
    ngx_str_set(&h->value, "foo");

    return ngx_http_next_header_filter(r);
}


static ngx_int_t
ngx_http_foo_header_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_foo_header_filter;

    return NGX_OK;
}
```

响应体
------

通过函数 ngx_http_output_filter(r, cl) 发响应体。该函数能被调用多次。每次它会发送作为buffer链的响应体的一部份。最后的body buffer应该有设置last_buf标记。

以下例子产生一个完整的HTTP输出 "foo" 作为响应体。为了让这个例子不止能在主请求运行，也在子请求能运行。输出的最后buffer会设置 last_in_chain 标记。标记 last_buf 只会对主请求设置，因为子请求的最后buffer不会作为整个输出的结束。

```
static ngx_int_t
ngx_http_bar_content_handler(ngx_http_request_t *r)
{
    ngx_int_t     rc;
    ngx_buf_t    *b;
    ngx_chain_t   out;

    /* send header */

    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = 3;

    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }

    /* send body */

    b = ngx_calloc_buf(r->pool);
    if (b == NULL) {
        return NGX_ERROR;
    }

    b->last_buf = (r == r->main) ? 1: 0;
    b->last_in_chain = 1;

    b->memory = 1;

    b->pos = (u_char *) "foo";
    b->last = b->pos + 3;

    out.buf = b;
    out.next = NULL;

    return ngx_http_output_filter(r, &out);
}
```

响应体过滤
------------

函数 ngx_http_output_filter(r, cl) 通过调用首个body filter handler ngx_http_top_body_filter 执行响应体过滤链。它假定每个body handler会调用链里的下一个handler直到最后的handler ngx_http_write_filter(r, cl) 被调用。

body filter handler会接收一个buffer链。这个handler会处理 buffers 并且传可能新的chain给下个handler。值得注意的是，传入的ngx_chain_t链接属于调用者。它们不用被复用或者改变。当handler完成后，调用者可以用它的输出链来跟踪其发送的buffer。如果想保存buffer chain或替换一些继续要发送的buffer，该handler应该分配它自己的链。

以下是一个简单的计算响应体大小的body模块。结果作为 $counter 变量可以被用在 access 日志。

```
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>


typedef struct {
    off_t  count;
} ngx_http_counter_filter_ctx_t;


static ngx_int_t ngx_http_counter_body_filter(ngx_http_request_t *r,
    ngx_chain_t *in);
static ngx_int_t ngx_http_counter_variable(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data);
static ngx_int_t ngx_http_counter_add_variables(ngx_conf_t *cf);
static ngx_int_t ngx_http_counter_filter_init(ngx_conf_t *cf);


static ngx_http_module_t  ngx_http_counter_filter_module_ctx = {
    ngx_http_counter_add_variables,        /* preconfiguration */
    ngx_http_counter_filter_init,          /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    NULL,                                  /* create location configuration */
    NULL                                   /* merge location configuration */
};


ngx_module_t  ngx_http_counter_filter_module = {
    NGX_MODULE_V1,
    &ngx_http_counter_filter_module_ctx,   /* module context */
    NULL,                                  /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};


static ngx_http_output_body_filter_pt  ngx_http_next_body_filter;

static ngx_str_t  ngx_http_counter_name = ngx_string("counter");


static ngx_int_t
ngx_http_counter_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_chain_t                    *cl;
    ngx_http_counter_filter_ctx_t  *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_counter_filter_module);
    if (ctx == NULL) {
        ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_counter_filter_ctx_t));
        if (ctx == NULL) {
            return NGX_ERROR;
        }

        ngx_http_set_ctx(r, ctx, ngx_http_counter_filter_module);
    }

    for (cl = in; cl; cl = cl->next) {
        ctx->count += ngx_buf_size(cl->buf);
    }

    return ngx_http_next_body_filter(r, in);
}


static ngx_int_t
ngx_http_counter_variable(ngx_http_request_t *r, ngx_http_variable_value_t *v,
    uintptr_t data)
{
    u_char                         *p;
    ngx_http_counter_filter_ctx_t  *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_counter_filter_module);
    if (ctx == NULL) {
        v->not_found = 1;
        return NGX_OK;
    }

    p = ngx_pnalloc(r->pool, NGX_OFF_T_LEN);
    if (p == NULL) {
        return NGX_ERROR;
    }

    v->data = p;
    v->len = ngx_sprintf(p, "%O", ctx->count) - p;
    v->valid = 1;
    v->no_cacheable = 0;
    v->not_found = 0;

    return NGX_OK;
}


static ngx_int_t
ngx_http_counter_add_variables(ngx_conf_t *cf)
{
    ngx_http_variable_t  *var;

    var = ngx_http_add_variable(cf, &ngx_http_counter_name, 0);
    if (var == NULL) {
        return NGX_ERROR;
    }

    var->get_handler = ngx_http_counter_variable;

    return NGX_OK;
}


static ngx_int_t
ngx_http_counter_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_counter_body_filter;

    return NGX_OK;
}
```

构建过滤模块
------------

当写一个body或header过滤模块是，要特别注意filter的顺序。已经有一些已经注册的标准nginx模块。注册一个filter模块到相对其它模块的正确位置是很重要的。通常filter会在模块自己的postconfiguration handler里注册。filter的调用顺序跟它们的注册时的顺序刚好相反。

nginx给第三方模块提供了个特殊的槽口 HTTP_AUX_FILTER_MODULES。想在这个插槽注册一个filter模块，模块的配置里应该将 ngx_module_type 变量设置值为 HTTP_AUX_FILTER。

以下例子显示一个filter模块的配置文件，并且假设只有一个源文件 ngx_http_foo_filter_module.c。

```
ngx_module_type=HTTP_AUX_FILTER
ngx_module_name=ngx_http_foo_filter_module
ngx_module_srcs="$ngx_addon_dir/ngx_http_foo_filter_module.c"

. auto/module
```

缓冲复用
--------

当处理或更改缓冲区流时，经常需要复用已分配的buffer。nginx代码里比较标准通用的处理方式是保留两个buffer链：free and busy。 free 链保留所有空闲的 buffer。这些buffer可以拿来复用。busy 链保存所有当前模块发送的buffer，但仍然被其它的filter handler使用。如果它的大小大于0,则认为该buffer还在使用。通常一个buffer被一个filter消费时，它的pos（或file_pos对文件buffer而言）会移向last （或file_pos对文件buffer而言）。一旦整个buffer被完全消费完，它就可以复用了。为将新空闲的buffer更新到空闲chain，需要完整的遍历busy链，并将大小为0的buffer移到free的首部。这种操作很常见，所以有个特殊的函数 ngx_chain_update_chains(free, busy, out, tag) 专门处理这个。这个函数追加output chain到busy，并且将空闲的buffer从busy移到free。只有匹配tag的buffer才能复用。这样就让一个模块只能复用它自己分配的buffer。

以下例子为每个新进的buffer加入字符串 "foo"。该模块尽可能的复用这些新分配的buffer。注意：为了让该例子运行的没问题，需要安装header filter，并且将 content_length_n 设置为-1 （这节上面有提到）。

```
typedef struct {
    ngx_chain_t  *free;
    ngx_chain_t  *busy;
}  ngx_http_foo_filter_ctx_t;


ngx_int_t
ngx_http_foo_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_int_t                   rc;
    ngx_buf_t                  *b;
    ngx_chain_t                *cl, *tl, *out, **ll;
    ngx_http_foo_filter_ctx_t  *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_foo_filter_module);
    if (ctx == NULL) {
        ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_foo_filter_ctx_t));
        if (ctx == NULL) {
            return NGX_ERROR;
        }

        ngx_http_set_ctx(r, ctx, ngx_http_foo_filter_module);
    }

    /* create a new chain "out" from "in" with all the changes */

    ll = &out;

    for (cl = in; cl; cl = cl->next) {

        /* append "foo" in a reused buffer if possible */

        tl = ngx_chain_get_free_buf(r->pool, &ctx->free);
        if (tl == NULL) {
            return NGX_ERROR;
        }

        b = tl->buf;
        b->tag = (ngx_buf_tag_t) &ngx_http_foo_filter_module;
        b->memory = 1;
        b->pos = (u_char *) "foo";
        b->last = b->pos + 3;

        *ll = tl;
        ll = &tl->next;

        /* append the next incoming buffer */

        tl = ngx_alloc_chain_link(r->pool);
        if (tl == NULL) {
            return NGX_ERROR;
        }

        tl->buf = cl->buf;
        *ll = tl;
        ll = &tl->next;
    }

    *ll = NULL;

    /* send the new chain */

    rc = ngx_http_next_body_filter(r, out);

    /* update "busy" and "free" chains for reuse */

    ngx_chain_update_chains(r->pool, &ctx->free, &ctx->busy, &out,
                            (ngx_buf_tag_t) &ngx_http_foo_filter_module);

    return rc;
}
```

负载均衡
-------
ngx_http_upstream_module提供了向远程服务器发送HTTP请求的基本功能。其他具体的协议模块，例如HTTP或FastCDI，都会使用这个功能。该模块同时还提供了可以定制负载均衡算法的接口并默认实现了round-robin（轮询）算法

例如，提供其他的负载均衡算法的模块有least_conn和hash这些。需要注意的是，这些模块实际上是作为upstream模块的扩展而实现的，他们之间共享了大量的代码，比如对于服务器组的表示。keepalive模块是另外一个例子，这是一个独立的模块，扩展了upstream的功能。

ngx_http_upstream_module可以通过在配置文件中配置upstream块来显式配置，或者通过使用可以接受URL作为参数的指令来隐式开启，比如proxy_pass这种指令。只有显示的配置才能选择负载均衡算法。upstream模块有自己的指令上下文NGX_HTTP_UPS_CONF。相关结构体定义如下：

```
struct ngx_http_upstream_srv_conf_s {
    ngx_http_upstream_peer_t         peer;
    void                           **srv_conf;

    ngx_array_t                     *servers;  /* ngx_http_upstream_server_t */

    ngx_uint_t                       flags;
    ngx_str_t                        host;
    u_char                          *file_name;
    ngx_uint_t                       line;
    in_port_t                        port;
    ngx_uint_t                       no_port;  /* unsigned no_port:1 */

#if (NGX_HTTP_UPSTREAM_ZONE)
    ngx_shm_zone_t                  *shm_zone;
#endif
};
```

* srv_conf — upstream模块的配置上下文
* servers — ngx_http_upstream_server_t的数组，存放的是对upstream块中一组server指令解析的配置
* flags — 指定特定负载均衡算法支持哪些特性（通过server指令的参数配置）的标记位。
   * NGX_HTTP_UPSTREAM_CREATE — 用来区分显式定义的upstream和通过proxy_pass类型指令(FastCGI, SCGI等)隐式创建的upstream
   * NGX_HTTP_UPSTREAM_WEIGHT — 支持“weight”
   * NGX_HTTP_UPSTREAM_MAX_FAILS — 支持“max_fails”
   * NGX_HTTP_UPSTREAM_FAIL_TIMEOUT — 支持“fail_timeout”
   * NGX_HTTP_UPSTREAM_DOWN — 支持“down”
   * NGX_HTTP_UPSTREAM_BACKUP — 支持“backup”
   * NGX_HTTP_UPSTREAM_MAX_CONNS — 支持“max_conns”
* host — upstream的名字
* file_name, line — 配置文件名字以及upstream块所在行
* port and no_port — 显式upstream未使用
* shm_zone — 此upstream使用的共享内存
* peer — 存放用来初始化upstream配置通用方法的对象：

```
typedef struct {
    ngx_http_upstream_init_pt        init_upstream;
    ngx_http_upstream_init_peer_pt   init;
    void                            *data;
} ngx_http_upstream_peer_t;
```

实现负载均衡算法的模块必须设置这些方法并初始化私有数据。
如果init_upstream在配置阶段没有初始化，ngx_http_upstream_module会将其默认设置成ngx_http_upstream_init_round_robin。

   * init_upstream(cf, us) — 配置阶段方法，用于初始化一组服务器并初始化init()方法。一个典型的负载均衡模块使用upstream块中的一组服务器来创建某种有效的数据结构并在data成员中存放自身的配置。
   * init(r, us) — 初始化用于每个请求的ngx_http_upstream_peer_t.peer (不要和之前用于每个upstream的ngx_http_upstream_srv_conf_t.peer搞混了)结构，该结构用于进行负载均衡。该结构会作为所有处理服务器选择的回调函数的data参数传递。

当nginx需要将请求转给其他服务器进行处理时，它会调用配置好的负载均衡算法来选择一个地址，并发起连接。选择算法是从ngx_http_upstream_peer_t.peer对象中获取的，该对象的类型是ngx_peer_connection_t：

```
struct ngx_peer_connection_s {
    [...]

    struct sockaddr                 *sockaddr;
    socklen_t                        socklen;
    ngx_str_t                       *name;

    ngx_uint_t                       tries;

    ngx_event_get_peer_pt            get;
    ngx_event_free_peer_pt           free;
    ngx_event_notify_peer_pt         notify;
    void                            *data;

#if (NGX_SSL || NGX_COMPAT)
    ngx_event_set_peer_session_pt    set_session;
    ngx_event_save_peer_session_pt   save_session;
#endif

    [..]
};
```

这个结构体有如下成员：

* sockaddr, socklen, name — 待连接的upstream服务器的地址；此为负载均衡算法的输出参数
* data — 每请求的负载均衡算法所需数据；记录选择算法的状态并且通常会含有指向upstream配置的指针。此data会被作为参数传递给所有处理服务器选择的函数（见下文）
* tries — 连接upstream服务器的重试次数
* get, free, notify, set_session, and save_session - 负载均衡算法模块的方法，详细见下文

所有的方法至少接受两个参数：peer连接对象pc以及由ngx_http_upstream_srv_conf_t.peer.init()创建的data参数。注意，一般来说，由于负载均衡算法的”chaining”，这个data和pc.data是不同的，

* get(pc, data) — 当upstream模块需要将请求发送给一个服务器而需要知道服务器地址的时候，该方法会被调用。该方法负责填写ngx_peer_connection_t结构的sockaddr，socklen和name成员。返回值有如下几种：
   * NGX_OK — 服务器已选择
   * NGX_ERROR — 发生了内部错误
   * NGX_BUSY — 当前没有可用服务器。有多种原因会导致这个情况的发生，例如：动态服务器组为空，全部服务器均为失败状态，全部服务器已经达到最大连接数或者其他类似情况。
   * NGX_DONE — keepalive模块用这个返回值来说明底层连接进行了复用，因此不需要和upstream服务器间创建一条新连接。
* free(pc, data, state) — 当upstream模块同某个upstream服务器通信结束后，调用此方法。state参数指示了upstream连接的完成状态，是一个bitmask，可以被设置成这些值：NGX_PEER_FAILED - 失败，NGX_PEER_NEXT - 403和404的特殊情况，不作为失败对待，NGX_PEER_KEEPALIVE。此外，尝试次数也在这个方法递减。
* notify(pc, data, type) — 开源版本中未使用。
* set_session(pc, data)和save_session(pc, data) — SSL相关方法，用于缓存同upstream服务器间的SSL会话，由round-robin负载均衡算法实现。

