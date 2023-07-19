---
title: "如何调试OpenResty的Lua代码"
date: 2019-07-10T23:56:36+08:00
lastmod: 2019-07-10T23:56:36+08:00
categories:
  - OpenResty
tags:
  - openresty
  - lua
  - debug
draft: false
---

## 在适当位置打印输出

### 使用print

使用`print()`, 相当于`ngx.log(ngx.NOTICE, ...)`, 适用于所有阶段.

`nginx.conf`

```nginx
# 设置为notice级别, 默认为error级别
error_log  logs/error.log  notice;
```

```lua
-- 在lua代码中需要调试的位置添加：
print(var)
```

通过在`error.log`中查看相关输出

> 注意: <br>
> 1) print无法处理lua的table对象, 需要先序列化. <br>
> 2) print无法处理函数对象. <br>
> 3) 需要提前在nginx.conf中设置`error_log  logs/error.log  notice;`, 才能正常使用print()

### 使用nxg.say或ngx.print

使用`ngx.say()`或`ngx.print()`, 这俩区别是在内容输出时前一个比后一个多一个换行符.

```lua
-- 注意：
-- 这两个api只能用于rewrite、access、content三个阶段.
-- 直接在前端输出内容.
-- 该api用于rewrite阶段时, 将跳过其他2个阶段, 同理用于access阶段时, 将跳过content阶段, 但剩余其他阶段不会跳过.
-- 不能处理输出函数对象.
-- 如果输出的变量是lua的table对象, 需要对table进行序列化, 如下：
json = require "cjson"
json_encode = json.encode

foo = {1, bar="2"}
ngx.say(json_encode(foo))
```

## 设置断点

### 函数块内代码的断点

```lua
do return end
```

放置在函数代码块中需要断点的位置, 这样断点位置后的函数代码不会被运行, 注意不会终止代码文件运行.

### 模块级别(文件)的断点

```lua
return
```

用在条件代码块时, return不论是否放在最后一行, 都会最后一个执行, 并会终止当前代码文件的运行.

用于循环代码块时, 会在return所在位置终止当前代码文件的运行.

```lua
do return end
```

在代码文件级别、条件代码块、循环代码块需要设置断点的位置设置`do return end`, 将终止当前代码文件的运行.

> 注意: <br>
> 以上说的终止代码文件执行, 终止的是`return`所在的当前文件(模块)级别, 并不会影响其他执行阶段；<br>
> 至于是否会终止当前执行阶段, 就要看当前执行阶段的代码块中是否有`return`了, 如果`return`只是出现在当前代码文件的`require`的模块中, 或者是当前代码文件调用的外部定义的函数中, 这个`return`也不会影响到当前执行阶段.

### 当前请求的断点

```lua
ngx.exit(status)   -- status >= 200 (i.e., ngx.HTTP_OK and above)
```

当前请求将在`ngx.exit`的位置给用户响应, 后续代码不会执行,

> 注意: `header_filter/body_filter/log`阶段的代码还是会执行的.

### 当前阶段的断点

`ngx.exit(0)` 或者 `ngx.exit(ngx.OK)`

- 当前阶段的位于`ngx.exit(0)`或`ngx.exit(ngx.OK)`后的所有代码不会执行, 后续阶段不受影响
- 适用于阶段函数调用的外部函数中
- 注意使用`ngx.exit(-1)`或`ngx.exit(ngx.ERROR)`后, 只有`log`阶段会执行, 并且不会给客户端响应任何内容和响应码.

## lua_code_cache指令

`lua_code_cache`指令默认是`on`状态, 即会对代码和一些共享数据做缓存, 大大提高性能, 这个在生产环境一定是要打开的.

对于测试阶段, 对于一些测试场景也需要打开, 比如对模块级别变量的测试, 压测等场景.

其他测试场景, 在使用`*_by_lua_file`的方式时, 为了不必频繁`reload nginx`来重新加载代码的麻烦, 可以将该指令设置为`off`.

> syntax: lua_code_cache on | off <br>
> default: lua_code_cache on <br>
> context: http, server, location, location if
