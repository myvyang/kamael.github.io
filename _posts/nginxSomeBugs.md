## 信息泄露一

### refresh头

```
var http = require('http');

http.createServer(function (request, response) {

	// 发送 HTTP 头部
	// HTTP 状态值: 200 : OK
	// 内容类型: text/plain
	response.writeHead(200, {'Content-Type': 'text/plain'});
	response.writeHead(200, {'Refresh': '0;url=member/abc/a.html'});

	// 发送响应数据 "Hello World"
	response.end('Hello World\n');
}).listen(8888);
```

refresh头和location头一样使用，可以控制页面跳转。

### nginx的proxy_redirect功能

```
Syntax:	proxy_redirect default;
proxy_redirect off;
proxy_redirect redirect replacement;
Default:	
proxy_redirect default;
Context:	http, server, location
```

Sets the text that should be changed in the “Location” and “Refresh” header fields of a proxied server response. Suppose a proxied server returned the header field `Location: http://localhost:8000/two/some/uri/`. The directive`proxy_redirect http://localhost:8000/two/ http://frontend/one/;`
will rewrite this string to `Location: http://frontend/one/some/uri/`.


### 补丁一

在 nginx 0.4.4 之前，这个功能实现上存在问题，导致当proxy_redirect语句中存在变量(例如`$host`)时，上面的`Location: http://localhost:8000/two/some/uri/`在改写后，将变成``Location: http://frontend/one/`，后面的路径丢失了。

补丁一(c9098081e2086757d089658bab60f5903af91d33)修复了这个问题。

```
commit c9098081e2086757d089658bab60f5903af91d33
Author: Igor Sysoev <igor@sysoev.ru>
Date:   Tue Sep 26 21:15:52 2006 +0000

    fix proxy_redirect with variable

diff --git a/src/http/modules/ngx_http_proxy_module.c b/src/http/modules/ngx_http_proxy_module.c
index 0d80b314..f4007be8 100644
--- a/src/http/modules/ngx_http_proxy_module.c
+++ b/src/http/modules/ngx_http_proxy_module.c
@@ -1397,8 +1397,11 @@ ngx_http_proxy_rewrite_redirect_vars(ngx_http_request_t *r, ngx_table_elt_t *h,
     e.ip = pr->replacement.vars.lengths;
     e.request = r;

-    for (len = prefix; *(uintptr_t *) e.ip; len += lcode(&e)) {
+    len = prefix + h->value.len - pr->redirect.len;
+
+    while (*(uintptr_t *) e.ip) {
         lcode = *(ngx_http_script_len_code_pt *) e.ip;
+        len += lcode(&e);
     }

     data = ngx_palloc(r->pool, len);
@@ -1418,6 +1421,9 @@ ngx_http_proxy_rewrite_redirect_vars(ngx_http_request_t *r, ngx_table_elt_t *h,
         code(&e);
     }

+    ngx_memcpy(e.pos, h->value.data + prefix + pr->redirect.len,
+               h->value.len - pr->redirect.len - prefix);
+
     h->value.len = len;
     h->value.data = data;
```

### 补丁二

但是补丁一的修复中，多余的`prefix`导致了最终的data的长度，比目标url所需要的空间更大，导致可以读取到url之外的内容。

例如http头为`0;url=member/abc/a.html`，那么`h->value`为`0;url=member/abc/a.html`，`pr->redirect`为`member/`，prefix为`0;url=`的长度。即`h->value.len - pr->redirect.len`已经包含了prefix部分，这里prefix多余了。导致生产的`h->value.len`超过了有效数据的长度。

这个问题在补丁二(82e1933529e3e75840262fb17808fee1f5690793)中修正。

```
commit 82e1933529e3e75840262fb17808fee1f5690793
Author: Igor Sysoev <igor@sysoev.ru>
Date:   Mon Jun 7 14:33:50 2010 +0000

    fix rewritten Refresh header length

diff --git a/src/http/modules/ngx_http_proxy_module.c b/src/http/modules/ngx_http_proxy_module.c
index 2d31d261..a6f8755b 100644
--- a/src/http/modules/ngx_http_proxy_module.c
+++ b/src/http/modules/ngx_http_proxy_module.c
@@ -1765,7 +1765,7 @@ ngx_http_proxy_rewrite_redirect_text(ngx_http_request_t *r, ngx_table_elt_t *h,
         return NGX_DECLINED;
     }

-    len = prefix + pr->replacement.text.len + h->value.len - pr->redirect.len;
+    len = pr->replacement.text.len + h->value.len - pr->redirect.len;

     data = ngx_pnalloc(r->pool, len);
     if (data == NULL) {
@@ -1812,7 +1812,7 @@ ngx_http_proxy_rewrite_redirect_vars(ngx_http_request_t *r, ngx_table_elt_t *h,
     e.ip = pr->replacement.vars.lengths;
     e.request = r;

-    len = prefix + h->value.len - pr->redirect.len;
+    len = h->value.len - pr->redirect.len;

     while (*(uintptr_t *) e.ip) {
         lcode = *(ngx_http_script_len_code_pt *) e.ip;
```

### 影响版本

nginx 0.4.3 - 0.8.1

## CRASH

commit 44d8bc2ff19385cf85c09bffdc21fe7a9eb19976
Author: Igor Sysoev <igor@sysoev.ru>
Date:   Fri Oct 20 19:07:50 2006 +0000

    fix segfault if $server_addr failed

diff --git a/src/http/modules/ngx_http_charset_filter_module.c b/src/http/modules/ngx_http_charset_filter_module.c
index b9c9ade9..cae9bbe5 100644
--- a/src/http/modules/ngx_http_charset_filter_module.c
+++ b/src/http/modules/ngx_http_charset_filter_module.c
@@ -250,6 +250,10 @@ ngx_http_charset_header_filter(ngx_http_request_t *r)
                 vv = ngx_http_get_indexed_variable(r,
                                                charset - NGX_HTTP_CHARSET_VAR);

+                if (vv == NULL || vv->not_found) {
+                    return NGX_ERROR;
+                }
+
+                    return NGX_ERROR;
+                }
+
                 charset = ngx_http_charset_get_charset(charsets, n,
                                                        (ngx_str_t *) vv);
             }
@@ -293,6 +297,10 @@ ngx_http_charset_header_filter(ngx_http_request_t *r)
             vv = ngx_http_get_indexed_variable(r,
                                         source_charset - NGX_HTTP_CHARSET_VAR);

+            if (vv == NULL || vv->not_found) {
+                return NGX_ERROR;
+            }
+
             source_charset = ngx_http_charset_get_charset(charsets, n,
                                                           (ngx_str_t *) vv);
         }
diff --git a/src/http/modules/ngx_http_map_module.c b/src/http/modules/ngx_http_map_module.c
index 0a533c0f..fb8e8ee6 100644
--- a/src/http/modules/ngx_http_map_module.c
+++ b/src/http/modules/ngx_http_map_module.c
@@ -115,6 +115,11 @@ ngx_http_map_variable(ngx_http_request_t *r, ngx_http_variable_value_t *v,

     vv = ngx_http_get_flushed_variable(r, map->index);

+    if (vv == NULL || vv->not_found) {
+        *v = *map->default_value;
+        return NGX_OK;
+    }
+
     len = vv->len;

     if (len && map->hostnames && vv->data[len - 1] == '.') {
diff --git a/src/http/ngx_http_variables.c b/src/http/ngx_http_variables.c
index 12014582..48b46f07 100644
--- a/src/http/ngx_http_variables.c
+++ b/src/http/ngx_http_variables.c
@@ -367,7 +367,7 @@ ngx_http_get_indexed_variable(ngx_http_request_t *r, ngx_uint_t index)
     r->variables[index].valid = 0;
     r->variables[index].not_found = 1;

-    return NULL;
+    return &r->variables[index];
 }
 
## segfault一

```
commit 03011fa512bc04879f17774450712dade3dc77b9
Author: Igor Sysoev <igor@sysoev.ru>
Date:   Fri Dec 15 10:24:57 2006 +0000

    fix segfault when $host is used and
    *) request is "GET http://host" without CR or LF, or timed out
    *) request is "GET      http://host" with a large blank space

diff --git a/src/http/ngx_http_request.c b/src/http/ngx_http_request.c
index 2c1ac956..f6dcd6a5 100644
--- a/src/http/ngx_http_request.c
+++ b/src/http/ngx_http_request.c
@@ -1104,7 +1104,9 @@ ngx_http_alloc_large_header_buffer(ngx_http_request_t *r,

         if (r->host_start) {
             r->host_start = new + (r->host_start - old);
-            r->host_end = new + (r->host_end - old);
+            if (r->host_end) {
+                r->host_end = new + (r->host_end - old);
+            }
         }

         if (r->port_start) {
diff --git a/src/http/ngx_http_variables.c b/src/http/ngx_http_variables.c
index c31defc4..ab43638f 100644
--- a/src/http/ngx_http_variables.c
+++ b/src/http/ngx_http_variables.c
@@ -678,9 +678,13 @@ ngx_http_variable_host(ngx_http_request_t *r, ngx_http_variable_value_t *v,
             v->data = r->server_name.data;
         }

-    } else {
+    } else if (r->host_end) {
         v->len = r->host_end - r->host_start;
         v->data = r->host_start;
+
+    } else {
+        v->not_found = 1;
+        return NGX_OK;
     }

     v->valid = 1;
     
// 这里 r->host_end 为 0 地址，参与计算属于未定义行为，计算结果是错误的。
```

## segfault二

```
commit 9813a1999c0297210785c064ea2a02a064d58900
Author: Igor Sysoev <igor@sysoev.ru>
Date:   Fri Apr 15 13:50:27 2011 +0000

    fix segfault in IPv6 parsing while processing invalid IPv4 address X.YYYY.Z
    patch by Maxim Dounin

diff --git a/src/core/ngx_inet.c b/src/core/ngx_inet.c
index 7440c280..ac1ca8bf 100644
--- a/src/core/ngx_inet.c
+++ b/src/core/ngx_inet.c
@@ -110,7 +110,7 @@ ngx_inet6_addr(u_char *p, size_t len, u_char *addr)
         }

         if (c == '.' && nibbles) {
-            if (n < 2) {
+            if (n < 2 || digit == NULL) {
                 return NGX_ERROR;
             }
```

在 ngx_inet6_addr 中，构造输入`[a.xxxxxxx.a]`，则

```
ngx_int_t
ngx_inet6_addr(u_char *p, size_t len, u_char *addr)
{
    u_char      c, *zero, *digit, *s, *d;
    size_t      len4;
    ngx_uint_t  n, nibbles, word;

    if (len == 0) {
        return NGX_ERROR;
    }

    zero = NULL;
    digit = NULL;
    len4 = 0;
    nibbles = 0;
    word = 0;
    n = 8;

    if (p[0] == ':') {
        p++;
        len--;
    }

    for (/* void */; len; len--) {
        c = *p++;

        if (c == ':') {                       // 直接跳过
            if (nibbles) {
                digit = p;
                len4 = len;
                *addr++ = (u_char) (word >> 8);
                *addr++ = (u_char) (word & 0xff);

                if (--n) {
                    nibbles = 0;
                    word = 0;
                    continue;
                }

            } else {
                if (zero == NULL) {
                    digit = p;
                    len4 = len;
                    zero = addr;
                    continue;
                }
            }

            return NGX_ERROR;
        }

        if (c == '.' && nibbles) {           // 第一次跳过，后面++nibbles后再来执行
            if (n < 2) {
                return NGX_ERROR;
            }

            word = ngx_inet_addr(digit, len4 - 1);  // 第二次循环会执行这里，digit = NULL, len4 - 1 下溢了。这里会segfault
            if (word == INADDR_NONE) {
                return NGX_ERROR;
            }

            word = ntohl(word);
            *addr++ = (u_char) ((word >> 24) & 0xff);
            *addr++ = (u_char) ((word >> 16) & 0xff);
            n--;
            break;
        }

        if (++nibbles > 4) {                 // ++nibbles == 1
            return NGX_ERROR;
        }
        
        ...

        c |= 0x20;

        if (c >= 'a' && c <= 'f') {          // 不重要，continue
            word = word * 16 + (c - 'a') + 10;
            continue;
        }

        return NGX_ERROR;
    }
```

## 越界读取

```
commit 80e3cba5fdf8b0af17bc95a20e5eeaf13e232490
Author: Maxim Dounin <mdounin@mdounin.ru>
Date:   Mon Nov 14 13:38:02 2011 +0000

    Reverted incorrect change in internal md5 (part of r3928).

diff --git a/src/core/ngx_md5.c b/src/core/ngx_md5.c
index 09a93991..440c75bc 100644
--- a/src/core/ngx_md5.c
+++ b/src/core/ngx_md5.c
@@ -47,7 +47,8 @@ ngx_md5_update(ngx_md5_t *ctx, const void *data, size_t size)
             return;
         }

-        data = ngx_cpymem(&ctx->buffer[used], data, free);
+        ngx_memcpy(&ctx->buffer[used], data, free);
+        data = (u_char *) data + free;
         size -= free;
         (void) ngx_md5_body(ctx, ctx->buffer, 64);
     }
```

原代码：

```
#define ngx_cpymem(dst, src, n)   (((u_char *) memcpy(dst, src, n)) + (n))

void
ngx_md5_update(ngx_md5_t *ctx, const void *data, size_t size)
{
    size_t  used, free;

    used = (size_t) (ctx->bytes & 0x3f);
    ctx->bytes += size;

    if (used) {
        free = 64 - used;

        if (size < free) {
            ngx_memcpy(&ctx->buffer[used], data, size);
            return;
        }

        data = ngx_cpymem(&ctx->buffer[used], data, free);
        size -= free;
        (void) ngx_md5_body(ctx, ctx->buffer, 64);
    }

    if (size >= 64) {
        data = ngx_md5_body(ctx, data, size & ~(size_t) 0x3f);
        size &= 0x3f;
    }

    ngx_memcpy(ctx->buffer, data, size);
}
```

问题出在

```
SYNOPSIS         top

       #include <string.h>

       void *memcpy(void *dest, const void *src, size_t n);

RETURN VALUE         top

       The memcpy() function returns a pointer to dest.
```

这个地方需要返回src的。返回dest导致取到的用来计算md5的值是越界读取到的值，计算错误。

## 内存覆盖

```
commit 6c3c3bbe4254b2f2d65e4a27ed2618b37391606c
Author: Igor Sysoev <igor@sysoev.ru>
Date:   Tue Jul 19 11:24:16 2011 +0000

    fix segfault if cache key is larger than upstream buffer size
    patch by Lanshun Zhou

diff --git a/src/http/ngx_http_upstream.c b/src/http/ngx_http_upstream.c
index f9f09cb3..ad5b449e 100644
--- a/src/http/ngx_http_upstream.c
+++ b/src/http/ngx_http_upstream.c
@@ -661,6 +661,15 @@ ngx_http_upstream_cache(ngx_http_request_t *r, ngx_http_upstream_t *u)

         ngx_http_file_cache_create_key(r);

+        if (r->cache->header_start >= u->conf->buffer_size) {
+            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
+                "cache key too large, increase upstream buffer size %uz",
+                u->conf->buffer_size);
+
+            r->cache = NULL;
+            return NGX_DECLINED;
+        }
+
         switch (ngx_http_test_predicates(r, u->conf->cache_bypass)) {

         case NGX_ERROR:
```

在 ngx_http_file_cache_create_key 中给 cache->header_start 赋值。这里len为key的长度，默认的可以为`$scheme$proxy_host$request_uri`。这里我们可以控制 request_uri 很长，导致 cache->header_start 超过 u->conf->buffer_size(4096)。

```
void
ngx_http_file_cache_create_key(ngx_http_request_t *r)
{
    size_t             len;
    ngx_str_t         *key;
    ngx_uint_t         i;
    ngx_md5_t          md5;
    ngx_http_cache_t  *c;

    c = r->cache;

    len = 0;

    ngx_crc32_init(c->crc32);
    ngx_md5_init(&md5);

    key = c->keys.elts;
    for (i = 0; i < c->keys.nelts; i++) {
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http cache key: \"%V\"", &key[i]);

        len += key[i].len;

        ngx_crc32_update(&c->crc32, key[i].data, key[i].len);
        ngx_md5_update(&md5, key[i].data, key[i].len);
    }

    c->header_start = sizeof(ngx_http_file_cache_header_t)
                      + sizeof(ngx_http_file_cache_key) + len + 1;

    ngx_crc32_final(c->crc32);
    ngx_md5_final(c->key, &md5);
}
```

在 ngx_http_upstream_process_header 中

```
#if (NGX_HTTP_CACHE)

        if (r->cache) {
            u->buffer.pos += r->cache->header_start;
            u->buffer.last = u->buffer.pos;
        }
#endif
    }

    for ( ;; ) {

        n = c->recv(c, u->buffer.last, u->buffer.end - u->buffer.last);
```

一般的，`u->buffer.pos = u->buffer.start, u->buffer.end - u->buffer.start = 4096(buffer_size)`。那么这里recv的参数`u->buffer.end - u->buffer.last`将下溢。而`u->buffer.last`超出了当前buffer块的地址。新接收的数据将覆盖后面的buffer快，类似堆溢出。

### 复现

nginx.conf配置只需要配了proxy_cache就行，参数无所谓。

```
    proxy_cache_path cache levels=1:2 keys_zone=one:10m;

    server {
        listen       8088;
        server_name  localhost;

        location /a/ {
            root   html;
            proxy_pass http://127.0.0.1:8888;
            proxy_cache one;
            index  index.html index.htm;
        }
```

请求URL

```
http://127.0.0.1:8088/a/a[a*4096]
```

### 过多传输BUG

```
commit f48b45119557790fc26f7d4b3d081ea6f27c9301
Author: Maxim Dounin <mdounin@mdounin.ru>
Date:   Thu Aug 18 15:27:57 2011 +0000

    Correctly set body if it's preread and there are extra data.

    Previously all available data was used as body, resulting in garbage after
    real body e.g. in case of pipelined requests.  Make sure to use only as many
    bytes as request's Content-Length specifies.

diff --git a/src/http/ngx_http_request_body.c b/src/http/ngx_http_request_body.c
index be311a61..57c201b9 100644
--- a/src/http/ngx_http_request_body.c
+++ b/src/http/ngx_http_request_body.c
@@ -143,6 +143,7 @@ ngx_http_read_client_request_body(ngx_http_request_t *r,

             r->header_in->pos += (size_t) r->headers_in.content_length_n;
             r->request_length += r->headers_in.content_length_n;
+            b->last = r->header_in->pos;

             if (r->request_body_in_file_only) {
                 if (ngx_http_write_request_body(r, rb->bufs) != NGX_OK) {
```

`ngx_http_write_request_body` 写入文件的字节数，是按照`b->last - b->pos`来的。

```
ssize_t
ngx_write_chain_to_file(ngx_file_t *file, ngx_chain_t *cl, off_t offset,
    ngx_pool_t *pool)
{
    u_char        *prev;
    size_t         size;
    ssize_t        n;
    ngx_array_t    vec;
    struct iovec  *iov, iovs[NGX_IOVS];

    /* use pwrite() if there is the only buf in a chain */

    if (cl->next == NULL) {
        return ngx_write_file(file, cl->buf->pos,
                              (size_t) (cl->buf->last - cl->buf->pos),
                              offset);
    }
```

这样会导致将多于 content_length 的垃圾数据写入了请求。

### 1.1.9 停止


### 1.9.9 的一堆BUG

`r->name_resend_queue`初始化在

```
typedef struct ngx_rbtree_node_s  ngx_rbtree_node_t;

struct ngx_rbtree_node_s {
    ngx_rbtree_key_t       key;
    ngx_rbtree_node_t     *left;
    ngx_rbtree_node_t     *right;
    ngx_rbtree_node_t     *parent;
    u_char                 color;
    u_char                 data;
};

typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};

typedef struct {
    ...
    ngx_rbtree_node_t         addr_sentinel;
    ngx_queue_t               name_resend_queue;
    ngx_queue_t               addr_resend_queue;

    ...
} ngx_resolver_t;


#define ngx_queue_init(q)                                                     \
    (q)->prev = q;                                                            \
    (q)->next = q


ngx_resolver_t *
ngx_resolver_create(ngx_conf_t *cf, ngx_str_t *names, ngx_uint_t n)
{
        ...
        ngx_resolver_t        *r;
        ...
        r = ngx_calloc(sizeof(ngx_resolver_t), cf->log);
        ...
        
        ...
        ngx_queue_init(&r->name_resend_queue);
```

#### 修复1

```
commit c44fd4e837f979912749a5a19490ccb9b46398d3
Author: Roman Arutyunyan <arut@nginx.com>
Date:   Tue Jan 26 16:46:18 2016 +0300

    Resolver: fixed possible segmentation fault on DNS format error.

diff --git a/src/core/ngx_resolver.c b/src/core/ngx_resolver.c
index 70138851..14bd2b31 100644
--- a/src/core/ngx_resolver.c
+++ b/src/core/ngx_resolver.c
@@ -1312,7 +1312,7 @@ ngx_resolver_process_response(ngx_resolver_t *r, u_char *buf, size_t n)
         times = 0;

         for (q = ngx_queue_head(&r->name_resend_queue);
-             q != ngx_queue_sentinel(&r->name_resend_queue) || times++ < 100;
+             q != ngx_queue_sentinel(&r->name_resend_queue) && times++ < 100;
              q = ngx_queue_next(q))
         {
            rn = ngx_queue_data(q, ngx_resolver_node_t, queue);
            qident = (rn->query[0] << 8) + rn->query[1];
```

修复之前，循环列表只有一个节点头，和一个数据节点。但是`q != ngx_queue_sentinel(&r->name_resend_queue)`失效，导致节点头也被作为数据节点使用。`rn =(u_char *) q - 8`，rn的地址不是一个`ngx_resolver_node_t`。`rn->query = (u_char *) q - 8 + 192`，这个地方不一定是一个合法的指针地址，可能导致非法读。

```
typedef struct {
    /* PTR: resolved name, A: name to resolve */
    u_char                   *name;
    ngx_queue_t               queue;
    ...
    u_char                   *query; // offset: 192
    ...
} ngx_resolver_node_t;

#define ngx_queue_data(q, type, link)                                         \
    (type *) ((u_char *) q - offsetof(type, link))

ngx_queue_data(q, ngx_resolver_node_t, queue);
    =>
        (ngx_resolver_node_t *) ((u_char *) q - offsetof(ngx_resolver_node_t, queue))
    =>
        (ngx_resolver_node_t *) ((u_char *) q - 8)

rn->query
    =>
        (u_char *) q - 8 + 192

```

### 修复2

```
commit 4b581a7c21e4328d059bf400a059c0458fc9f806
Author: Ruslan Ermilov <ru@nginx.com>
Date:   Tue Jan 26 16:46:31 2016 +0300

    Resolver: fixed crashes in timeout handler.

    If one or more requests were waiting for a response, then after
    getting a CNAME response, the timeout event on the first request
    remained active, pointing to the wrong node with an empty
    rn->waiting list, and that could cause either null pointer
    dereference or use-after-free memory access if this timeout
    expired.

    If several requests were waiting for a response, and the first
    request terminated (e.g., due to client closing a connection),
    other requests were left without a timeout and could potentially
    wait indefinitely.

    This is fixed by introducing per-request independent timeouts.
    This change also reverts 954867a2f0a6 and 5004210e8c78.
```

更新代码太多不贴。写下漏洞原理。

首先在nginx发起dns请求的时候，绑定了两个handler，`ngx_resolver_read_response`和`ngx_resolver_timeout_handler`，一个在接收到DNS响应的时候触发，一个在DNS响应超时的时候触发。

如果我们可以控制DNS服务器的响应，我们可以构造一个CNAME的响应，这个会导致`ngx_resolver_process_a`中的如下代码执行：

```
        ctx = rn->waiting;
        rn->waiting = NULL;
```

rn 为 ngx_resolver_node_t 类型，表示一个DNS查询节点。由于DNS查询是异步的，nginx会将发出的DNS请求对应的节点保存在一个红黑树中。ctx 为 ngx_resolver_ctx_s 类型，表示一次DNS查询的上下文，一个 ngx_resolver_node_t 节点可以对应多个 ngx_resolver_ctx_s 上下文。当两次DNS查询的内容是一样时，会将多个ctx挂在同一个rn的链表里：

ngx_resolve_name_locked 中：

```
        if (rn->waiting) {

            ctx->next = rn->waiting;
            rn->waiting = ctx;
            ctx->state = NGX_AGAIN;

            return NGX_AGAIN;
        }
```

经过上面的CNAME响应后，rn->waiting 为空。但是rn对应的ngx_resolver_timeout_handler还是会在超时时触发。

ngx_resolve_name_locked 中：

```
    if (ctx->event == NULL) {
        ctx->event = ngx_resolver_calloc(r, sizeof(ngx_event_t));
        if (ctx->event == NULL) {
            goto failed;
        }

        ctx->event->handler = ngx_resolver_timeout_handler;
        ctx->event->data = rn;
        ctx->event->log = r->log;
        rn->ident = -1;

        ngx_add_timer(ctx->event, ctx->timeout);
    }
```

然后

```
static void
ngx_resolver_timeout_handler(ngx_event_t *ev)
{
    ngx_resolver_ctx_t   *ctx, *next;
    ngx_resolver_node_t  *rn;

    rn = ev->data;
    ctx = rn->waiting;
    rn->waiting = NULL;

    do {
        ctx->state = NGX_RESOLVE_TIMEDOUT;
        next = ctx->next;

        ctx->handler(ctx);

        ctx = next;
    } while (ctx);
}
```

这里取到的ctx为NULL，导致异常。

如果在超时之前，第一个请求异常退出了(客户端中断连接)，会触发`ngx_http_upstream_rd_check_broken_connection`，进而触发`ngx_resolve_name_done`：

```
    if (ctx->event && ctx->event->timer_set) {
        ngx_del_timer(ctx->event);
    }
```

删除超时事件。这会导致其他挂在同一rn上的请求的ctx都不会超时，因为当后一个请求需要查询的DNS在前面已经在查询中时，逻辑不会走到设置超时的那一步。所有的ctx的超时的触发依赖第一个ctx的超时。这个在`ngx_resolver_timeout_handler`中也可以看出来。

见`ngx_resolve_name_locked`

```
static ngx_int_t
ngx_resolve_name_locked(ngx_resolver_t *r, ngx_resolver_ctx_t *ctx)
{
    uint32_t              hash;
    ngx_int_t             rc;
    ngx_uint_t            naddrs;
    ngx_addr_t           *addrs;
    ngx_resolver_ctx_t   *next;
    ngx_resolver_node_t  *rn;

    ngx_strlow(ctx->name.data, ctx->name.data, ctx->name.len);

    hash = ngx_crc32_short(ctx->name.data, ctx->name.len);

    rn = ngx_resolver_lookup_name(r, &ctx->name, hash);
    // 查询是否已有同名的DNS查询

    if (rn) {
        if (rn->valid >= ngx_time()) {
            ...
            ...
                return NGX_AGAIN;
            ...
            ...
```

修复方式为，给每一个ctx独立的超时事件，然后在超时时分别处理。这样就不会存在上述问题了。`rn->waiting`链表只剩下触发handler的作用。


### 修复3

```
commit 4016e6b1da4fbf9c45963211791be124cd7ffb8f
Author: Ruslan Ermilov <ru@nginx.com>
Date:   Tue Jan 26 16:46:38 2016 +0300

    Resolver: fixed CNAME processing for several requests.

    When several requests were waiting for a response, then after getting
    a CNAME response only the last request was properly processed, while
    others were left waiting.
```

这个还是比较隐蔽的，需要代码走到特定地方。

先在`ngx_resolver_process_a`接收到一个CNAME的响应

```
    if (cname) {
        ...
        ctx = rn->waiting;
        rn->waiting = NULL;

        if (ctx) {
            ctx->name = name;

            (void) ngx_resolve_name_locked(r, ctx);
        }
```

当有多个请求同时到达服务器时，服务器将多个ctx挂在rn->waiting上。经过上面的代码，rn->waiting为空，然后执行`ngx_resolve_name_locked(r, ctx)`。

在`ngx_resolve_name_locked`中，需要走如下路线：

```
    rn = ngx_resolver_lookup_name(r, &ctx->name, hash);
    if (rn) {
        if (rn->valid >= ngx_time()) {
            ...
            if (naddrs) {
                ...
                ctx->next = rn->waiting;
                rn->waiting = NULL;

```

到这里之前，ctx为之前的一串。如果我们让CNAME的结果为原域名，将会导致rn即为我们之前将rn->waiting设置为空的那个。这么一来，就导致了ctx->next上的那一串ctx被扔掉了。导致他们永远都不会被处理。

而修复方案，则是将ctx用ctx链中最后一个来代替，这样ctx->next肯定为空，不会导致断链。实质上将原来的链头插入变成链尾插入。

### 修复4

```
commit a3d42258d97ebd0b638c20976654d3edfbaf943f
Author: Roman Arutyunyan <arut@nginx.com>
Date:   Tue Jan 26 16:46:59 2016 +0300

    Resolver: fixed use-after-free memory accesses with CNAME.

    When several requests were waiting for a response, then after getting
    a CNAME response only the last request's context had the name updated.
    Contexts of other requests had the wrong name.  This name was used by
    ngx_resolve_name_done() to find the node to remove the request context
    from.  When the name was wrong, the request could not be properly
    cancelled, its context was freed but stayed linked to the node's waiting
    list.  This happened e.g. when the first request was aborted or timed
    out before the resolving completed.  When it completed, this triggered
    a use-after-free memory access by calling ctx->handler of already freed
    request context.  The bug manifests itself by
    "could not cancel <name> resolving" alerts in error_log.

    When a request was responded with a CNAME, the request context kept
    the pointer to the original node's rn->u.cname.  If the original node
    expired before the resolving timed out or completed with an error,
    this would trigger a use-after-free memory access via ctx->name in
    ctx->handler().

    The fix is to keep ctx->name unmodified.  The name from context
    is no longer used by ngx_resolve_name_done().  Instead, we now keep
    the pointer to resolver node to which this request is linked.
    Keeping the original name intact also improves logging.
```

`ngx_resolver_process_a`中

```
        if (ctx) {
            ctx->name = name;

            (void) ngx_resolve_name_locked(r, ctx);
        }
```

这里，只将第一个ctx的name修改了。

`ngx_resolve_name_done`中

```
        rn = ngx_resolver_lookup_name(r, &ctx->name, hash);

        if (rn) {
            p = &rn->waiting;
            w = rn->waiting;
        ...
        if (ctx->event) {
            ngx_resolver_free_locked(r, ctx->event);
        }
        ...
```

直接拿ctx->name查询rn。这就导致了后面的没改过name的ctx被忽略了(rn这时候为空)。

如果强行中断客户端，将导致触发`ngx_http_upstream_rd_check_broken_connection`，这个会将整个request结构销毁。但是在`ngx_resolve_name_done`中

```
        rn = ngx_resolver_lookup_name(r, &ctx->name, hash);

        if (rn) {
            p = &rn->waiting;
            w = rn->waiting;

            while (w) {
                if (w == ctx) {
                    *p = w->next;

                    goto done;
                }

                p = &w->next;
                w = w->next;
            }
        }
```

这一段是为了把当前ctx从rn-waiting链表中摘出来。但是由于ctx->name没有更新，查找到的rn的rn->waiting为空，导致摘链失败。

然后当A记录查询得到响应时，在`ngx_resolver_process_a`中

```
        next = rn->waiting;
        rn->waiting = NULL;

        /* unlock name mutex */

        while (next) {
            ctx = next;
            ctx->state = NGX_OK;
            ctx->naddrs = naddrs;

            if (addrs == NULL) {
                ctx->addrs = &ctx->addr;
                ctx->addr.sockaddr = (struct sockaddr *) &ctx->sin;
                ctx->addr.socklen = sizeof(struct sockaddr_in);
                ngx_memzero(&ctx->sin, sizeof(struct sockaddr_in));
                ctx->sin.sin_family = AF_INET;
                ctx->sin.sin_addr.s_addr = rn->u.addr;

            } else {
                ctx->addrs = addrs;
            }

            next = ctx->next;

            ctx->handler(ctx);
        }
```

这里的某个ctx就是已经回收的内存指针，导致UAF。

### 修复4

```
commit fe89d99796d42b86816e17d9c87ab16964768024
Author: Ruslan Ermilov <ru@nginx.com>
Date:   Tue Jan 26 16:47:14 2016 +0300

    Resolver: limited CNAME recursion.

    Previously, the recursion was only limited for cached responses.
```

没什么可说的。

### 越界读

```
commit 66be8c6608edd5d79d24581749e4621df465610d
Author: Valentin Bartenev <vbart@nginx.com>
Date:   Wed May 11 17:55:20 2016 +0300

    Core: fixed port handling in ngx_parse_inet6_url().

    This fixes buffer over-read when no port is specified in cases
    similar to 5df5d7d771f6, and catches missing port separator.

diff --git a/src/core/ngx_inet.c b/src/core/ngx_inet.c
index 33b303dd..d5b7cf96 100644
--- a/src/core/ngx_inet.c
+++ b/src/core/ngx_inet.c
@@ -861,7 +861,12 @@ ngx_parse_inet6_url(ngx_pool_t *pool, ngx_url_t *u)
             last = uri;
         }

-        if (*port == ':') {
+        if (port < last) {
+            if (*port != ':') {
+                u->err = "invalid host";
+                return NGX_ERROR;
+            }
+
             port++;

             len = last - port;
```

如果URI本身很短，会导致*port超出了URI的长度，读到了垃圾数据。如果
