
## 以前的理解

> $request_time是 nginx收到第一个字节 到 nginx把所有响应内容放到tcp 发送缓冲区的时间。

很长一段时间，我都觉得上面的说法是正确的。

直到前两天跟同事探讨$request_time，才发现并不完全正确，才把以前的一些疑问解释了，觉得挺有意义，这里记录一下。



request_time对于很多CDN厂商来说是一个十分敏感的变量，某些时候可能涉及客户对于质量的考察，当然也许只是一些不太了解$request_time变量的人。

Nginx文档对于$request_time的定义如下：

> request processing time in seconds with a milliseconds resolution; time elapsed since the first bytes were read from the client

文档只介绍了何时开始计算request_time，没有说何时结束。



## 新的理解

对于开启长连接的请求处理，上面的说法是完全正确的。

但是对于短连接的呢？



大概是16年初，曾经某水果台曾使用我们公司CDN。客户使用几家CDN,对比发现我们的CDN慢速请求（byte_sent/request_time）数较多。

讲道理的话，这时候应该优化了。当然首先想到的肯定是sndbuf，恩，其实很多公司也确实是这么做的。

把sndbuf调整到2M, 果然排名靠前了不少。



这时候售前同学就统计了，说Http 1.0的慢速请求数还是很多。

sndbuf还区分协议的？

显然，http1.0和http1.1的一个主要区别就是http1.0默认是关闭长连接的。



查了下代码，发现 `ngx_http_finalize_connection`中对于是否开启长连接，走了两种不同的路径。

显然，开启了keep_alive，会走ngx_http_set_keepalive分支；但是，如若不开启，keepalive则有两种可能：

1. ngx_http_close_request;
2. ngx_http_set_lingering_close; 显然调用lingering_close的话，会导致打印日志的时间推迟；

```c
    if (!ngx_terminate
         && !ngx_exiting
         && r->keepalive
         && clcf->keepalive_timeout > 0)
    {
        ngx_http_set_keepalive(r);
        return;
    }

    if (clcf->lingering_close == NGX_HTTP_LINGERING_ALWAYS
        || (clcf->lingering_close == NGX_HTTP_LINGERING_ON
            && (r->lingering_close
                || r->header_in->pos < r->header_in->last
                || r->connection->read->ready)))
    {
        ngx_http_set_lingering_close(r);
        return;
    }

    ngx_http_close_request(r, 0);
```

Nginx对于epoll采用的是NGX_USE_GREEDY_EVENT策略（The event filter requires to do i/o operation until EAGAIN: epoll.）。

这种策略导致，在调用`ngx_http_finalize_connection`时，read->ready一直为1，从而调用ngx_http_set_lingering_close关闭连接。

但是这个问题在1.11.13版本，nginx引入epoll对于EPOLL_RDHUP的处理后有所改变，代码如下：

[nginx diff](https://github.com/nginx/nginx/commit/12f436718963f8343e38ad6d0e8f7251c95984cd#diff-fbb307b8718d9152c5dc4563247f5349)

```c
        if (n == 0) { //读取到的字节数为0，表示对端关闭连接
            rev->ready = 0;
            rev->eof = 1;
            return 0;
        }
        if (n > 0) {
#if (NGX_HAVE_EPOLLRDHUP)
            if ((ngx_event_flags & NGX_USE_EPOLL_EVENT)
                && ngx_use_epoll_rdhup)
            {
                if ((size_t) n < size) {
                    if (!rev->pending_eof) {
                        rev->ready = 0;
                    }
                    rev->available = 0;
                }
                return n;
            }
#endif
            if ((size_t) n < size
                && !(ngx_event_flags & NGX_USE_GREEDY_EVENT)) 
            {
                rev->ready = 0;
            }

            return n;
        }
        err = ngx_socket_errno;
        if (err == NGX_EAGAIN || err == NGX_EINTR) {
            ngx_log_debug0(NGX_LOG_DEBUG_EVENT, c->log, err,
                           "recv() not ready");
            n = NGX_AGAIN;

        } else {
            n = ngx_connection_error(c, err, "recv() failed");
            break;
        }
```



所以对于nginx 1.11.13之后的版本，无论是否开启keepalive上述的结论仍然是正确的。

对于之前版本，如果想避免lingering_close，可以在配置项中`lingering_close off`。



## TCP Socket对于linger close的支持



close(l_onoff=0缺省状态):在套接口上不能在发出发送或接收请求;套接口发送缓冲区中的内容被发送到对端.如果描述字引用计数变为0;在发送完发送缓冲区中的数据后,跟以正常的TCP连接终止序列(发送FIN);套接口接受缓冲区中内容被丢弃。

close(l_onoff = 1, l_linger =0):在套接口上不能再发出发送或接受请求,如果描述子引用计数变为0,RST被发送到对端;连接的状态被置为CLOSED(没有TIME_WAIT状态),套接口发送缓冲区和套接口接受缓冲区的数据被丢弃。

close(l_onoff =1, l_linger != 0):在套接口上不能在发出发送或接收请求;套接口发送缓冲区中的内容被发送到对端.如果描述字引用计数变为0;在发送完发送缓冲区中的数据后,跟以正常的TCP连接终止序列(发送FIN);套接口接受缓冲区中内容被丢弃;如果在连接变成CLOSED状态前延滞时间到,那么close返回EWOULDBLOCK错误。