---
title: "如何快速学习OpenResty的API"
date: 2019-05-28T20:23:30+08:00
lastmod: 2019-05-28T20:23:30+08:00
categories:
    - OpenResty
tags:
    - openresty
    - lua
    - api
draft: true
---
学会使用官方文档查询信息非常重要, 下面我们以OpenResty的`ngx.arg`这个API来演示如何快速学习并使用OpenResty的api.

### 方法1：查看`lua-nginx-module`官方文档

> https://github.com/openresty/lua-nginx-module#nginx-api-for-lua

`ngx.arg`部分的截图:

![ngx.arg](../../imgs/ngx_arg_api.jpg)


### 方法2：通过OpenResty提供的`CLI(command line interface)`工具`"restydoc"`

```bash
restydoc -s ngx.arg
```

输出如下(输出的内容和方法1的官方文档一模一样):

```text
ngx.arg
       syntax: val = ngx.arg[index]

       context: set_by_lua*, body_filter_by_lua*

       When this is used in the context of the set_by_lua* directives, this
       table is read-only and holds the input arguments to the config
       directives:

            value = ngx.arg[n]

       Here is an example

            location /foo {
                set $a 32;
                set $b 56;

                set_by_lua $sum
                    'return tonumber(ngx.arg[1]) + tonumber(ngx.arg[2])'
                    $a $b;

                echo $sum;
            }

       that writes out 88, the sum of 32 and 56.

       When this table is used in the context of body_filter_by_lua*, the
       first element holds the input data chunk to the output filter code and
       the second element holds the boolean flag for the "eof" flag indicating
       the end of the whole output data stream.

       The data chunk and "eof" flag passed to the downstream Nginx output
       filters can also be overridden by assigning values directly to the
       corresponding table elements. When setting "nil" or an empty Lua string
       value to "ngx.arg[1]", no data chunk will be passed to the downstream
       Nginx output filters at all.
```

### 方法3: 通过`lua-nginx-module`提供的测试案例学习api

> https://github.com/openresty/lua-nginx-module/tree/master/t

我们可以把`lua-nginx-module`通过`git clone`到本地服务器, 然后通过`grep -rl ngx.arg t/`来找哪些测试案例文件里有关于这个`api`的内容.

### 总结

我们可以把以上方法中涉及到的测试案例代码粘贴到自己的测试环境中进行测试, 体会其中的意思.

通过文档和测试, 我们可知:

`ngx.arg`可以用于两个阶段: `set_by_lua*`, `body_filter_by_lua*`

用于`body_filter`阶段时: `local chunk, eof = ngx.arg[1], ngx.arg[2]`

`ngx.arg[1]`表示body里的一行数据;

`ngx.arg[2]`是一个布尔值, `false`或`true`, 如果有数据, 则为`false`, 如果无数据(或`body`尾部), 则为`ture`.
