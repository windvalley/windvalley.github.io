---
draft: false
title: "Nginx主要的超时时间指令详解"
date: 2016-06-02T10:42:15+08:00
lastmod: 2016-06-02T10:42:15+08:00
authors:
    - windvalley
categories:
    - OpenResty
tags:
    - nginx
    - timeout
---

## 模块`ngx_http_core_module`下的超时时间指令

> Syntax:    client_body_timeout time; <br>
> Default:   client_body_timeout 60s; <br>
> Context:   http, server, location

适用于有请求体的客户端请求, 比如`POST`, 表示nginx读取客户端请求体的超时时间, 默认60秒超时.这个时间是指两次成功读取操作的间隔时间, 并不包括整个请求体数据的传输时间.

如果在设置的超时时间内用户没有传输任何请求体数据, 则算超时, 中断连接, 并返回给用户`408`状态码, 表示请求超时.

> Syntax:    client_header_timeout time; <br>
> Default:    client_header_timeout 60s; <br>
> Context:    http, server

如果用户端没有在指定的超时时间内完成整个请求头的传输, 则断开连接, 给用户返回`408`状态码, 表示请求超时.

> Syntax:    send_timeout time; <br>
> Default:   send_timeout 60s; <br>
> Context:   http, server, location

nginx响应给客户端数据的超时时间, 注意这个时间指的是nginx两次成功的写操作的间隔时间, 不包括响应数据的传输时间.

如果在这个指定时间内, 客户端没有收到任何数据(没有给nginx反馈确认收到数据), 则nginx断开连接.

> Syntax:    keepalive_timeout timeout [header_timeout]; <br>
> Default:    keepalive_timeout 75s; <br>
> Context:    http, server, location

第一个参数`timeout`, 表示nginx在和一个支持`keep-alive`的客户端的连接中, 当没有数据传输后不马上断开连接, 等待超时时间到了如果依然没有数据传输, 则主动断开连接, 很多情况是在nginx主动断开连接前用户端已经主动断开了连接.

如果第一个`timeout`参数设置为`0`, 则表示即使客户端要求`keep-alive`连接, nginx也会在响应完毕后马上关闭连接, 响应头中有`Connection:close`.

第二个参数`header_timeout`是可选参数, 如果设置的话nginx会在响应头中设置`Keep-Alive:timeout=time`的字段, 表示希望浏览器按照这个超时时间主动断开连接.

以上2个参数可设置不同的超时时间.

另外, `Keep-Alive:timeout=time`响应头可以被Mozilla和Konqueror系列浏览器识别, 而MSIE浏览器会按照自身默认设置的60秒超时时间来主动关闭keep-alive连接.

Mozilla: Mozilla是基金会名称, Firefox浏览器是其产品, 简单说两者是公司和公司研发的产品的区别.在此处应该特指firefox浏览器.

Konqueror: 是KDE桌面系统下的浏览器, 主要用于Linux和BSD家族的操作系统.Safari浏览器的渲染引擎WebKit就是基于Konqueror的渲染引擎KHTML改进的.而Chrome是基于WebKit的.

> Syntax:    reset_timedout_connection on | off; <br>
> Default:    reset_timedout_connection off; <br>
> Context:    http, server, location

对于超时的TCP连接, nginx发起RST重置, 释放socket占用的内存.

这个主要是为了避免服务端已经关闭的socket长时间处于`FIN_WAIT1`状态(由于客户端没有发送ack或发送的ack丢失), 导致浪费内存.

这里需要注意的是超时的keep-alive连接还是会正常关闭的.

> Syntax:    resolver_timeout time; <br>
> Default:    resolver_timeout 30s; <br>
> Context:    http, server, location

域名解析的超时时间.

> Syntax:    lingering_close off | on | always; <br>
> Default:    lingering_close on; <br>
> Context:    http, server, location <br>
> This directive appeared in versions 1.1.0 and 1.0.6.

控制nginx如何关闭和客户端的tcp连接.整体来说`lingering_close`指令主要是为了防止nginx主动关闭tcp连接时, 意外发送`RST`导致客户端异常或者说客户端丢失数据的情况出现.

默认值是`on`, 表明nginx在关闭tcp连接前, 如果nginx的启发式算法预测客户端存在额外发送数据的可能性, 就会先等待并处理客户端发送过来的额外数据(处理在这里是忽略, 新来的请求不会再去响应了, 直接忽略).

值是`always`的时候, 表明nginx在任何情况下都会等待并处理(忽略)客户端额外发送过来的数据.

值是`off`的时候, 表明nginx会直接关闭tcp连接(正常时发送`fin`或异常时发送`rst`), 在正常场景下不建议设置为`off`.

> Syntax:    lingering_time time; <br>
> Default:    lingering_time 30s; <br>
> Context:    http, server, location

当`lingering_close`生效的时候, 此指令才会起作用.

这个指令设置的秒数, 表明在关闭tcp连接前nginx读取并且忽略客户端发送过来的额外数据的最大时间, 如果超过时间即使客户端还有数据在发送也会直接关闭tcp连接.

> Syntax:    lingering_timeout time; <br>
> Default:    lingering_timeout 5s; <br>
> Context:    http, server, location

当`lingering_close`生效的时候, 此指令才会起作用.

这个指令设置的超时时间表示nginx在主动关闭tcp连接前, 等待客户端发送额外数据的最大时间, 在这个时间内没有收到客户端任何数据, 则关闭tcp连接.如果在这个时间内收到了客户端额外数据, 则读取并忽略这些数据, 然后再等待, 循环, 直到达到`lingering_timeout`超时时间或者`lingering_time`设置的时间, 则nginx才发起关闭tcp连接.

另外说下nginx会主动关闭连接的场景:

- `keepalive_timeout`设置的超时时间到的时候
- `keepalive_timeout`设置为0的情况, nginx只要响应完成就会发起关闭tcp连接
- 客户端不支持keepalive的情况, 也就是客户端的请求头中没有`Connection:keep-alive`

## 模块ngx_http_proxy_module下的超时时间指令

模块名字中的proxy, 指的是nginx作为代理服务器的意思, 所以下面所有指令的proxy都是指的nginx本身.

> Syntax:    proxy_cache_lock on | off; <br>
> Default:    proxy_cache_lock off; <br>
> Context:    http, server, location <br>
> This directive appeared in version 1.1.12.

若开启, 表示对同一个文件的多个客户端请求(包括range请求), nginx只允许一个请求去上游服务器请求数据, 并且从上游服务器拉取整个文件响应给客户端, 同时开始在nginx本地填充缓存文件.其他请求会排队阻塞, 等待第一个请求在nginx本地完成整个文件的缓存后响应给这些请求, 或者如果在`proxy_cache_lock_timeout`指定的时间超时后文件还没有缓存完成, nginx就会允许这些请求去上游服务器获取数据, 但是这些请求不会在nginx本地建立缓存.

> Syntax:    proxy_cache_lock_timeout time; <br>
> Default:    proxy_cache_lock_timeout 5s; <br>
> Context:    http, server, location <br>
> This directive appeared in version 1.1.12.

当超时时间到了后, nginx会把每一个阻塞的请求不做修改的转发到上游服务器, 此处不做修改主要是指如果是range请求, 就保留range头, 只从上游服务器获取所需range大小的数据, 而不是像第一个请求那如果是range请求会把相关请求做修改而从上游服务器获取整个文件.另外这些请求也不会像第一个请求那样在nginx本地创建缓存文件.

由于阻塞用户请求带来的体验很差, 建议这里设置为`100ms`或`200ms`, 这样其他请求也能及时从源站获取数据, 由于大部分是range请求并且当第一个请求的缓存文件完成后可以及时从nginx缓存获取数据, 所以源站压力也不会很大.

> Syntax:    proxy_cache_lock_age time; <br>
> Default:    proxy_cache_lock_age 5s; <br>
> Context:    http, server, location <br>
> This directive appeared in version 1.7.8.

表示第一个请求在nginx本地完成整个文件缓存的最长时间, 如果在指定期限内没有完成整个文件的缓存, 那么nginx会再转发一个请求到上游服务器并在nginx本地重新缓存这个文件.
这个超时时间应该总是设置的比我们预估的完成文件缓存的时间要长一点, 这个根据实际业务情况进行设置.比如, 如果文件在10M以上的比较多, 可以设置为200s.

> Syntax:    proxy_connect_timeout time; <br>
> Default:    proxy_connect_timeout 60s; <br>
> Context:    http, server, location

nginx和上游的一台服务器建立连接的超时时间.需要注意的是这个超时时间通常不可能超过75s.

> Syntax:    proxy_send_timeout time; <br>
> Default:    proxy_send_timeout 60s; <br>
> Context:    http, server, location

nginx向上游某个服务器发送请求的超时时间, 这个时间不包括请求数据的传输时间, 而是指两次成功写操作的间隔时间.如果在指定时间内上游服务器没有收到任何东西, nginx则会断开和这个上游服务器的连接.

> Syntax:    proxy_read_timeout time; <br>
> Default:    proxy_read_timeout 60s; <br>
> Context:    http, server, location

nginx读取上游某个服务器响应的超时时间, 这个时间不包括响应数据的传输时间, 而是指两次成功读操作的间隔时间.如果在指定时间内上游服务器没有发送任何东西过来, nginx则断开和这个上游服务器的连接.

> Syntax:    proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | http_429 | non_idempotent | off ...; <br>
> Default:    proxy_next_upstream error timeout; <br>
> Context:    http, server, location

在上游服务器有多台服务器组成上游服务器池的情况, 这个指令定义的是在什么情况下nginx会把请求传递给下一个上游服务器.

`error`: 在nginx和上游服务器建立连接的时候, 或者建立连接后传递请求到上游服务器的时候, 或者请求后读取上游服务器响应头的时候出现任何错误的情况下.

`timeout`: 在nginx和上游服务器建立连接的时候(proxy_connect_timeout), 或者建立连接后传递请求到上游服务器的时候(proxy_send_timeout), 或者请求后读取上游服务器响应头的时候(proxy_read_timeout)出现超时的情况下.

`invalid_header`: 上游服务器返回给nginx的空的或非法(指的是不符合协议规范)的响应的情况下.

`http_xxx`: 上游服务器返回指定的状态码的情况下.

`non_idempotent`: 正常情况下nginx去上游服务器池请求的时候, 对于非幂等的请求(比如`POST`, `LOCAK`, `PATCH`), 即使出现上述指定的情况, 也不会去下一个服务器请求, 除非把这个参数明确的加上.

`off`: 即使遇到上述场景, 也不会尝试请求下一个服务器.

应该牢记的是在发生以上指定的情况, nginx只有在还没有给客户端返回任何东西的情况下, 才会去尝试请求下一个上游服务器.也就是说如果nginx和上游服务器的通信错误发生在nginx已经响应了一部分东西给客户端的情况下, 是不会去尝试请求下一台上游服务器的.

这个指令除了用来定义什么情况下尝试请求下一个上游服务器, 还定义了什么才算是和上游某台服务器通信的不成功尝试(主要用于`upstream`指令下的`server`的`max_fails`), 其中`error`、`timeout`、`invalid_header`即使不在指令中明确指定, 也会被认为是不成功尝试, 而`http_5xx`和`http_429`只有在指令中明确指定才会被认为是不成功尝试. 另外`http_403`和`http_404`就从来不会被认为是不成功尝试.

> Syntax:    proxy_next_upstream_timeout time; <br>
> Default:    proxy_next_upstream_timeout 0; <br>
> Context:    http, server, location <br>
> This directive appeared in version 1.7.5.

这个指令表示nginx和上游服务器池通信的总的超时时间.如果达到这个指令设置的超时时间, 则nginx不再重试下一个上游服务器, 直接将错误消息响应给客户端.

需要注意的是nginx和上游某台服务器的通信中, 在每次遇到`proxy_connect_timeout`或`proxy_send_timeout`或`proxy_read_timeout`指令所设置的超时时间过期后, 都会检查`proxy_next_upstream_timeout`所设置的超时时间是否已过期, 如果没过期则nginx尝试连接下一个上游服务器, 如果过期了, 则停止继续尝试连接下一台上游服务器.

举一个例子:

如果`proxy_read_timeout`设置为100s, `proxy_next_upstream_timeout`设置为50s, 如果nginx连接一台上游服务器遇到读超时的情况, 会等待100s后超时, 而不是50s.

默认为`0`, 表示不设置nginx尝试上游服务器池总的超时时间, 此时nginx和上游服务器池的总超时时间将由`connect/send/read`所设置的超时时间和上游服务器数量或者`proxy_next_upstream_tries`来综合决定.

> Syntax:    proxy_next_upstream_tries number; <br>
> Default:    proxy_next_upstream_tries 0; <br>
> Context:    http, server, location <br>
> This directive appeared in version 1.7.5.

上游某台服务器出现错误时, nginx尝试将请求转发给下一台上游服务器的次数, 如果值为`0`, 则不进行限制.

一般这个数值设置为上游服务器的数量即可, 如果上游服务器较多, 可根据实际情况缩小这个数值.
