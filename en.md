Development guide
===========

* [Introduction](#introduction)
    * [Code layout](#code-layout)
    * [Include files](#include-files)
    * [Integers](#integers)
    * [Common return codes](#common-return-codes)
    * [Error handling](#error-handling)
* [Strings](#strings)
    * [Overview](#overview)
    * [Formatting](#formatting)
    * [Numeric conversion](#numeric-conversion)
    * [Regular expressions](#regular-expressions)
* [Containers](#containers)
    * [Array](#array)
    * [List](#list)
    * [Queue](#queue)
    * [Red Black tree](#red-Black-tree)
    * [Hash](#hash)
* [Memory management](#memory-management)
    * [Heap](#heap)
    * [Pool](#pool)
    * [Shared memory](#shared-memory)
* [Logging](#logging)
* [Cycle](#cycle)
* [Buffer](#buffer)
* [Networking](#networking)
    * [Connection](#connection)
* [Events](#events)
    * [Event](#event)
    * [I/O events](#i/O-events)
    * [Timer events](#timer-events)
    * [Posted events](#posted-events)
    * [Event loop](#event-loop)
* [Processes](#processes)
* [Modules](#modules)
    * [Adding new modules](#adding-new-modules)
    * [Core modules](#core-modules)
    * [Configuration directives](#configuration-directives)
* [HTTP](#hTTP)
    * [Connection](#connection)
    * [Request](#request)
    * [Configuration](#configuration)
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
    * [Load balancing](#load-balancing)
    
Introduction
============

Code layout
-----------
* auto — build scripts
* src
    * core — basic types and functions — string, array, log, pool etc
* event — event core
    * modules — event notification modules: epoll, kqueue, select etc
* http — core HTTP module and common code
    * modules — other HTTP modules
    * v2 — HTTPv2
* mail — mail modules
* os — platform-specific code
    * unix
    * win32
* stream — stream modules

Include files
-------------
Each nginx file should start with including the following two files:

```
#include <ngx_config.h>
#include <ngx_core.h>
```

In addition to that, HTTP code should include

```
#include <ngx_http.h>
```

Mail code should include

```
#include <ngx_mail.h>
```

Stream code should include

```
#include <ngx_stream.h>
```

Integers
--------
For general purpose, nginx code uses the following two integer types ngx_int_t and ngx_uint_t which are typedefs for intptr_t and uintptr_t.

Common return codes
-------------------
Most functions in nginx return the following codes:

* NGX_OK — operation succeeded
* NGX_ERROR — operation failed
* NGX_AGAIN — operation incomplete, function should be called again
* NGX_DECLINED — operation rejected, for example, if disabled in configuration. This is never an error
* NGX_BUSY — resource is not available
* NGX_DONE — operation done or continued elsewhere. Also used as an alternative success code
* NGX_ABORT — function was aborted. Also used as an alternative error code

Error handling
--------------
For getting the last system error code, the ngx_errno macro is available. It's mapped to errno on POSIX platforms and to GetLastError() call in Windows. For getting the last socket error number, the ngx_socket_errno macro is available. It's mapped to errno on POSIX systems as well, and to WSAGetLastError() call on Windows. For performance reasons the values of ngx_errno or ngx_socket_errno should not be accessed more than once in a row. The error value should be stored in a local variable of type ngx_err_t for using multiple times, if required. For setting errors, ngx_set_errno(errno) and ngx_set_socket_errno(errno) macros are available.

The values of ngx_errno or ngx_socket_errno can be passed to logging functions ngx_log_error() and ngx_log_debugX(), in which case system error text is added to the log message.

Example using ngx_errno:

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

Strings
=======

Overview
--------
For C strings, nginx code uses unsigned character type pointer u_char *.

The nginx string type ngx_str_t is defined as follows:

```
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

The len field holds the string length, data holds the string data. The string, held in ngx_str_t, may or may not be null-terminated after the len bytes. In most cases it’s not. However, in certain parts of code (for example, when parsing configuration), ngx_str_t objects are known to be null-terminated, and that knowledge is used to simplify string comparison and makes it easier to pass those strings to syscalls.

A number of string operations are provided in nginx. They are declared in src/core/ngx_string.h. Some of them are wrappers around standard C functions:

* ngx_strcmp()
* ngx_strncmp()
* ngx_strstr()
* ngx_strlen()
* ngx_strchr()
* ngx_memcmp()
* ngx_memset()
* ngx_memcpy()
* ngx_memmove()

Some nginx-specific string functions:

* ngx_memzero() fills memory with zeroes
* ngx_cpymem() does the same as ngx_memcpy(), but returns the final destination address This one is handy for appending multiple strings in a row
* ngx_movemem() does the same as ngx_memmove(), but returns the final destination address.
* ngx_strlchr() searches for a character in a string, delimited by two pointers

Some case conversion and comparison functions:

* ngx_tolower()
* ngx_toupper()
* ngx_strlow()
* ngx_strcasecmp()
* ngx_strncasecmp()

Formatting
----------
A number of formatting functions are provided by nginx. These functions support nginx-specific types:
* ngx_sprintf(buf, fmt, ...)
* ngx_snprintf(buf, max, fmt, ...)
* ngx_slrintf(buf, last, fmt, ...)
* ngx_vslprint(buf, last, fmt, args)
* ngx_vsnprint(buf, max, fmt, args)

The full list of formatting options, supported by these functions, can be found in src/core/ngx_string.c. Some of them are:
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

The ‘u’ modifier makes most types unsigned, ‘X’/‘x’ convert output to hex.

Example:
```
u_char     buf[NGX_INT_T_LEN];
size_t     len;
ngx_int_t  n;

/* set n here */

len = ngx_sprintf(buf, "%ui", n) — buf;
```


Numeric conversion
------------------
Several functions for numeric conversion are implemented in nginx:
* ngx_atoi(line, n) — converts a string of given length to a positive integer of type ngx_int_t. Returns NGX_ERROR on error
* ngx_atosz(line, n) — same for ssize_t type
* ngx_atoof(line, n) — same for off_t type
* ngx_atotm(line, n) — same for time_t type
* ngx_atofp(line, n, point) — converts a fixed point floating number of given length to a positive integer of type ngx_int_t. The result is shifted left by points decimal positions. The string representation of the number is expected to have no more than points fractional digits. Returns NGX_ERROR on error. For example, ngx_atofp("10.5", 4, 2) returns 1050
* ngx_hextoi(line, n) — converts hexadecimal representation of a positive integer to ngx_int_t. Returns NGX_ERROR on error

Regular expressions
-------------------
The regular expressions interface in nginx is a wrapper around the PCRE library. The corresponding header file is src/core/ngx_regex.h.

To use a regular expression for string matching, first, it needs to be compiled, this is usually done at configuration phase. Note that since PCRE support is optional, all code using the interface must be protected by the surrounding NGX_PCRE macro:

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

After successful compilation, ngx_regex_compile_t structure fields captures and named_captures are filled with count of all and named captures respectively found in the regular expression.

Later, the compiled regular expression may be used to match strings against it:

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

The arguments of ngx_regex_exec() are: the compiled regular expression re, string to match s, optional array of integers to hold found captures and its size. The captures array size must be a multiple of three, per requirements of the PCRE API. In the example, its size is calculated from a total number of captures plus one for the matched string itself.

Now, if there are matches, captures may be accessed:

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

The ngx_regex_exec_array() function accepts the array of ngx_regex_elt_t elements (which are just compiled regular expressions with associated names), a string to match and a log. The function will apply expressions from the array to the string until the match is found or no more expressions are left. The return value is NGX_OK in case of match and NGX_DECLINED otherwise, or NGX_ERROR in case of error.

Containers
==========

Array
-----
The nginx array type ngx_array_t is defined as follows

```
typedef struct {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
} ngx_array_t;
```

The elements of array are available through the elts field. The number of elements is held in the nelts field. The size field holds the size of a single element and is set when initializing the array.

An array can be created in a pool with the ngx_array_create(pool, n, size) call. An already allocated array object can be initialized with the ngx_array_init(array, pool, n, size) call.

```
ngx_array_t  *a, b;

/* create an array of strings with preallocated memory for 10 elements */
a = ngx_array_create(pool, 10, sizeof(ngx_str_t));

/* initialize string array for 10 elements */
ngx_array_init(&b, pool, 10, sizeof(ngx_str_t));
```

Adding elements to array are done with the following functions:

* ngx_array_push(a) adds one tail element and returns pointer to it
* ngx_array_push_n(a, n) adds n tail elements and returns pointer to the first one

If currently allocated memory is not enough for new elements, a new memory for elements is allocated and existing elements are copied to that memory. The new memory block is normally twice as large, as the existing one.

```
s = ngx_array_push(a);
ss = ngx_array_push_n(&b, 3);
```

List
----
List in nginx is a sequence of arrays, optimized for inserting a potentially large number of items. The list type is defined as follows:

```
typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
```

The actual items are stored in list parts, defined as follows:

```
typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
```

Initially, a list must be initialized by calling ngx_list_init(list, pool, n, size) or created by calling ngx_list_create(pool, n, size). Both functions receive the size of a single item and a number of items per list part. The ngx_list_push(list) function is used to add an item to the list. Iterating over the items is done by direct accessing the list fields, as seen in the example:

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

The primary use for the list in nginx is HTTP input and output headers.

The list does not support item removal. However, when needed, items can internally be marked as missing without actual removing from the list. For example, HTTP output headers which are stored as ngx_table_elt_t objects, are marked as missing by setting the hash field of ngx_table_elt_t to zero. Such items are explicitly skipped, when iterating over the headers.

Queue
-----
TODO

Red-Black tree
--------------
TODO

Hash
----
TODO

Memory management
=================

Heap
----
TODO

Pool
----
TODO

Shared memory
-------------
TODO

Logging
=======

Cycle
=====

Buffer
======

Networking
==========

Connection
----------
TODO

Events
======

Event
-----
TODO

I/O events
----------
TODO

Timer events
------------
TODO

Posted events
-------------
TODO

Event loop
----------
TODO

Processes
=========

Modules
=======

Adding new modules
------------------
TODO

Core modules
------------
TODO

Configuration directives
------------------------
TODO

HTTP
====

Connection
----------
TODO

Request
-------
TODO

Configuration
-------------
TODO

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
TODO

Complex values
--------------
TODO

Request redirection
-------------------
TODO

Subrequests
-----------
TODO

Request finalization
--------------------
TODO

Request body
------------
TODO

Response
--------
TODO

Response body
-------------
TODO

Body filters
------------
TODO

Building filter modules
-----------------------
TODO

Buffer reuse
------------
TODO

Load balancing
--------------
The ngx_http_upstream_module provides basic functionality to pass requests to remote servers. This functionality is used by modules that implement specific protocols, such as HTTP or FastCGI. The module also provides an interface for creating custom load balancing modules and implements a default round-robin balancing method.

Examples of modules that implement alternative load balancing methods are least_conn and hash. Note that these modules are actually implemented as extensions of the upstream module and share a lot of code, such as representation of a server group. The keepalive module is an example of an independent module, extending upstream functionality.

The ngx_http_upstream_module may be configured explicitly by placing the corresponding upstream block into the configuration file, or implicitly by using directives that accept a URL evaluated at some point to the list of servers, for example, proxy_pass. Only explicit configurations may use an alternative load balancing method. The upstream module configuration has its own directive context NGX_HTTP_UPS_CONF. The structure is defined as follows:

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

* srv_conf — configuration context of upstream modules
* servers — array of ngx_http_upstream_server_t, the result of parsing a set of server directives in the upstream block
* flags — flags that mostly mark which features (configured as parameters of the server directive) are supported by the particular load balancing method.
   * NGX_HTTP_UPSTREAM_CREATE — used to distinguish explicitly defined upstreams from automatically created by proxy_pass and “friends” (FastCGI, SCGI, etc.)
   * NGX_HTTP_UPSTREAM_WEIGHT — “weight” is supported
   * NGX_HTTP_UPSTREAM_MAX_FAILS — “max_fails” is supported
   * NGX_HTTP_UPSTREAM_FAIL_TIMEOUT — “fail_timeout” is supported
   * NGX_HTTP_UPSTREAM_DOWN — “down” is supported
   * NGX_HTTP_UPSTREAM_BACKUP — “backup” is supported
   * NGX_HTTP_UPSTREAM_MAX_CONNS — “max_conns” is supported
* host — the name of an upstream
* file_name, line — the name of the configuration file and the line where the upstream block is located
* port and no_port — unused by explicit upstreams
* shm_zone — a shared memory zone used by this upstream, if any
* peer — an object that holds generic methods for initializing upstream configuration:

```
typedef struct {
    ngx_http_upstream_init_pt        init_upstream;
    ngx_http_upstream_init_peer_pt   init;
    void                            *data;
} ngx_http_upstream_peer_t;
```

A module that implements a load balancing algorithm must set these methods and initialize private data. If init_upstream was not initialized during configuration parsing, ngx_http_upstream_module sets it to default ngx_http_upstream_init_round_robin.
   * init_upstream(cf, us) — configuration-time method responsible for initializing a group of servers and initializing the init() method in case of success. A typical load balancing module uses a list of servers in the upstream block to create some efficient data structure that it uses and saves own configuration to the data field.
   * init(r, us) — initializes per-request ngx_http_upstream_peer_t.peer (not to be confused with the ngx_http_upstream_srv_conf_t.peer described above which is per-upstream) structure that is used for load balancing. It will be passed as data argument to all callbacks that deal with server selection.
   
When nginx has to pass a request to another host for processing, it uses a configured load balancing method to obtain an address to connect to. The method is taken from the ngx_http_upstream_peer_t.peer object of type ngx_peer_connection_t:

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

The structure has the following fields:

* sockaddr, socklen, name — address of an upstream server to connect to; this is the output parameter of a load balancing method
* data — per-request load balancing method data; keeps the state of selection algorithm and usually includes the link to upstream configuration. It will be passed as an argument to all methods that deal with server selection (see below)
* tries — allowed number of attempts to connect to an upstream.
* get, free, notify, set_session, and save_session - methods of the load balancing module, see description below

All methods accept at least two arguments: peer connection object pc and the data created by ngx_http_upstream_srv_conf_t.peer.init(). Note that in general case it may differ from pc.data due to “chaining” of load balancing modules.

* get(pc, data) — the method is called when the upstream module is ready to pass a request to an upstream server and needs to know its address. The method is responsible to fill in the sockaddr, socklen, and name fields of ngx_peer_connection_t structure. The return value may be one of:
   * NGX_OK — server was selected
   * NGX_ERROR — internal error occurred
   * NGX_BUSY — there are no available servers at the moment. This can happen due to many reasons, such as: dynamic server group is empty, all servers in the group are in the failed state, all servers in the group are already handling the maximum number of connections or similar.
   * NGX_DONE — this is set by the keepalive module to indicate that the underlying connection was reused and there is no need to create a new connection to the upstream server.
* free(pc, data, state) — the method is called when an upstream module has finished work with a particular server. The state argument is the status of upstream connection completion. This is a bitmask, the following values may be set: NGX_PEER_FAILED — this attempt is considered unsuccessful, NGX_PEER_NEXT — a special case with codes 403 and 404 (see link above), which are not considered a failure. NGX_PEER_KEEPALIVE. Also, tries counter is decremented by this method.
* notify(pc, data, type) — currently unused in the OSS version.
* set_session(pc, data) and save_session(pc, data) — SSL-specific methods that allow to cache sessions to upstream servers. The implementation is provided by the round-robin balancing method.
