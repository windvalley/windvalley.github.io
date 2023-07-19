---
title: "Nginx tips"
date: 2017-01-07T13:48:38+08:00
lastmod: 2017-01-07T13:48:38+08:00
categories:
    - OpenResty
tags:
    - nginx
    - openresty
    - tips
keywords:
    -
draft: false
---

### `internal`指令.

只能用于`location`上下文;
表示只能是内部请求可以访问该`location`;
每个内部请求最多只能跳转`10`次, 超过限制次数, 会在`error_log`中打印500错误.

### 关闭日志

nginx 关闭某个server 块的`access_log`和`error_log`, 特别注意关闭`error_log`的用法.

```nginx
server {
    listen 443 ssl;
    server_name _;
    access_log  off;
    error_log /dev/null;
    }
```

### `last`和`break`的区别

break和last都能阻止继续执行后面的rewrite指令, last对于重写后的URI会重新匹配location, 而break不会重新匹配location.

last: 停止当前这个请求, 并根据rewrite匹配的规则重新发起一个内部请求. 新请求又从第一阶段开始执行.

break: 相对last, 重写完URI后, break并不会重新发起一个请求, 而只是跳过当前的rewrite阶段, 并执行本请求location后续的执行阶段.

### 请求头字段不能使用下划线

nginx默认request的header的中包含`_`时, 会自动忽略掉.

解决方法是: 在nginx里的`nginx.conf`配置文件中的http部分中添加如下配置:

```nginx
underscores_in_headers on;  # 默认是off的
```

另外不建议开启, 因为可能会存在安全风险.

### 理解`ngx_http_stub_status_module`模块引入的几个参数

```
Active connections: 291
server accepts handled requests
     16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106

# requests 是客户端的总请求数
# accepts 是nginx接受的客户端的请求数
# handled 是nginx处理的客户端的请求数
# 一般情况下accepts和handled的数量是相等的, 除非达到了一些系统资源的限制, 比如worker_connections的限制.

# Reading 表示当前有多少个tcp连接正在用于nginx读取客户端请求的header.
# Writint 表示当前有多少个tcp连接正在用于nginx给客户端写响应.
# Waiting 表示当前有多少个tcp连接正在用于nginx等待客户端的请求.
# Active connections 表示nginx和客户端之间的tcp连接的总数量, 是Reading、Writing、Waiting的数量之和.
```

### `ngx-http-proxy-module`模块引入的缓存功能的缓存文件说明

```
stat 18de49b189a66f57208c47852e2ae15d

Access: 2019-08-07 17:44:44.875091237 +0800   # 对于新生成的缓存文件, 只在第一次请求过来的时候更新此时间.
Modify: 2019-08-07 17:44:27.330409891 +0800   # 文件过期后, 第一个请求过来时, 会想上游验证文件是否还有效, 上游返回304后, 缓存文件内容不会变, 但会更新此时间.
Change: 2019-08-07 17:44:27.330409891 +0800   # 和上一条一致.
```

### 理解`$proxy_add_x_forwarded_for`

```
the "X-Forwarded-For" client request header field with the $remote_addr
variable appended to it, separated by a comma.
If the "X-Forwarded-For" field is not present in the client request header,
the $proxy_add_x_forwarded_for variable is equal to the $remote_addr variable.
```

这么理解:

用户请求前端nginx, 前端nginx设置的`proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for`指令, 把用户IP(即`$remote_addr`)赋值给`x-forwarded-for`头传递给kong,
kong nginx设置的`proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for`指令,把前端的IP(即`$remote_addr`)追加给`x-forwarded-for`头, 以逗号分隔, 传递给后端服务.
这样后端看到的`x-forwarded-for`头的值就是两个IP了, 用户ip和前端ip.

### 去掉用户请求uri结尾的`/`

```nginx
# 匹配uri(不包括url参数)结尾带/的请求
if ($uri ~ /$) {
    # 301重定向, 让用户重新请求uri结尾去掉/的url
    rewrite ^(.*)/(.*)$ $1$2 permanent;
}
```

### map指令用法

map指令是由`ngx_http_map_module`模块提供的, 默认情况下安装nginx都会安装该模块.

map 的主要作用是创建自定义变量, 通过使用nginx的内置变量, 去匹配某些特定规则,
如果匹配成功则设置某个值给自定义变量. 而这个自定义变量又可以作于他用.

场景:

匹配请求url的参数, 如果参数是debug则设置`$foo = 1`, 默认设置`$foo = 0`
```nginx
map $args $foo {
    default 0;
    debug   1;
}
```

解释:

`$args`是nginx内置变量, 就是获取的请求 url 的参数. 如果`$args`匹配到debug,
那么`$foo`的值会被设为`1`, 如果`$args`一个都匹配不到`$foo`就是`default`定义的值,
这里就是`0`.

另一个例子:
```nginx
    # 指定域名或者泛域名分片缓存
    map $channel $slice_store {
        hostnames;
        default         off;
    }
```

通过以上代码来说明`map`的一些用法:

`map`只能用于`http`上下文;

只有变量`slice_store`在使用的时候才会被赋值, 所以即使有很多条map语句,
对于没有使用该变量的清气来说也不会产生任何额外开销;

由上一条说明可知`$channel`变量可以在之后再定义;

`hostnames`是map的一个参数, 并不是作为`channel`变量的值存在的;

参数`hostnames`必须在map块的第一行,
`hostnames`表明`channel`变量的值可以是有前缀或者是有后缀的域名,
比如`channel`变量的值如果是: `*.example.com on`或者是`example.* off`等,
那么`slice_store`变量的值就是`on`或`off`,
如果`channel`变量值是`*.example.com ""` 或者`channel`变量没有值,
那么`slice_store`变量值将是`default`的值`off`.

### location指令详解

```
Syntax: location [ = | ~ | ~* | ^~ ] uri { ... }
        location @name { ... }
Default: —
Context: server, location
```

#### `[ = | ~ | ~* | ^~ ]`释疑

`[ = | ~ | ~* | ^~ ]`表示`location`后有如下5种修饰符的情况:

- `=`: 后面的`uri`是字符串, 对请求的`uri`做精确匹配(不匹配请求`uri`的参数), 该`location`被匹配到后终止继续匹配.
- `^~`: `uri`是字符串, 修饰符由一个非符号`^`和一个正则符号`~`组成, 表示本`location`匹配完成后, 不继续匹配正则表达式的`location`.
- `~`: `uri`是正则表达式, 对请求的`uri`做正则匹配, 大小写敏感.
- `~*`: `uri`是正则表达式, 对请求的`uri`做正则匹配, 大小写不敏感.
- `没有修饰符`: `location`后`uri`前没有任何修饰符, `uri`是字符串, 对请求的`uri`做字符串前缀匹配, 匹配到一个字符串最长的`location`, 然后继续匹配正则表达式的`location`.

#### 匹配顺序说明

如果在同一个上下文配置块内有多个`location block`, 针对上面介绍的5种类型`location`, `Nginx`的匹配顺序为:

首先匹配`uri`是字符串的`location`, 包括`没有修饰符`的、修饰符是`=`和`^~`的, 这3三种`location`按照在配置文件出现的先后顺序依次匹配;

如果遇到`=`修饰符的并且匹配上了, 就立刻终止继续匹配;

否则将这3种`location`全部匹配一遍, 匹配完成后, 如果修饰符是`^~`的某个`uri`字符串最长的`location`匹配上了, 则终止继续匹配;

否则记录下匹配到的没有修饰符的`uri`字符串最长的`location`, 并继续匹配修饰符是`~`和`~*`的`location`, 这2种`location`按照在配置文件中出现的先后顺序依次匹配, 匹配上就马上终止继续匹配;

如果都没有匹配上, 则使用之前记录下来的没有修饰符的`uri`字符串前缀最长的那个`location`.
