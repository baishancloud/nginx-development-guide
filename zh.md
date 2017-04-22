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
* [Buffer](#buffer)
* [Networking](#networking)
    * [Connection](#connection)
* [事件](#事件)
    * [事件](#事件)
    * [I/O事件](#I/O事件)
    * [定时器事件](#定时器事件)
    * [已加事件](#已加事件)
    * [遍历事件](#遍历事件)
* [进程](#进程)
* [模块](#模块)
    * [添加新模块](#添加新模块)
    * [核心模块](#核心模块)
    * [配置指令](#配置指令)
* [HTTP](#hTTP)
    * [Connection](#connection)
    * [Request](#request)
    * [配置](#配置)
    * [Phases](#phases)
    * [Variables](#variables)
    * [Complex values](#complex-values)
    * [Request redirection](#request-redirection)
    * [Subrequests](#subrequests)
    * [Request finalization](#request-finalization)
    * [Request body](#request-body)
    * [Response](#response)
    * [Response body](#response-body)
    * [Body filters](#body-filters)
    * [Building filter modules](#building-filter-modules)
    * [Buffer reuse](#buffer-reuse)
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
* ngx_slrintf(buf, last, fmt, ...)
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

Initially, a list must be initialized by calling ngx_list_init(list, pool, n, size) or created by calling ngx_list_create(pool, n, size). Both functions receive the size of a single item and a number of items per list part. The ngx_list_push(list) function is used to add an item to the list. Iterating over the items is done by direct accessing the list fields, as seen in the example:

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

Queue
-----

Queue in nginx is an intrusive doubly linked list, with each node defined as follows:

```
typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

The head queue node is not linked with any data. Before using, the list head should be initialized with ngx_queue_init(q) call. Queues support the following operations:

* ngx_queue_insert_head(h, x), ngx_queue_insert_tail(h, x) — insert a new node
* ngx_queue_remove(x) — remove a queue node
* ngx_queue_split(h, q, n) — split a queue at a node, queue tail is returned in a separate queue
* ngx_queue_add(h, n) — add second queue to the first queue
* ngx_queue_head(h), ngx_queue_last(h) — get first or last queue node
* ngx_queue_sentinel(h) - get a queue sentinel object to end iteration at
* ngx_queue_data(q, type, link) — get reference to the beginning of a queue node data structure, consid ering the queue field offset in it

Example:

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

Red-Black tree
--------------

The src/core/ngx_rbtree.h header file provides access to the effective implementation of red-black tree
s.

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

To deal with a tree as a whole, you need two nodes: root and sentinel. Typically, they are added to som
e custom structure, thus allowing to organize your data into a tree which leaves contain a link to or e
mbed your data.

To initialize a tree:

```
my_tree_t  root;

ngx_rbtree_init(&root.rbtree, &root.sentinel, insert_value_function);
```

The insert_value_function is a function that is responsible for traversing the tree and inserting new v
alues into correct place. For example, the ngx_str_rbtree_insert_value functions is designed to deal wi
th ngx_str_t type.

```
void ngx_str_rbtree_insert_value(ngx_rbtree_node_t *temp,
                                 ngx_rbtree_node_t *node,
                                 ngx_rbtree_node_t *sentinel)
```

Its arguments are pointers to a root node of an insertion, newly created node to be added, and a tree s
entinel.

The traversal is pretty straightforward and can be demonstrated with the following lookup function patt
ern:

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

The compare() is a classic comparator function returning value less, equal or greater than zero. To spe
ed up lookups and avoid comparing user objects that can be big, integer hash field is used.

To add a node to a tree, allocate a new node, initialize it and call ngx_rbtree_insert():

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

Hash
----

Hash table functions are declared in src/core/ngx_hash.h. Exact and wildcard matching is supported. The
 latter requires extra setup and is described in a separate section below.

To initialize a hash, one needs to know the number of elements in advance, so that nginx can build the
hash optimally. Two parameters that need to be configured are max_size and bucket_size. The details of
setting up these are provided in a separate document. Usually, these two parameters are configurable by
 user. Hash initialization settings are stored as the ngx_hash_init_t type, and the hash itself is ngx_
hash_t:

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

The key is a pointer to a function that creates hash integer key from a string. Two generic functions a
re provided: ngx_hash_key(data, len) and ngx_hash_key_lc(data, len). The latter converts a string to lo
wercase and thus requires the passed string to be writable. If this is not true, NGX_HASH_READONLY_KEY
flag may be passed to the function, initializing array keys (see below).

The hash keys are stored in ngx_hash_keys_arrays_t and are initialized with ngx_hash_keys_array_init(ar
r, type):

```
ngx_hash_keys_arrays_t  foo_keys;

foo_keys.pool = cf->pool;
foo_keys.temp_pool = cf->temp_pool;

ngx_hash_keys_array_init(&foo_keys, NGX_HASH_SMALL);
```

The second parameter can be either NGX_HASH_SMALL or NGX_HASH_LARGE and controls the amount of prealloc
ated resources for the hash. If you expect the hash to contain thousands elements, use NGX_HASH_LARGE.

The ngx_hash_add_key(keys_array, key, value, flags) function is used to insert keys into hash keys arra
y;

```
ngx_str_t k1 = ngx_string("key1");
ngx_str_t k2 = ngx_string("key2");

ngx_hash_add_key(&foo_keys, &k1, &my_data_ptr_1, NGX_HASH_READONLY_KEY);
ngx_hash_add_key(&foo_keys, &k2, &my_data_ptr_2, NGX_HASH_READONLY_KEY);
```

Now, the hash table may be built using the call to ngx_hash_init(hinit, key_names, nelts):

```
ngx_hash_init(&hash, foo_keys.keys.elts, foo_keys.keys.nelts);
```

This may fail, if max_size or bucket_size parameters are not big enough. When the hash is built, ngx_ha
sh_find(hash, key, name, len) function may be used to look up elements:

```
my_data_t   *data;
ngx_uint_t   key;

key = ngx_hash_key(k1.data, k1.len);

data = ngx_hash_find(&foo_hash, key, k1.data, k1.len);
if (data == NULL) {
    /* key not found */
}
```

Wildcard matching
-----------------

To create a hash that works with wildcards, ngx_hash_combined_t type is used. It includes the hash type
 described above and has two additional keys arrays: dns_wc_head and dns_wc_tail. The initialization of
 basic properties is done similarly to a usual hash:

```
ngx_hash_init_t      hash
ngx_hash_combined_t  foo_hash;

hash.hash = &foo_hash.hash;
hash.key = ...;
```

It is possible to add wildcard keys using the NGX_HASH_WILDCARD_KEY flag:

```
/* k1 = ".example.org"; */
/* k2 = "foo.*";        */
ngx_hash_add_key(&foo_keys, &k1, &data1, NGX_HASH_WILDCARD_KEY);
ngx_hash_add_key(&foo_keys, &k2, &data2, NGX_HASH_WILDCARD_KEY);
```

The function recognizes wildcards and adds keys into corresponding arrays. Please refer to the map modu
le documentation for the description of the wildcard syntax and matching algorithm.

Depending on the contents of added keys, you may need to initialize up to three keys arrays: one for ex
act matching (described above), and two for matching starting from head or tail of a string:

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

The keys array needs to be sorted, and initialization results must be added to the combined hash. The i
nitialization of dns_wc_tail array is done similarly.

The lookup in a combined hash is handled by the ngx_hash_find_combined(chash, key, name, len):

```
/* key = "bar.example.org"; — will match ".example.org" */
/* key = "foo.example.com"; — will match "foo.*"        */

hkey = ngx_hash_key(key.data, key.len);
res = ngx_hash_find_combined(&foo_hash, hkey, key.data, key.len);
```

Memory management
=================

Heap
----

To allocate memory from system heap, the following functions are provided by nginx:

* ngx_alloc(size, log) — allocate memory from system heap. This is a wrapper around malloc() with loggi
ng support. Allocation error and debugging information is logged to log
ngx_calloc(size, log) — same as ngx_alloc(), but memory is filled with zeroes after allocation
* ngx_memalign(alignment, size, log) — allocate aligned memory from system heap. This is a wrapper arou
nd posix_memalign() on those platforms which provide it. Otherwise implementation falls back to ngx_all
oc() which provides maximum alignment
ngx_free(p) — free allocated memory. This is a wrapper around free()

Pool
----

Most nginx allocations are done in pools. Memory allocated in an nginx pool is freed automatically when
 the pool in destroyed. This provides good allocation performance and makes memory control easy.

A pool internally allocates objects in continuous blocks of memory. Once a block is full, a new one is
allocated and added to the pool memory block list. When a large allocation is requested which does not
fit into a block, such allocation is forwarded to the system allocator and the returned pointer is stor
ed in the pool for further deallocation.

Nginx pool has the type ngx_pool_t. The following operations are supported:

* ngx_create_pool(size, log) — create a pool with given block size. The pool object returned is allocat
ed in the pool as well.
* ngx_destroy_pool(pool) — free all pool memory, including the pool object itself.
* ngx_palloc(pool, size) — allocate aligned memory from pool
* ngx_pcalloc(pool, size) — allocated aligned memory from pool and fill it with zeroes
* ngx_pnalloc(pool, size) — allocate unaligned memory from pool. Mostly used for allocating strings
* ngx_pfree(pool, p) — free memory, previously allocated in the pool. Only allocations, forwarded to th
e system allocator, can be freed.

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

Since chain links ngx_chain_t are actively used in nginx, nginx pool provides a way to reuse them. The
chain field of ngx_pool_t keeps a list of previously allocated links ready for reuse. For efficient all
ocation of a chain link in a pool, the function ngx_alloc_chain_link(pool) should be used. This functio
n looks up a free chain link in the pool list and only if it's empty allocates a new one. To free a lin
k ngx_free_chain(pool, cl) should be called.

Cleanup handlers can be registered in a pool. Cleanup handler is a callback with an argument which is c
alled when pool is destroyed. Pool is usually tied with a specific nginx object (like HTTP request) and
 destroyed in the end of that object’s lifetime, releasing the object itself. Registering a pool cleanu
p is a convenient way to release resources, close file descriptors or make final adjustments to shared
data, associated with the main object.

A pool cleanup is registered by calling ngx_pool_cleanup_add(pool, size) which returns ngx_pool_cleanup
_t pointer to be filled by the caller. The size argument allows allocating context for the cleanup hand
ler.

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

Shared memory
-------------

Shared memory is used by nginx to share common data between processes. Function ngx_shared_memory_add(c
f, name, size, tag) adds a new shared memory entry ngx_shm_zone_t to the cycle. The function receives n
ame and size of the zone. Each shared zone must have a unique name. If a shared zone entry with the pro
vided name exists, the old zone entry is reused, if its tag value matches too. Mismatched tag is consid
ered an error. Usually, the address of the module structure is passed as tag, making it possible to reu
se shared zones by name within one nginx module.

The shared memory entry structure ngx_shm_zone_t has the following fields:

* init — initialization callback, called after shared zone is mapped to actual memory
* data — data context, used to pass arbitrary data to the init callback
* noreuse — flag, disabling shared zone reuse from the old cycle
* tag — shared zone tag
* shm — platform-specific object of type ngx_shm_t, having at least the following fields:
    * addr — mapped shared memory address, initially NULL
    * size — shared memory size
    * name — shared memory name
    * log — shared memory log
    * exists — flag, showing that shared memory was inherited from the master process (Windows-specific
)

Shared zone entries are mapped to actual memory in ngx_init_cycle() after configuration is parsed. On P
OSIX systems, mmap() syscall is used to create shared anonymous mapping. On Windows, CreateFileMapping(
)/MapViewOfFileEx() pair is used.

For allocating in shared memory, nginx provides slab pool ngx_slab_pool_t. In each nginx shared zone, a
 slab pool is automatically created for allocating memory in that zone. The pool is located in the begi
nning of the shared zone and can be accessed by the expression (ngx_slab_pool_t *) shm_zone->shm.addr.
Allocation in shared zone is done by calling one of the functions ngx_slab_alloc(pool, size)/ngx_slab_c
alloc(pool, size). Memory is freed by calling ngx_slab_free(pool, p).

Slab pool divides all shared zone into pages. Each page is used for allocating objects of the same size
. Only the sizes which are powers of 2, and not less than 8, are considered. Other sizes are rounded up
 to one of these values. For each page, a bitmask is kept, showing which blocks within that page are in
 use and which are free for allocation. For sizes greater than half-page (usually, 2048 bytes), allocat
ion is done by entire pages.

To protect data in shared memory from concurrent access, mutex is available in the mutex field of ngx_s
lab_pool_t. The mutex is used by the slab pool while allocating and freeing memory. However, it can be
used to protect any other user data structures, allocated in the shared zone. Locking is done by callin
g ngx_shmtx_lock(&shpool->mutex), unlocking is done by calling ngx_shmtx_unlock(&shpool->mutex).

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

Buffer
======

For input/output operations, nginx provides the buffer type ngx_buf_t. Normally, it's used to hold data
 to be written to a destination or read from a source. Buffer can reference data in memory and in file.
 Technically it's possible that a buffer references both at the same time. Memory for the buffer is all
ocated separately and is not related to the buffer structure ngx_buf_t.

The structure ngx_buf_t has the following fields:

* start, end — the boundaries of memory block, allocated for the buffer
* pos, last — memory buffer boundaries, normally a subrange of start .. end
* file_pos, file_last — file buffer boundaries, these are offsets from the beginning of the file
* tag — unique value, used to distinguish buffers, created by different nginx module, usually, for the
purpose of buffer reuse
* file — file object
* temporary — flag, meaning that the buffer references writable memory
* memory — flag, meaning that the buffer references read-only memory
* in_file — flag, meaning that current buffer references data in a file
* flush — flag, meaning that all data prior to this buffer should be flushed
* recycled — flag, meaning that the buffer can be reused and should be consumed as soon as possible
* sync — flag, meaning that the buffer carries no data or special signal like flush or last_buf. Normal
ly, such buffers are considered an error by nginx. This flags allows skipping the error checks
* last_buf — flag, meaning that current buffer is the last in output
* last_in_chain — flag, meaning that there's no more data buffers in a (sub)request
* shadow — reference to another buffer, related to the current buffer. Usually current buffer uses data
 from the shadow buffer. Once current buffer is consumed, the shadow buffer should normally also be mar
ked as consumed
* last_shadow — flag, meaning that current buffer is the last buffer, referencing a particular shadow b
uffer
* temp_file — flag, meaning that the buffer is in a temporary file

For input and output buffers are linked in chains. Chain is a sequence of chain links ngx_chain_t, defi
ned as follows:

```
typedef struct ngx_chain_s  ngx_chain_t;

struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};
```

Each chain link keeps a reference to its buffer and a reference to the next chain link.

Example of using buffers and chains:

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

Networking
==========

Connection
----------

Connection type ngx_connection_t is a wrapper around a socket descriptor. Some of the structure fields are:

* fd — socket descriptor
* data — arbitrary connection context. Normally, a pointer to a higher level object, built on top of the connection, like HTTP request or Stream session
* read, write — read and write events for the connection
* recv, send, recv_chain, send_chain — I/O operations for the connection
* pool — connection pool
* log — connection log
* sockaddr, socklen, addr_text — remote socket address in binary and text forms
* local_sockaddr, local_socklen — local socket address in binary form. Initially, these fields are empty. Function ngx_connection_local_sockaddr() should be used to get socket local address
* proxy_protocol_addr, proxy_protocol_port - PROXY protocol client address and port, if PROXY protocol is enabled for the connection
* ssl — nginx connection SSL context
* reusable — flag, meaning, that the connection is at the state, when it can be reused
* close — flag, meaning, that the connection is being reused and should be closed

An nginx connection can transparently encapsulate SSL layer. In this case the connection ssl field holds a pointer to an ngx_ssl_connection_t structure, keeping all SSL-related data for the connection, including SSL_CTX and SSL. The handlers recv, send, recv_chain, send_chain are set as well to SSL functions.

The number of connections per nginx worker is limited by the worker_connections value. All connection structures are pre-created when a worker starts and stored in the connections field of the cycle object. To reach out for a connection structure, ngx_get_connection(s, log) function is used. The function receives a socket descriptor s which needs to be wrapped in a connection structure.

Since the number of connections per worker is limited, nginx provides a way to grab connections which are currently in use. To enable or disable reuse of a connection, function ngx_reusable_connection(c, reusable) is called. Calling ngx_reusable_connection(c, 1) sets the reuse flag of the connection structure and inserts the connection in the reusable_connections_queue of the cycle. Whenever ngx_get_connection() finds out there are no available connections in the free_connections list of the cycle, it calls ngx_drain_connections() to release a specific number of reusable connections. For each such connection, the close flag is set and its read handler is called which is supposed to free the connection by calling ngx_close_connection(c) and make it available for reuse. To exit the state when a connection can be reused ngx_reusable_connection(c, 0) is called. An example of reusable connections in nginx is HTTP client connections which are marked as reusable until some data is received from the client.


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

Connection
----------

Each client HTTP connection runs through the following stages:

* ngx_event_accept() accepts a client TCP connection. This handler is called in response to a read noti
fication on a listen socket. A new ngx_connecton_t object is created at this stage. The object wraps th
e newly accepted client socket. Each nginx listener provides a handler to pass the new connection objec
t to. For HTTP connections it's ngx_http_init_connection(c)
* ngx_http_init_connection() performs early initialization of an HTTP connection. At this stage an ngx_
http_connection_t object is created for the connection and its reference is stored in connection's data
 field. Later it will be substituted with an HTTP request object. PROXY protocol parser and SSL handsha
ke are started at this stage as well
* ngx_http_wait_request_handler() is a read event handler, that is called when data is available in the
 client socket. At this stage an HTTP request object ngx_http_request_t is created and set to connectio
n's data field
* ngx_http_process_request_line() is a read event handler, which reads client request line. The handler
 is set by ngx_http_wait_request_handler(). Reading is done into connection's buffer. The size of the b
uffer is initially set by the directive client_header_buffer_size. The entire client header is supposed
 to fit the buffer. If the initial size is not enough, a bigger buffer is allocated, whose size is set
by the large_client_header_buffers directive
* ngx_http_process_request_headers() is a read event handler, which is set after ngx_http_process_reque
st_line() to read client request header
* ngx_http_core_run_phases() is called when the request header is completely read and parsed. This func
tion runs request phases from NGX_HTTP_POST_READ_PHASE to NGX_HTTP_CONTENT_PHASE. The last phase is sup
posed to generate response and pass it along the filter chain. The response in not necessarily sent to
the client at this phase. It may remain buffered and will be sent at the finalization stage
* ngx_http_finalize_request() is usually called when the request has generated all the output or produc
ed an error. In the latter case an appropriate error page is looked up and used as the response. If the
 response is not completely sent to the client by this point, an HTTP writer ngx_http_writer() is activ
ated to finish sending outstanding data
* ngx_http_finalize_connection() is called when the response is completely sent to the client and the r
equest can be destroyed. If client connection keepalive feature is enabled, ngx_http_set_keepalive() is
 called, which destroys current request and waits for the next request on the connection. Otherwise, ng
x_http_close_request() destroys both the request and the connection

Request
-------

For each client HTTP request the ngx_http_request_t object is created. Some of the fields of this object:

* connection — pointer to a ngx_connection_t client connection object. Several requests may reference the same connection object at the same time - one main request and its subrequests. After a request is deleted, a new request may be created on the same connection.

    Note that for HTTP connections ngx_connection_t's data field points back to the request. Such request is called active, as opposed to the other requests tied with the connection. Active request is used to handle client connection events and is allowed to output its response to the client. Normally, each request becomes active at some point to be able to send its output

* ctx — array of HTTP module contexts. Each module of type NGX_HTTP_MODULE can store any value (normally, a pointer to a structure) in the request. The value is stored in the ctx array at the module's ctx_index position. The following macros provide a convenient way to get and set request contexts:

    * ngx_http_get_module_ctx(r, module) — returns module's context
    * ngx_http_set_ctx(r, c, module) — sets c as module's context
* main_conf, srv_conf, loc_conf — arrays of current request configurations. Configurations are stored at module's ctx_index positions
* read_event_handler, write_event_handler - read and write event handlers for the request. Normally, an HTTP connection has ngx_http_request_handler() set as both read and write event handlers. This function calls read_event_handler and write_event_handler handlers of the currently active request
* cache — request cache object for caching upstream response
* upstream — request upstream object for proxying
* pool — request pool. This pool is destroyed when the request is deleted. The request object itself is allocated in this pool. For allocations which should be available throughout the client connection's lifetime, ngx_connection_t's pool should be used instead
* header_in — buffer where client HTTP request header in read
* headers_in, headers_out — input and output HTTP headers objects. Both objects contain the headers field of type ngx_list_t keeping the raw list of headers. In addition to that, specific headers are available for getting and setting as separate fields, for example content_length_n, status etc
* request_body — client request body object
* start_sec, start_msec — time point when the request was created. Used for tracking request duration
* method, method_name — numeric and textual representation of client HTTP request method. Numeric values for methods are defined in src/http/ngx_http_request.h with macros NGX_HTTP_GET, NGX_HTTP_HEAD, NGX_HTTP_POST etc
* http_protocol, http_version, http_major, http_minor - client HTTP protocol version in its original textual form (“HTTP/1.0”, “HTTP/1.1” etc), numeric form (NGX_HTTP_VERSION_10, NGX_HTTP_VERSION_11 etc) and separate major and minor versions
* request_line, unparsed_uri — client original request line and URI
* uri, args, exten — current request URI, arguments and file extention. The URI value here might differ from the original URI sent by the client due to normalization. Throughout request processing, these value can change while performing internal redirects
* main — pointer to a main request object. This object is created to process client HTTP request, as opposed to subrequests, created to perform a specific sub-task within the main request
* parent — pointer to a parent request of a subrequest
* postponed — list of output buffers and subrequests in the order they are sent and created. The list is used by the postpone filter to provide consistent request output, when parts of it are created by subrequests
* post_subrequest — pointer to a handler with context to be called when a subrequest gets finalized. Unused for main requests
* posted_requests — list of requests to be started or resumed. Starting or resuming is done by calling the request's write_event_handler. Normally, this handler holds the request main function, which at first runs request phases and then produces the output.

    A request is usually posted by the ngx_http_post_request(r, NULL) call. It is always posted to the main request posted_requests list. The function ngx_http_run_posted_requests(c) runs all requests, posted in the main request of the passed connection's active request. This function should be called in all event handlers, which can lead to new posted requests. Normally, it's called always after invoking a request's read or write handler

* phase_handler — index of current request phase
* ncaptures, captures, captures_data — regex captures produced by the last regex match of the request. While processing a request, there's a number of places where a regex match can happen: map lookup, server lookup by SNI or HTTP Host, rewrite, proxy_redirect etc. Captures produced by a lookup are stored in the above mentioned fields. The field ncaptures holds the number of captures, captures holds captures boundaries, captures_data holds a string, against which the regex was matched and which should be used to extract captures. After each new regex match request captures are reset to hold new values
* count — request reference counter. The field only makes sense for the main request. Increasing the counter is done by simple r->main->count++. To decrease the counter ngx_http_finalize_request(r, rc) should be called. Creation of a subrequest or running request body read process increase the counter
* subrequests — current subrequest nesting level. Each subrequest gets the nesting level of its parent decreased by one. Once the value reaches zero an error is generated. The value for the main request is defined by the NGX_HTTP_MAX_SUBREQUESTS constant
* uri_changes — number of URI changes left for the request. The total number of times a request can change its URI is limited by the NGX_HTTP_MAX_URI_CHANGES constant. With each change the value is decreased until it reaches zero. In the latter case an error is generated. The actions considered as URI changes are rewrites and internal redirects to normal or named locations
* blocked — counter of blocks held on the request. While this value is non-zero, request cannot be terminated. Currently, this value is increased by pending AIO operations (POSIX AIO and thread operations) and active cache lock
* buffered — bitmask showing which modules have buffered the output produced by the request. A number of filters can buffer output, for example sub_filter can buffer data due to a partial string match, copy filter can buffer data because of the lack of free output_buffers etc. As long as this value is non-zero, request is not finalized, expecting the flush
* header_only — flag showing that output does not require body. For example, this flag is used by HTTP HEAD requests
* keepalive — flag showing if client connection keepalive is supported. The value is inferred from HTTP version and “Connection” header value

* header_sent — flag showing that output header has already been sent by the request
* internal — flag showing that current request is internal. To enter the internal state, a request should pass through an internal redirect or be a subrequest. Internal requests are allowed to enter internal locations
* allow_ranges — flag showing that partial response can be sent to client, if requested by the HTTP Range header
* subrequest_ranges — flag showing that a partial response is allowed to be sent while processing a subrequest
* single_range — flag showing that only a single continuous range of output data can be sent to the client. This flag is usually set when sending a stream of data, for example from a proxied server, and the entire response is not available at once
* main_filter_need_in_memory, filter_need_in_memory — flags showing that the output should be produced in memory buffers but not in files. This is a signal to the copy filter to read data from file buffers even if sendfile is enabled. The difference between these two flags is the location of filter modules which set them. Filters called before the postpone filter in filter chain, set filter_need_in_memory requesting that only the current request output should come in memory buffers. Filters called later in filter chain set main_filter_need_in_memory requiring that both the main request and all the subrequest read files in memory while sending output
* filter_need_temporary — flag showing that the request output should be produced in temporary buffers, but not in readonly memory buffers or file buffers. This is used by filters which may change output directly in the buffers, where it's sent

配置
-------------

Each HTTP module may have three types of configuration:

* Main configuration. This configuration applies to the entire nginx http{} block. This is global confi
guration. It stores global settings for a module
* Server configuration. This configuraion applies to a single nginx server{}. It stores server-specific
 settings for a module
* Location configuration. This configuraion applies to a single location{}, if{} or limit_except() bloc
k. This configuration stores settings specific to a location

Configuration structures are created at nginx configuration stage by calling functions, which allocate
these structures, initialize them and merge. The following example shows how to create a simple module
location configuration. The configuration has one setting foo of unsiged integer type.

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

As seen in the example, ngx_http_foo_create_loc_conf() function creates a new configuration structure a
nd ngx_http_foo_merge_loc_conf() merges a configuration with another configuration from a higher level.
 In fact, server and location configuration do not only exist at server and location levels, but also c
reated for all the levels above. Specifically, a server configuration is created at the main level as w
ell and location configurations are created for main, server and location levels. These configurations
make it possible to specify server and location-specific settings at any level of nginx configuration f
ile. Eventually configurations are merged down. To indicate a missing setting and ignore it while mergi
ng, nginx provides a number of macros like NGX_CONF_UNSET and NGX_CONF_UNSET_UINT. Standard nginx merge
 macros like ngx_conf_merge_value() and ngx_conf_merge_uint_value() provide a convenient way to merge a
 setting and set the default value if none of configurations provided an explicit value. For complete l
ist of macros for different types see src/core/ngx_conf_file.h.

To access configuration of any HTTP module at configuration time, the following macros are available. T
hey receive ngx_conf_t reference as the first argument.

* ngx_http_conf_get_module_main_conf(cf, module)
* ngx_http_conf_get_module_srv_conf(cf, module)
* ngx_http_conf_get_module_loc_conf(cf, module)

The following example gets a pointer to a location configuration of standard nginx core module ngx_http
_core_module and changes location content handler kept in the handler field of the structure.

As seen in the example, ngx_http_foo_create_loc_conf() function creates a new configuration structure a
nd ngx_http_foo_merge_loc_conf() merges a configuration with another configuration from a higher level.
 In fact, server and location configuration do not only exist at server and location levels, but also c
reated for all the levels above. Specifically, a server configuration is created at the main level as w
ell and location configurations are created for main, server and location levels. These configurations
make it possible to specify server and location-specific settings at any level of nginx configuration f
ile. Eventually configurations are merged down. To indicate a missing setting and ignore it while mergi
ng, nginx provides a number of macros like NGX_CONF_UNSET and NGX_CONF_UNSET_UINT. Standard nginx merge
 macros like ngx_conf_merge_value() and ngx_conf_merge_uint_value() provide a convenient way to merge a
 setting and set the default value if none of configurations provided an explicit value. For complete l
ist of macros for different types see src/core/ngx_conf_file.h.

To access configuration of any HTTP module at configuration time, the following macros are available. T
hey receive ngx_conf_t reference as the first argument.

* ngx_http_conf_get_module_main_conf(cf, module)
* ngx_http_conf_get_module_srv_conf(cf, module)
* ngx_http_conf_get_module_loc_conf(cf, module)

The following example gets a pointer to a location configuration of standard nginx core module ngx_http
_core_module and changes location content handler kept in the handler field of the structure.

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

In runtime the following macros are available to get configurations of HTTP modules.

* ngx_http_get_module_main_conf(r, module)
* ngx_http_get_module_srv_conf(r, module)
* ngx_http_get_module_loc_conf(r, module)

These macros receive a reference to an HTTP request ngx_http_request_t. Main configuration of a request
 never changes. Server configuration may change from a default one after choosing a virtual server for
a request. Request location configuration may change multiple times as a result of a rewrite or interna
l redirect. The following example shows how to access HTTP configuration in runtime.

```
static ngx_int_t
ngx_http_foo_handler(ngx_http_request_t *r)
{
    ngx_http_foo_loc_conf_t  *flcf;

    flcf = ngx_http_get_module_loc_conf(r, ngx_http_foo_module);

    ...
}
```


Phases
------
Each HTTP request passes through a list of HTTP phases. Each phase is specialized in a particular type of processing. Most phases allow installing handlers. The phase handlers are called successively once the request reaches the phase. Many standard nginx modules install their phase handlers as a way to get called at a specific request processing stage. Following is the list of nginx HTTP phases.

* NGX_HTTP_POST_READ_PHASE is the earliest phase. The ngx_http_realip_module installs its handler at this phase. This allows to substitute client address before any other module is invoked
* NGX_HTTP_SERVER_REWRITE_PHASE is used to run rewrite script, defined at the server level, that is out of any location block. The ngx_http_rewrite_module installs its handler at this phase
* NGX_HTTP_FIND_CONFIG_PHASE — a special phase used to choose a location based on request URI. This phase does not allow installing any handlers. It only performs the default action of choosing a location. Before this phase, the server default location is assigned to the request. Any module requesting a location configuration, will receive the default server location configuration. After this phase a new location is assigned to the request
* NGX_HTTP_REWRITE_PHASE — same as NGX_HTTP_SERVER_REWRITE_PHASE, but for a new location, chosen at the prevous phase
* NGX_HTTP_POST_REWRITE_PHASE — a special phase, used to redirect the request to a new location, if the URI was changed during rewrite. The redirect is done by going back to NGX_HTTP_FIND_CONFIG_PHASE. No handlers are allowed at this phase
* NGX_HTTP_PREACCESS_PHASE — a common phase for different types of handlers, not associated with access check. Standard nginx modules ngx_http_limit_conn_module and ngx_http_limit_req_module register their handlers at this phase
* NGX_HTTP_ACCESS_PHASE — used to check access permissions for the request. Standard nginx modules such as ngx_http_access_module and ngx_http_auth_basic_module register their handlers at this phase. If configured so by the satisfy directive, only one of access phase handlers may allow access to the request in order to confinue processing
* NGX_HTTP_POST_ACCESS_PHASE — a special phase for the satisfy any case. If some access phase handlers denied the access and none of them allowed, the request is finalized. No handlers are supported at this phase
* NGX_HTTP_TRY_FILES_PHASE — a special phase, for the try_files feature. No handlers are allowed at this phase
* NGX_HTTP_CONTENT_PHASE — a phase, at which the response is supposed to be generated. Multiple nginx standard modules register their handers at this phase, for example ngx_http_index_module or ngx_http_static_module. All these handlers are called sequentially until one of them finally produces the output. It's also possible to set content handlers on a per-location basis. If the ngx_http_core_module's location configuration has handler set, this handler is called as the content handler and content phase handlers are ignored
* NGX_HTTP_LOG_PHASE is used to perform request logging. Currently, only the ngx_http_log_module registers its handler at this stage for access logging. Log phase handlers are called at the very end of request processing, right before freeing the request

Following is the example of a preaccess phase handler.

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

Phase handlers are expected to return specific codes:

* NGX_OK — proceed to the next phase
* NGX_DECLINED — proceed to the next handler of the current phase. If current handler is the last in current phase, move to the next phase
* NGX_AGAIN, NGX_DONE — suspend phase handling until some future event. This can be for example asynchronous I/O operation or just a delay. It is supposed, that phase handling will be resumed later by calling ngx_http_core_run_phases()
* Any other value returned by the phase handler is treated as a request finalization code, in particular, HTTP response code. The request is finalized with the code provided

Some phases treat return codes in a slightly different way. At content phase, any return code other that NGX_DECLINED is considered a finalization code. As for the location content handlers, any return from them is considered a finalization code. At access phase, in satisfy any mode, returning a code other than NGX_OK, NGX_DECLINED, NGX_AGAIN, NGX_DONE is considered a denial. If none of future access handlers allow access or deny with a new code, the denial code will become the finalization code.

Variables
---------

Accessing existing variables
----------------------------

Variables may be referenced using index (this is the most common method) or names (see below in the section about creating variables). Index is created at configuration stage, when a variable is added to configuration. The variable index can be obtained using ngx_http_get_variable_index():

```
ngx_str_t  name;  /* ngx_string("foo") */
ngx_int_t  index;

index = ngx_http_get_variable_index(cf, &name);
```

Here, the cf is a pointer to nginx configuration and the name points to a string with the variable name. The function returns NGX_ERROR on error or valid index otherwise, which is typically stored somewhere in a module configuration for future use.

All HTTP variables are evaluated in the context of HTTP request and results are specific to and cached in HTTP request. All functions that evaluate variables return ngx_http_variable_value_t type, representing the variable value:

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

where:

* len — length of a value
* data — value itself
* valid — value is valid
* not_found — variable was not found and thus the data and len fields are irrelevant; this may happen, for example, with such variables as $arg_foo when a corresponding argument was not passed in a request
* no_cacheable — do not cache result
* escape — used internally by the logging module to mark values that require escaping on output

The ngx_http_get_flushed_variable() and ngx_http_get_indexed_variable() functions are used to obtain the variable value. They have the same interface - accepting a HTTP request r as a context for evaluating the variable and an index, identifying it. Example of typical usage:

```
ngx_http_variable_value_t  *v;

v = ngx_http_get_flushed_variable(r, index);

if (v == NULL || v->not_found) {
    /* we failed to get value or there is no such variable, handle it */
    return NGX_ERROR;
}

/* some meaningful value is found */
```

The difference between functions is that the ngx_http_get_indexed_variable() returns cached value and ngx_http_get_flushed_variable() flushes cache for non-cacheable variables.

There are cases when it is required to deal with variables which names are not known at configuration time and thus they cannot be accessed using indexes, for example in modules like SSI or Perl. The ngx_http_get_variable(r, name, key) function may be used in such cases. It searches for the variable with a given name and its hash key.

Creating variables
------------------

To create a variable ngx_http_add_variable() function is used. It takes configuration (where variable is registered), variable name and flags that control its behaviour:

* NGX_HTTP_VAR_CHANGEABLE  — allows redefining the variable; If another module will define a variable with such name, no conflict will happen. For example, this allows user to override variables using the set directive.
* NGX_HTTP_VAR_NOCACHEABLE  — disables caching, is useful for such variables as $time_local
* NGX_HTTP_VAR_NOHASH  — indicates that this variable is only accessible by index, not by name. This is a small optimization which may be used when it is known that the variable is not needed in modules like SSI or Perl.
* NGX_HTTP_VAR_PREFIX  — the name of this variable is a prefix. A handler must implement additional logic to obtain value of specific variable. For example, all “arg_” variables are processed by the same handler which performs lookup in request arguments and returns value of specific argument.

The function returns NULL in case of error or a pointer to ngx_http_variable_t:

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

The get and set handlers are called to obtain or set the variable value, data will be passed to variable handlers, index will hold assigned variable index, used to reference the variable.

Usually, a null-terminated static array of such structures is created by a module and processed at the preconfiguration stage to add variables into configuration:

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

This function is used to initialize the preconfiguration field of the HTTP module context and is called before parsing HTTP configuration, so it could refer to these variables.

The get handler is responsible for evaluating the variable in a context of specific request, for example:

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

It returns NGX_ERROR in case of internal error (for example, failed memory allocation) or NGX_OK otherwise. The status of variable evaluation may be understood by inspecting flags of the ngx_http_variable_value_t (see description above).

The set handler allows setting the property referred by the variable. For example, the $limit_rate variable set handler modifies the request's limit_rate field:

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

Complex values
--------------

A complex value, despite its name, provides an easy way to evaluate expressions that may contain text, variables, and their combination.

The complex value description in ngx_http_compile_complex_value is compiled at the configuration stage into ngx_http_complex_value_t which is used at runtime to obtain evaluated expression results.

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

Here, ccv holds all parameters that are required to initialize the complex value cv:

* cf — configuration pointer
* value — string for parsing (input)
* complex_value — compiled value (output)
* zero — flag that enables zero-terminating value
* conf_prefix — prefixes result with configuration prefix (the directory where nginx is currently looking for configuration)
* root_prefix — prefixes result with root prefix (this is the normal nginx installation prefix)

The zero flag is usable when results are to be passed to libraries that require zero-terminated strings, and prefixes are handy when dealing with filenames.

Upon successful compilation, cv.lengths may be inspected to get information about the presence of variables in the expression. The NULL value means that the expression contained static text only, and there is no need in storing it as a complex value, so a simple string can be used.

The ngx_http_set_complex_value_slot() is a convenient function used to initialize complex value completely right in the directive declaration.

At runtime, a complex value may be calculated using the ngx_http_complex_value() function:

```
ngx_str_t  res;

if (ngx_http_complex_value(r, &cv, &res) != NGX_OK) {
    return NGX_ERROR;
}
```

Given the request r and previously compiled value cv the function will evaluate expression and put result into res.

Request redirection
-------------------

An HTTP request is always connected to a location via the loc_conf field of the ngx_http_request_t structure. This means that at any point the location configuration of any module can be retrieved from the request by calling ngx_http_get_module_loc_conf(r, module). Request location may be changed several times throughout its lifetime. Initially, a default server location of the default server is assigned to a request. Once a request switches to a different server (chosen by the HTTP “Host” header or SSL SNI extension), the request switches to the default location of that server as well. The next change of the location takes place at the NGX_HTTP_FIND_CONFIG_PHASE request phase. At this phase a location is chosen by request URI among all non-named locations configured for the server. The ngx_http_rewrite_module may change the request URI at the NGX_HTTP_REWRITE_PHASE request phase as a result of rewrite and return to the NGX_HTTP_FIND_CONFIG_PHASE phase for choosing a new location based on the new URI.

It is also possible to redirect a request to a new location at any point by calling one of the functions ngx_http_internal_redirect(r, uri, args) or ngx_http_named_location(r, name).

The function ngx_http_internal_redirect(r, uri, args) changes the request URI and returns the request to the NGX_HTTP_SERVER_REWRITE_PHASE phase. The request proceeds with a server default location. Later at NGX_HTTP_FIND_CONFIG_PHASE a new location is chosen based on the new request URI.

The following example performs an internal redirect with the new request arguments.

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

The following example performs a redirect to a named location @foo.

```
ngx_int_t
ngx_http_foo_named_redirect(ngx_http_request_t *r)
{
    ngx_str_t  name;

    ngx_str_set(&name, "foo");

    return ngx_http_named_location(r, &name);
}
```

Both functions ngx_http_internal_redirect(r, uri, args) and ngx_http_named_location(r, name) may be called when a request already has some contexts saved in its ctx field by nginx modules. These contexts could become inconsistent with the new location configuration. To prevent inconsistency, all request contexts are erased by both redirect functions.

Redirected and rewritten requests become internal and may access the internal locations. Internal requests have the internal flag set.

Subrequests
-----------

Subrequests are primarily used to include output of one request into another, possibly mixed with other data. A subrequest looks like a normal request, but shares some data with its parent. Particularly, all fields related to client input are shared since a subrequest does not receive any other input from client. The request field parent for a subrequest keeps a link to its parent request and is NULL for the main request. The field main keeps a link to the main request in a group of requests.

A subrequest starts with NGX_HTTP_SERVER_REWRITE_PHASE phase. It passes through the same phases as a normal request and is assigned a location based on its own URI.

Subrequest output header is always ignored. Subrequest output body is placed by the ngx_http_postpone_filter into the right position in relation to other data produced by the parent request.

Subrequests are related to the concept of active requests. A request r is considered active if c->data == r, where c is the client connection object. At any point, only the active request in a request group is allowed to output its buffers to the client. A non-active request can still send its data to the filter chain, but they will not pass beyond the ngx_http_postpone_filter and will remain buffered by that filter until the request becomes active. Here are some rules of request activation:

* Initially, the main request is active
* The first subrequest of an active request becomes active right after creation
* The ngx_http_postpone_filter activates the next request in active request's subrequest list, once all data prior to that request are sent
* When a request is finalized, its parent is activated

A subrequest is created by calling the function ngx_http_subrequest(r, uri, args, psr, ps, flags), where r is the parent request, uri and args are URI and arguments of the subrequest, psr is the output parameter, receiving the newly created subrequest reference, ps is a callback object for notifying the parent request that the subrequest is being finalized, flags is subrequest creation flags bitmask. The following flags are available:

* NGX_HTTP_SUBREQUEST_IN_MEMORY - subrequest output should not be sent to the client, but rather stored in memory. This only works for proxying subrequests. After subrequest finalization its output is available in r->upstream->buffer buffer of type ngx_buf_t
* NGX_HTTP_SUBREQUEST_WAITED - the subrequest done flag is set even if it is finalized being non-active. This subrequest flag is used by the SSI filter
* NGX_HTTP_SUBREQUEST_CLONE - the subrequest is created as a clone of its parent. It is started at the same location and proceeds from the same phase as the parent request

The following example creates a subrequest with the URI of "/foo".

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

This example clones the current request and sets a finalization callback for the subrequest.

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

Subrequests are normally created in a body filter. In this case subrequest output can be treated as any other explicit request output. This means that eventually the output of a subrequest is sent to the client after all explicit buffers passed prior to subrequest creation and before any buffers passed later. This ordering is preserved even for large hierarchies of subrequests. The following example inserts a subrequest output after all request data buffers, but before the final buffer with the last_buf flag.

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

A subrequest may also be created for other purposes than data output. For example, the ngx_http_auth_request_module creates a subrequest at NGX_HTTP_ACCESS_PHASE phase. To disable any output at this point, the subrequest header_only flag is set. This prevents subrequest body from being sent to the client. Its header is ignored anyway. The result of the subrequest can be analyzed in the callback handler.

Request finalization
--------------------

An HTTP request is finalized by calling the function ngx_http_finalize_request(r, rc). It is usually finalized by the content handler after sending all output buffers to the filter chain. At this point the output may not be completely sent to the client, but remain buffered somewhere along the filter chain. If it is, the ngx_http_finalize_request(r, rc) function will automatically install a special handler ngx_http_writer(r) to finish sending the output. A request is also finalized in case of an error or if a standard HTTP response code needs to be returned to the client.

The function ngx_http_finalize_request(r, rc) expects the following rc values:

* NGX_DONE - fast finalization. Decrement request count and destroy the request if it reaches zero. The client connection may still be used for more requests after that
* NGX_ERROR, NGX_HTTP_REQUEST_TIME_OUT (408), NGX_HTTP_CLIENT_CLOSED_REQUEST (499) - error finalization. Terminate the request as soon as possible and close the client connection.
* NGX_HTTP_CREATED (201), NGX_HTTP_NO_CONTENT (204), codes greater than or equal to NGX_HTTP_SPECIAL_RESPONSE (300) - special response finalization. For these values nginx either sends a default code response page to the client or performs the internal redirect to an error_page location if it's configured for the code
* Other codes are considered success finalization codes and may activate the request writer to finish sending the response body. Once body is completely sent, request count is decremented. If it reaches zero, the request is destroyed, but the client connection may still be used for other requests. If count is positive, there are unfinished activities within the request, which will be finalized at a later point.

Request body
------------

For dealing with client request body, nginx provides the following functions: ngx_http_read_client_request_body(r, post_handler) and ngx_http_discard_request_body(r). The first function reads the request body and makes it available via the request_body request field. The second function instructs nginx to discard (read and ignore) the request body. One of these functions must be called for every request. Normally, it is done in the content handler.

Reading or discarding client request body from a subrequest is not allowed. It should always be done in the main request. When a subrequest is created, it inherits the parent request_body object which can be used by the subrequest if the main request has previously read the request body.

The function ngx_http_read_client_request_body(r, post_handler) starts the process of reading the request body. When the body is completely read, the post_handler callback is called to continue processing the request. If request body is missing or already read, the callback is called immediately. The function ngx_http_read_client_request_body(r, post_handler) allocates the request_body request field of type ngx_http_request_body_t. The field bufs of this object keeps the result as a buffer chain. The body can be saved in memory buffers or file buffers, if client_body_buffer_size is not enough to fit the entire body in memory.

The following example reads client request body and returns its size.

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

The following fields of the request affect the way request body is read:

* request_body_in_single_buf - read body to a single memory buffer
* request_body_in_file_only - always read body to a file, even if fits the memory buffer
* request_body_in_persistent_file - do not unlink the file right after creation. Such a file can be moved to another directory
* request_body_in_clean_file - unlink the file the when the request is finalized. This can be useful when a file was supposed to be moved to another directory but eventually was not moved for some reason
* request_body_file_group_access - enable file group access. By default a file is created with 0600 access mask. When the flag is set, 0660 access mask is used
* request_body_file_log_level - log file errors with this log level
* request_body_no_buffering - read request body without buffering

When the request_body_no_buffering flag is set, the unbuffered mode of reading the request body is enabled. In this mode, after calling ngx_http_read_client_request_body(), the bufs chain may keep only a part of the body. To read the next part, the ngx_http_read_unbuffered_request_body(r) function should be called. The return value of NGX_AGAIN and the request flag reading_body indicate that more data is available. If bufs is NULL after calling this function, there is nothing to read at the moment. The request callback read_event_handler will be called when the next part of request body is available.

Response
--------

An HTTP response in nginx is produced by sending the response header followed by the optional response body. Both header and body are passed through a chain of filters and eventually get written to the client socket. An nginx module can install its handler into the header or body filter chain and process the output coming from the previous handler.

Response header
---------------

Output header is sent by the function ngx_http_send_header(r). Prior to calling this function, r->headers_out should contain all the data required to produce the HTTP response header. It's always required to set the status field of r->headers_out. If the response status suggests that a response body follows the header, content_length_n can be set as well. The default value for this field is -1, which means that the body size is unknown. In this case, chunked transfer encoding is used. To output an arbitrary header, headers list should be appended.

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

Header filters
--------------

The ngx_http_send_header(r) function invokes the header filter chain by calling the top header filter handler ngx_http_top_header_filter. It's assumed that every header handler calls the next handler in chain until the final handler ngx_http_header_filter(r) is called. The final header handler constructs the HTTP response based on r->headers_out and passes it to the ngx_http_writer_filter for output.

To add a handler to the header filter chain, one should store its address in ngx_http_top_header_filter global variable at configuration time. The previous handler address is normally stored in a module's static variable and is called by the newly added handler before exiting.

The following is an example header filter module, adding the HTTP header "X-Foo: foo" to every output with the status 200.

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

Response body
-------------

Response body is sent by calling the function ngx_http_output_filter(r, cl). The function can be called multiple times. Each time it sends a part of the response body passed as a buffer chain. The last body buffer should have the last_buf flag set.

The following example produces a complete HTTP output with "foo" as its body. In order for the example to work not only as a main request but as a subrequest as well, the last_in_chain_flag is set in the last buffer of the output. The last_buf flag is set only for the main request since a subrequest's last buffers does not end the entire output.

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

Body filters
------------

The function ngx_http_output_filter(r, cl) invokes the body filter chain by calling the top body filter handler ngx_http_top_body_filter. It's assumed that every body handler calls the next handler in chain until the final handler ngx_http_write_filter(r, cl) is called.

A body filter handler receives a chain of buffers. The handler is supposed to process the buffers and pass a possibly new chain to the next handler. It's worth noting that the chain links ngx_chain_t of the incoming chain belong to the caller. They should never be reused or changed. Right after the handler completes, the caller can use its output chain links to keep track of the buffers it has sent. To save the buffer chain or to substitute some buffers before sending further, a handler should allocate its own chain links.

Following is the example of a simple body filter counting the number of body bytes. The result is available as the $counter variable which can be used in the access log.

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

Building filter modules
-----------------------

When writing a body or header filter, a special care should be taken of the filters order. There's a number of header and body filters registered by nginx standard modules. It's important to register a filter module in the right place in respect to other filters. Normally, filters are registered by modules in their postconfiguration handlers. The order in which filters are called is obviously the reverse of when they are registered.

A special slot HTTP_AUX_FILTER_MODULES for third-party filter modules is provided by nginx. To register a filter module in this slot, the ngx_module_type variable should be set to the value of HTTP_AUX_FILTER in module's configuration.

The following example shows a filter module config file assuming it only has one source file ngx_http_foo_filter_module.c

```
ngx_module_type=HTTP_AUX_FILTER
ngx_module_name=ngx_http_foo_filter_module
ngx_module_srcs="$ngx_addon_dir/ngx_http_foo_filter_module.c"

. auto/module
```

Buffer reuse
------------

When issuing or altering a stream of buffers, it's often desirable to reuse the allocated buffers. A standard approach widely adopted in nginx code is to keep two buffer chains for this purpose: free and busy. The free chain keeps all free buffers. These buffers can be reused. The busy chain keeps all buffers sent by the current module which are still in use by some other filter handler. A buffer is considered in use if its size is greater than zero. Normally, when a buffer is consumed by a filter, its pos (or file_pos for a file buffer) is moved towards last (file_last for a file buffer). Once a buffer is completely consumed, it's ready to be reused. To update the free chain with newly freed buffers, it's enough to iterate over the busy chain and move the zero size buffers at the head of it to free. This operation is so common that there is a special function ngx_chain_update_chains(free, busy, out, tag) which does this. The function appends the output chain out to busy and moves free buffers from the top of busy to free. Only the buffers with the given tag are reused. This lets a module reuse only the buffers allocated by itself.

The following example is a body filter inserting the “foo” string before each incoming buffer. The new buffers allocated by the module are reused if possible. Note that for this example to work properly, it's also required to set up a header filter and reset content_length_n to -1, which is beyond the scope of this section.

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

