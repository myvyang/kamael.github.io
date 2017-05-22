
### O_NOATIME

读文件的时候不修改access time.

```
       O_NOATIME (since Linux 2.6.8)
              Do not update the file last access time (st_atime in the
              inode) when the file is read(2).

              This flag can be employed only if one of the following
              conditions is true:

              *  The effective UID of the process matches the owner UID of
                 the file.

              *  The calling process has the CAP_FOWNER capability in its
                 user namespace and the owner UID of the file has a mapping
                 in the namespace.

              This flag is intended for use by indexing or backup programs,
              where its use can significantly reduce the amount of disk
              activity.  This flag may not be effective on all filesystems.
              One example is NFS, where the server maintains the access
              time.
```

### script机制

```
typedef struct {
    ngx_http_script_code_pt         code;
    uintptr_t                       len;
} ngx_http_script_copy_code_t;


typedef struct {
    ngx_http_script_code_pt         code;
    uintptr_t                       index;
} ngx_http_script_var_code_t;


typedef struct {
    ngx_http_script_code_pt         code;
    ngx_http_set_variable_pt        handler;
    uintptr_t                       data;
} ngx_http_script_var_handler_code_t;


typedef struct {
    ngx_http_script_code_pt          code;
    uintptr_t                        n;
} ngx_http_script_copy_capture_code_t;
```

例如`http://$host:8088/one/`，被拆为`http://`,`$host`,`:8088/one/`三部分，第一部分被直接复制。第二部分运行时替换成对应的值。第三部分被直接复制。

直接复制的时候，内存布局为

lengths

```
ngx_http_script_copy_len_code
len
```

values

```
ngx_http_script_copy_code
len
data
```

替换变量时，内存布局为

lengths

```
ngx_http_script_copy_var_len_code
index
```

values

```
ngx_http_script_copy_var_code
index
```

然后用`NULL`结束。

其中`ngx_http_script_copy_len_code`,`ngx_http_script_copy_code`都是`ngx_http_script_code_pt`类型的函数。 

### http请求流程

```
*_process_events
  ngx_event_accept
    ngx_http_init_connection
      (rev->handler = ngx_http_init_request)
*_process_events
  ngx_http_init_request
    (rev->handler = ngx_http_process_request_line)
    ngx_http_process_request_line
      ngx_http_read_request_header
      ngx_http_parse_request_line
      (rev-handler = ngx_http_process_request_headers)
      (ngx_http_process_request_headers)
*_process_events
  ngx_http_process_request_headers
    (rev->handler = ngx_http_request_handler)
    (c->write->handler = ngx_http_request_handler)
    (r->read_event_handler = ngx_http_block_read)
    ngx_http_core_run_phases
      phase_engine.handlers[r->phase_handler].checker(r, &ph[r->phase_handler])
      (进入分阶段的处理)
        ngx_http_core_content_phase
          ngx_http_proxy_handler
            ngx_http_read_client_request_body
              ngx_http_upstream_init
                ngx_http_upstream_connect
                  ngx_http_upstream_send_request
                  (c->read->handler = ngx_http_upstream_process_header)
*_process_events
  ngx_http_request_handler
*_process_events
  ngx_http_upstream_process_header
    ngx_http_upstream_send_response
      ngx_http_upstream_process_body
        ngx_event_pipe
          ngx_event_pipe_read_upstream
*_process_events
  ngx_http_keepalive_handler
```

### http2请求流程

```
*_process_events
  ngx_event_accept
    ngx_http_init_connection
      (rev->handler = ngx_http_ssl_handshake)
*_process_events
  ngx_http_ssl_handshake
    ngx_ssl_create_connection
        ngx_ssl_handshake
            SSL_do_handshake
            c->read->handler = ngx_ssl_handshake_handler;
            c->write->handler = ngx_ssl_handshake_handler;
    c->ssl->handler = ngx_http_ssl_handshake_handler
    ngx_http_ssl_handshake_handler(c)
*_process_events
    ngx_ssl_handshake_handler
        ngx_ssl_handshake
        c->ssl->handler(ngx_http_ssl_handshake_handler)
            ngx_http_v2_init
                rev->handler = ngx_http_v2_read_handler;
                c->write->handler = ngx_http_v2_write_handler;
                ngx_http_v2_read_handler
```

### gzip机制存在bug

`accept-encoding: agzipgzip, `被视为有效的gzip头。

### resolver流程

```
ngx_resolver_ctx_t * ngx_resolve_start(ngx_resolver_t *r, ngx_resolver_ctx_t *temp)

ngx_resolve_name(ngx_resolver_ctx_t *ctx)
  ngx_resolve_name_locked
  (有缓存)
    ngx_http_upstream_resolve_handler
  (无缓存)
    ngx_resolver_create_name_query
    ngx_resolver_send_query
      uc->connection->read->handler = ngx_resolver_read_response
      ngx_send
      ctx->event->handler = ngx_resolver_timeout_handler
ngx_resolver_read_response
  ngx_resolver_process_response
    ngx_resolver_process_a
      ngx_http_upstream_resolve_handler
        ngx_resolve_name_done
          ngx_resolver_expire
    ngx_resolver_process_ptr
```













