---
title: "OpenResty数据共享的几种方式"
date: 2019-07-22T23:03:32+08:00
authors:
  - windvalley
categories:
  - OpenResty
tags:
  - openresty
  - lua
draft: false
---

## ngx.var 变量

数据生命周期:

- 同一个请求的所有执行阶段, 请求结束后自动销毁, 另外, 可以跨 location,
  也就是说在内部重定向(ngx.exec)或外部重定向(ngx.redirect)产生的子请求中该变量依然有效;
- 另外很重要的一点是 ngx.var 可以在 lua 代码和 Nnginx C 模块之间共享数据,
  其他数据共享方式都做不到.

`ngx.var`是获取 nginx 变量, 包括 nginx 配置文件中通过`set`或`set_by_lua*`自定义的变量、
或者 nginx 内置的变量(比如`remote_addr`等)、
或者表示 http 请求头字段的变量(获取方法: `ngx.var.http_`)和
表示请求 url 某个参数的变量(获取方法: `ngx.var.arg_`).

不能使用`local`直接声明具体的变量, 需要先在 Nginx 配置文件中定义`foo`变量, 比如:
`set $foo ""`

`local ngx.var.foo = 123`,
此写法错误, 因为`ngx.var`是请求(包括子请求)各个阶段全局有效的,
不能`local`化, 但是可以像下面这样:

```lua
local ngx_var = ngx.var  -- 先local ngx_var = ngx.var 来缓存代码, 有利于提高性能
ngx_var.foo = 123   -- 注意foo必须提前通过set来定义, 才能赋值
```

另外需要注意绝大部分的 Nginx 内置变量是不能被重新赋值的,
比如`$arg_PARAMETER`、`http_NAME`、`$remote_addr`、`$query_string`等等,
极少数的可以, 比如`$args`、`$limit_rate`.

该变量只能用来引用字符串类型数据, 不支持引用`Lua`的其他数据结构.

该变量类型因为涉及到字符串`hash`、`hash`表查找等过程, 所以性能较差,
如需在各个阶段使用, 为了提高性能可以把`ngx.var.`赋值给`ngx.ctx`,
比如`ngx.ctx.host = ngx.var.http_host`.

该变量生命周期的测试方法:

`nginx.conf`

```nginx
# 注意这个要是notice级别, 否则print打印的内容不会出现在error.log.
error_log path/error.log notice;
lua_code_cache on; # 需要打开缓存测试才有意义.

location /t1/ {
    rewrite_lua_by_block { print(ngx.var.http_hw)} -- 或者如下注释, 用来提高性能,
                                                   -- 其他每个阶段都可以效仿,
                                                   -- 通过local 缓存代码提高性能.
    -- rewrite_by_lua_block {
    --        local ngx_var = ngx.var
    --         print(ngx_var.http_hw)
    -- }
    access_lua_by_block { print(ngx.var.http_hw) }
    content_lua_by_block {
        print(ngx.var.http_hw)
        ngx.exec('/t2/')  -- 内部重定向, 注意内部重定向后,
                          -- 当前location的剩余执行阶段都不会被运行.
        --ngx.redirect('/t2/') -- 外部重定向, 外部重定向后,
                               -- 当前location的剩余执行阶段依然会被运行.
        }
    header_filter_lua_by_block { print(ngx.var.http_hw) }
    body_filter_lua_by_block { print(ngx.var.http_hw) }
    log_lua_by_block { print(ngx.var.http_hw) }
}

location /t2/ {
    rewrite_by_lua_block {print(ngx.var.http_hw)}
    access_by_lua_block {print(ngx.var.http_hw) }
    content_by_lua_block {
        print(ngx.var.http_hw)
        ngx.say(ngx.var.http_hw)
        }
    header_filter_by_lua_block { print(ngx.var.http_hw) }
    body_filter_by_lua_block { print(ngx.var.http_hw) }
    log_by_lua_block { print(ngx.var.http_hw) }
}
```

执行请求测试, 可以通过`error.log`中看到`location /t1/`的 rewrite、access、content 和
`location /t2/`的全部阶段都可以看到`ngx.var`的变量值,
说明了`ngx.var`变量的生命周期是可以跨 location 的全请求执行阶段:

`curl localhost -H 'hw: hello world' -L`

不传递`hw`头再请求一次, `error.log`中的`ngx.var`变量值全部为`nil`,
可知`ngx.var`的生命周期不能跨请求存在:

`curl localhost` # hw 头的值为 nil, 相当于给 ngx.var.http_hw 重新赋值为了 nil.

## ngx.ctx 变量

数据生命周期:

- 同一个请求的各个阶段, 请求结束后自动销毁; 注意无法跨`location`生存,
  即在重定向后的子请求中该变量将是一个空`table`;

- 值得一提的是, `ngx.ctx`虽然不能跨`location`,
  但有一个库`lua-resty-ctxdump`可以解决这个问题.

- 注意如果`ngx.ctx`用于`init_worker_by_lua*`阶段,
  那么`ngx.ctx`的生命周期到当前阶段 lua 调用的结束.

`ngx.ctx`变量其实是一个 lua 的`table`,
速度相对`ngx.var`快, 可以存储各种 lua 对象.

使用时最好通过`local ngx_ctx = ngx.ctx`的方式来缓存`ngx.ctx` API, 提高性能,
不过要特别注意, 不能在模块级别进行缓存, 要在函数级别进行缓存,
因为第一个请求的`ngx.ctx`已经在请求结束后销毁了呀, 新的请求过来时,
由于`local ngx_ctx = ngx.ctx` 没有在函数中声明,
所以函数中的`ngx_ctx`引用的还是上一次请求的`ngx.ctx` 内存地址,
和新请求的`ngx.ctx`没有半毛钱关系, 所以新请求的`ngx.ctx.foo`由于没有被定义过,
故值为`nil`.

测试方法:

测试前提, `nginx.conf`中的这个指令值不能是`off`: `lua_code_cache = on`

`ctx_test.lua`

```lua
local _M = {}

local ngx_ctx = ngx.ctx -- 对ngx.ctx API进行缓存, 放到模块级别

function _M.bar()
    ngx_ctx.foo = 'test'
end

return _M
```

`nginx.conf location {}`配置:

```nginx
content_by_lua_block {
    local ctx_test = require "ctx_test"
        ctx_test.bar()
        ngx.say(ngx.ctx.foo)
    }
```

测试, 第一次请求:
`curl localhost/t/`

输出:
`test`

测试, 第二次请求:
`curl localhost/t/`

输出:
`nil`

表明`loca ngx_ctx = ngx.ctx` 放到模块级别的时候, 只对第一次请求可以获取变量 foo 值.

下面对`ctx_test.lua`进行修改如下:

```lua
local _M = {}

function _M.bar()
    local ngx_ctx = ngx.ctx -- 对ngx.ctx API进行缓存, 放到函数级别
    ngx_ctx.foo = 'test'
end

return _M
```

然后进行第二次测试:

`curl localhost/t/1` # 第一次请求

输出: `test`

`curl localhost/t/` # 第二次请求

输出:
`test`

表明将`ngx.ctx` API 的缓存语句放到函数级别后, 所有请求都可以获取变量 foo 值

生命周期测试方法:

验证过程类似验证`ngx.var`的生命周期部分.

```nginx
location /t1/ {
    rewrite_by_lua_block {
        local ngx_ctx = ngx.ctx # local的目的是用来缓存ngx.ctx这个table, 优化性能
        ngx_ctx.hw = ngx.var.http_hw  # 注意具体到变量部分, 即hw部分, 就不能加local了
        print(ngx_ctx.hw)
    }
    access_by_lua_block {
        local ngx_ctx = ngx.ctx # 每个阶段的代码都先local一下来提供性能
        print(ngx_ctx.hw)
    }
  ... # 其他阶段类似
}
```

## 自定义的全局变量

也就是没有使用 local 声明的变量.

数据生命周期:

- 同一个 worker 下的所有请求的全部执行阶段.

- `reload`后变量所引用的内容才会被销毁.

应尽量避免使用该类型变量来共享数据, 很容易造成冲突.

生命周期的测试方法:

`nginx.conf`

```nginx
error_log path/error.log notice;  # 注意这个要是notice级别,
                                  # 否则print打印的内容不会出现在error.log.
lua_code_cache on; # 需要打开缓存测试才有意义

rewrite_lua_by_block {
    foo = ngx.var.http_hw
    if foo then
        hw = foo
    end
    print(hw)
    }
access_lua_by_block { print(hw) }
content_lua_by_block { print(hw) }
header_filter_lua_by_block { print(hw) }
body_filter_lua_by_block { print(hw) }
log_lua_by_block { print(hw) }
```

执行请求测试, 可以通过`error.log`中看到每个阶段都打印了`hello world`,
说明了`hw`这个全局变量在同一个请求的每个阶段都有效:

`curl localhost -H 'hw: hello world'`

不传递`hw`头再请求一次, 在`error.log`中发现每个阶段也同样都打印了`hello world`,
说明了全局变量`hw`对同一个 worker 下的所有请求都有效:

`curl localhost`

## 模块级别变量

数据生命周期:

在同一个 worker 内的所有请求都可以共享该变量, 直到`reload`后原变量的引用才会销毁,
新变量的引用会在`require`的时候重新加载.

变量一定要定义在模块中(就是要通过`require`来引用),
一定要使用`local`声明来避免`race condition`, 最好也同时定义变量的读取函数,
如果涉及到变更, 也最好有相关变更函数.

尽量对该变量进行只读, 当然有写入需求也是可以的, 需要注意避免`race condition`的情况.

变量引用的可以是`lua`语法中的任何数据结构, 比如变量表示的可以是字符串,
也可以是`table`, 大部分情况使用`table`.

我们的 lua 程序要通过`require`加载变量所在模块后, 才能使用该共享变量.

模块只会被加载一次, 也就是只有第一个请求会`require`来加载模块,
以后的其他请求都会直接共享第一个请求加载的数据(包括代码).

注意`reload`之后, 之前加载的代码和数据都会清空, 由新的第一个请求重新`require`加载.

另外还需要注意的是, 如果 Nginx 配置文件中配置了`lua_code_cache off`来关闭代码缓存,
那么每个请求都会进行`require`重新加载模块,
所以如果我们对这个变量进行变更操作后会发现其他请求看不到内容有变更,
我们在测试的时候一定要注意这个坑.

生命周期测试:

`nginx.conf`

```nginx
lua_code_cache on;  # 不能是off
worker_processes  1; # 必须启用一个worker
```

在`lua_package_path`定义的路径下写一个模块, 如下:

```lua
-- module_share_var.lua 模块文件的名字
local _M = {}  -- 一定要加local, 避免race condition

-- 定义模块级变量, 此处一定要为local, 避免race condition
local data_share = {
    cat = 1,
    dog = 2,
    pig = 3
}

-- 此处不能加local, 因为_M已经在前面加local声明了
_M.insert = function(name, value)  -- lua函数的另一种写法
    data_share[name] = value
end

-- 此处不能加local 因为_M已经在前面加local生命了
function _M.get_data() -- lua函数的常规写法
   return data_share
end

return _M
```

在 lua 项目代码的`rewrite`或`access`阶段写如下代码:

```lua
local data_share = require "module_share_var" -- 加载模块变量的所在模块
local ngx_re = require "ngx.re"
local ngx_var = ngx.var

local header_value = ngx_var.http_cat

if header_value then
    local res, err = ngx_re.split(header_value, ":")
    data_share.insert(res[1], res[2]) -- 向模块变量插入一条数据
end
```

在 lua 项目代码的`content`阶段写如下代码:

```lua
local data_share = require "module_share_var"
local cjson = require "cjson"

ngx.say(cjson.encode(data_share.get_data()))
```

开始测试:
`curl localhost/t1/`

输出: `{"dog":2,"pig":3,"cat":1}`

继续测试:
`curl localhost/t1/ -H 'cat: cat111: 111'`

输出: `{"dog":2,"pig":3,"cat":1,"cat111":" 111"}`

继续测试:
`curl localhost/t1/ -H 'cat: cat222: 222'`

输出:
`{"pig":3,"cat222":" 222","cat111":" 111","dog":2,"cat":1}`

reload:
`openresty -s reload`

测试: `curl localhost/t1/`

输出: `{"dog":2,"pig":3,"cat":1}`

由上测试可知, 模块级别变量的生命周期在同一个 worker 下的所有请求的所有阶段,
`reload`才会导致原变量引用销毁.

## 通过 lrucache 库共享数据

属于模块级别变量共享数据的方法, 不考虑数据因为 lru 算法被清理的情况,
数据生命周期和模块级别变量的生命周期一致, 同一个 worker 下的所有请求的各个阶段有效.

必须用于模块中.

实现方法举例:

```lua
local lrucache = require "resty.lrucache"
local lru = lrucache.new(1000)

local _M = {}

local local_func1(arg)   -- 本地函数, 供_M中的函数调用
    local key1 = arg["x1"]
    local key2 = arg["x2"]
    local key = key1 .. key2

    local obj = lru:get(key)
    if obj then
        return obj:find(ngx_var.xxx) or 0
    end

    obj = ...   -- 此处省略obj的实现方法

    lru:set(key, obj)
    return obj:find(ngx_var.xxx) or 0
end

_M.func1 = function()
    local foo = ...   -- 省略foo的获取方法
    local bar = local_func1(foo)

    ...   -- 省略其他功能代码
end

return _M
```

其他项目文件通过 require 该模块来使用.

## 通过把变量作为参数传递给回调阶段函数来共享数据

数据生命周期:

- 在我们实现的某一个功能模块的各个执行阶段共享数据;

- 在`init_by_lua*`阶段给该变量赋的值, 是所有 worker 的所有请求在剩余各个阶段共享的;

- 在`init_worker_by_lua*`阶段给变量赋的值, 是该 worker 的所有请求在剩余各个阶段共享的;
  其他阶段给变量赋的值, 只对同一个请求的剩下阶段有效, 请求结束后, 自动销毁,
  且不能跨 location, 即对于子请求无效.

实现方法:

把我们要写的项目功能进行模块化,
在模块文件里写该项目功能需要的相关执行阶段函数(回调函数, 给项目方调用),
然后写一个`require`这些功能模块的入口模块文件(项目主文件),
这样这些功能模块代码就都在一个文件里了,
我们可以在这个入口模块文件中通过定义一个`local`变量,
然后把这个变量通过给回调函数传参的方式进行传递,
这样就间接实现了数据在同一个功能模块的不同执行阶段的共享.

实现代码和测试方法举例:

`nginx.conf`

```nginx
# http {}
    lua_package_path '/usr/local/etc/openresty/lua/?.lua;;';
    init_by_lua_block {
        project1 = require "project1" -- 项目project1以模块的形式进行加载

        -- 这里的init_master会被当作index去project1这个table里去检索,
        -- 最终返回的是一个函数(闭包, 把init_master这个参数封装进去了),
        -- 然后通过()对该函数进行执行,
        -- 至此, 项目project1的所有功能模块的init_master函数就都执行完了.
        project1.init_master()
       }

    init_worker_by_lua_block {
        project1.init_worker()
    }
# server { location {} }
       location /t2/ {
            rewrite_by_lua_block { project1.rewrite() }
            access_by_lua_block { project1.access() }
            content_by_lua_block { project1.content() }
            header_filter_by_lua_block { project1.header_filter() }
            body_filter_by_lua_block { project1.body_filter() }
            log_by_lua_block { project1.log() }
       }
```

功能模块 1 文件:

```lua
-- /usr/local/etc/openresty/lua/modules/module1.lua
local cjson = require "cjson"

local _M = {}

_M.name = "module1" -- 这里必须给模块命名, 后面会用这个名字还区分不同模块的变量

_M.init_master = function(ctx)
    ctx.foo1 = "I am init_master"  -- 由于在项目模块文件中已经local定义了ctx,
                                   -- 这里不能再使用local定义, 否则无法传递变量.
    print(cjson.encode(ctx))
end

_M.init_worker = function(ctx)
    ctx.foo2 = "I am init_workser"
    print(cjson.encode(ctx))
end

_M.rewrite = function(ctx)
    ctx.foo3 = "I am rewrite phase"
    print(cjson.encode(ctx))
end

_M.access = function(ctx)
    ctx.foo3 = ctx.foo3 .. "--> I am access phase"
    print(cjson.encode(ctx))
end

_M.content = function(ctx)
    ctx.foo3 = ctx.foo3 .. "--> I am content phase"
    print(cjson.encode(ctx))
end

_M.header_filter = function(ctx)
    ctx.foo3 = ctx.foo3 .. "--> I am header_filter phase"
    print(cjson.encode(ctx))
end

_M.body_filter = function(ctx)
    ctx.foo3 = ctx.foo3 .. "--> I am body_filter phase"
    print(cjson.encode(ctx))
end

_M.log = function(ctx)
    ctx.foo3 = ctx.foo3 .. "--> I am log phase"
    print(cjson.encode(ctx))
end

return _M
```

`project1`项目入口文件:

```lua
-- /usr/local/etc/openresty/lua/project1.lua

local module1 = require "modules.module1"  -- 功能模块1
--local module2 = require "modules.module2" -- 功能模块2
--local module3 = require "modules.module3" -- 功能模块3

local _M = {}

_M._VERSION = "0.1"    -- 用来表示项目project1的版本号, 便于管理维护.

-- 如果模块之间在执行时有顺序依赖, 需要按先后顺序排序.
local modules = {
    module1,
    --module2,
    --module3,
}

local ctxs = {}  -- 定义用于存储不同模块的共享变量ctx的table.

setmetatable(_M, { __index = function(self, handler_name)
    func = function()
        for i=1, #modules do
            local module = modules[i]
            local module_name = module.name -- 每个功能模块必须有自己的名字,
                                            -- 用于区分共享变量.

            local ctx = ctxs[module_name] or {}  -- 这里需要使用local定义ctx
            local f = module[handler_name]
            local err = f and f(ctx)  -- 如果函数存在就执行函数,
                                      -- 函数返回值应该是nil, 否则报错.
            ctxs[module_name] = ctx

            if err then
                ngx.log(ngx.ERR, module_name, err)
            end
        end
    end

    -- 这样赋值操作后, 之后的对handler_name的检索就不需要走__index方法了,
    -- 代码加速的作用.
    _M[handler_name] = func

    -- 给检索handler_name的table(就是nginx配置文件中的prject1),
    -- 返回一个函数指针(地址)func,
    -- 这个func指向的函数包含并执行了项目所有功能模块的handler_name函数,
    -- 最终通过nginx配置文件的project1.handler_name()的小括号来触发了对这些功能模块的handler_name函数的执行.
    return func
end
})

return _M
```

开始测试: `openresty -s reload`

可以在`error.log`中看到如下输出:

```vim
2019/07/21 22:26:13 [notice] 644#9204: signal 1 (SIGHUP) received from 908, reconfiguring
2019/07/21 22:26:13 [notice] 644#9204: reconfiguring
2019/07/21 22:26:13 [notice] 644#9204: [lua] module1.lua:9: f(): {"foo1":"I am init_master"}
2019/07/21 22:26:13 [notice] 644#9204: using the "kqueue" event method
2019/07/21 22:26:13 [notice] 644#9204: start worker processes
2019/07/21 22:26:13 [notice] 644#9204: start worker process 909
2019/07/21 22:26:13 [notice] 909#34683: *6 [lua] module1.lua:14: f(): {"foo2":"I am init_workser","foo1":"I am init_master"}, context: init_worker_by_lua*
```

由以上输出可知, 变量在`init_lua`和`init_worker`两个阶段进行了传递.

请求测试:

`curl localhost/t2/`

```log
2019/07/21 22:26:25 [notice] 909#34683: *7 [lua] module1.lua:19: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:26:25 [notice] 909#34683: *7 [lua] module1.lua:24: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:26:25 [notice] 909#34683: *7 [lua] module1.lua:29: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase--> I am content phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:26:25 [notice] 909#34683: *7 [lua] module1.lua:34: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase--> I am content phase--> I am header_filter phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:26:25 [notice] 909#34683: *7 [lua] module1.lua:39: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase--> I am content phase--> I am header_filter phase--> I am body_filter phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:26:25 [notice] 909#34683: *7 [lua] module1.lua:44: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase--> I am content phase--> I am header_filter phase--> I am body_filter phase--> I am log phase","foo1":"I am init_master"} while logging request, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
```

再执行一遍请求: `curl localhost/t2/`

```log
2019/07/21 22:29:47 [notice] 909#34683: *8 [lua] module1.lua:19: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:29:47 [notice] 909#34683: *8 [lua] module1.lua:24: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:29:47 [notice] 909#34683: *8 [lua] module1.lua:29: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase--> I am content phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:29:47 [notice] 909#34683: *8 [lua] module1.lua:34: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase--> I am content phase--> I am header_filter phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:29:47 [notice] 909#34683: *8 [lua] module1.lua:39: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase--> I am content phase--> I am header_filter phase--> I am body_filter phase","foo1":"I am init_master"}, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
2019/07/21 22:29:47 [notice] 909#34683: *8 [lua] module1.lua:44: f(): {"foo2":"I am init_workser","foo3":"I am rewrite phase--> I am access phase--> I am content phase--> I am header_filter phase--> I am body_filter phase--> I am log phase","foo1":"I am init_master"} while logging request, client: 127.0.0.1, server: localhost, request: "GET /t2/ HTTP/1.1", host: "localhost"
```

由以上两次请求的结果可知,
`init_by_lua`和`init_worker_by_lua`阶段对变量赋的值对所有请求都有效;
其他阶段对变量的赋值, 仅对当次请求有效.

## ngx.shared.DICT

数据生命周期: 可以在所有 worker 之间共享数据, 对所有请求的各个执行阶段都有效.

只支持缓存字符串类型数据, 不支持 lua 其他数据类型.
当我们需要存储 table 等复杂数据类型时, 需要先序列化再存储, 获取数据时再反序列化使用.

对外提供了`20`多个`Lua API`, 不过所有的这些 API 都是原子操作,
你不用担心多个 worker 和高并发的情况下的竞争问题.

必须提前在 Nginx 配置文件的`http上下文`中, 声明共享内存的大小, 并且不能在运行期变更.

基本使用方法:

`nginx.conf`:

```nginx
# nginx.conf  http {}
lua_shared_dict dogs 10m;
```

`lua code`:

```lua
local dict = ngx.shared.dogs
dict:set("Tom", 56)
ngx.say(dict:get("Tom"))
```

使用管理类 API:

```lua
require "resty.core.shdict"
local cats = ngx.shared.cats
local capacity_bytes = cats:capacity()
local free_page_bytes = cats:free_space()
```

## 使用数据库存储数据

比如使用 redis、memcached、MySQL、PostgreSQL 等.

数据生命周期: 除了故障问题、主动清理数据等情况, 数据一直都在.

## 数据共享方式总结

| 数据共享方法          | 数据的生命周期                                                                                                                                                                                                           | 可使用的上下文(context)                                                                                                                                                      | 支持的数据类型 | 特点                                                                                                                                                        |
| :-------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ngx.ctx` API         | 单个请求级别但不能跨 location                                                                                                                                                                                            | `init_worker`, set, rewrite, access, content, `header_filter`, `body_filter`, log, balancer, `ngx.timer.*`                                                                   | 任何 lua 对象  | 数据查询速度比`ngx.var`快                                                                                                                                   |
| `ngx.var` API         | 单个请求级别且可跨 location                                                                                                                                                                                              | set, rewrite, access, content, `header_filter`, `body_filter`, log                                                                                                           | 字符串类型     | 如果不考虑使用外部数据库方案的话, 这个 API 是 lua 代码和`Nginx C`模块之间共享数据的唯一方法. 对于 lua 复杂对象, 通过序列化和反序列化和 lua 程序进行数据交互 |
| lua 自定义全局变量    | 单个 worker 下的所有请求                                                                                                                                                                                                 | 所有执行阶段                                                                                                                                                                 | 任何 lua 对象  | 容易造成 racecondition; 不推荐使用或者说一定不要使用                                                                                                        |
| 模块级别变量          | 单个 worker 下的所有请求                                                                                                                                                                                                 | 所有执行阶段                                                                                                                                                                 | 任何 lua 对象  | 变量只能定义在模块中, 通过 require 模块使用变量                                                                                                             |
| `resty.lrucache`      | 单个 worker 下的所有请求                                                                                                                                                                                                 | 所有执行阶段                                                                                                                                                                 | 任何 lua 对象  | 只能用于模块中, 通过 require 模块来使用                                                                                                                     |
| 变量传参给回调函数    | 在一个功能模块的各个执行阶段共享数据: 1.在`init_by_lua*`阶段给该变量赋的值, 是 server 级别的; 2.在`init_worker_by_lua*`阶段给变量赋的值,是单个 worker 级别的; 3.其他阶段给变量赋的值, 单个请求级别的, 且不能跨 location. | 所有执行阶段                                                                                                                                                                 | 任何 lua 对象  | 相同功能模块的不同执行阶段共享数据                                                                                                                          |
| `ngx.shared.DICT` API | server 级别, 即所有 worker 之间共享数据                                                                                                                                                                                  | init, `init_worker`, set,rewrite, access, content, `header_filter`, `body_filter`, log, `ngx.timer.*`, balancer, `ssl_certificate`, `ssl_session_fetch`, `ssl_session_store` | 字符串类型     | 对于 lua 复杂对象, 通过序列化和反序列化和 lua 程序进行数据交互                                                                                              |
| 数据库服务            | server 级别                                                                                                                                                                                                              | 所有执行阶段                                                                                                                                                                 | 字符串类型     | 对于 lua 复杂对象,通过序列化和反序列化和 lua 程序进行数据交互                                                                                               |
