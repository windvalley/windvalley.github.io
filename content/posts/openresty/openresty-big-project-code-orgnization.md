---
title: "OpenResty中大型项目的代码组织方式介绍"
date: 2019-07-01T14:34:54+08:00
lastmod: 2019-07-01T14:34:54+08:00
authors:
    - windvalley
categories:
    - OpenResty
tags:
    - openresty
    - lua
draft: false
---

适用于中大型项目, 包含多个功能模块,
每个功能模块需要在各自的不同执行阶段通过回调函数传参的方式来高效的共享数据.

## 具体的某个功能模块文件举例

`lua/modules/module1.lua`

```lua
local cjson = require "cjson"

local _M = {}

_M.name = "module1" -- 这里必须给模块命名, 后面会用这个名字还区分不同模块的变量.
_M._VERSION = "0.1"

_M.init_master = function(ctx)
    ctx.foo1 = "I am init_master"  -- 由于在项目模块文件中已经local定义了ctx, 这里不能再使用local定义, 否则无法传递变量.
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
    ctx.foo3 = ctx.foo3 .."--> I am access phase"
    print(cjson.encode(ctx))
end

_M.content = function(ctx)
    ctx.foo3 = ctx.foo3 .."--> I am content phase"
    print(cjson.encode(ctx))
end

_M.header_filter = function(ctx)
    ctx.foo3 = ctx.foo3 .."--> I am header_filter phase"
    print(cjson.encode(ctx))
end

_M.body_filter = function(ctx)
    ctx.foo3 = ctx.foo3 .."--> I am body_filter phase"
    print(cjson.encode(ctx))
end

_M.log = function(ctx)
    ctx.foo3 = ctx.foo3 .."--> I am log phase"
    print(cjson.encode(ctx))
end

return _M
```

## project1项目入口文件

`lua/project1.lua`

```lua
local module1 = require "modules.module1"  -- 功能模块1.
--local module2 = require "modules.module2" -- 功能模块2.
--local module3 = require "modules.module3" -- 功能模块3 , 多个功能模块依次require加载.

local _M = {}

_M._VERSION = "0.1"  -- 用来表示项目project1的版本号, 便于管理维护.

-- 如果模块之间在执行时有顺序依赖, 需要按先后顺序排序, 比如从redis读取配置的一定是放在第一个位置.
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
            local module_name = module.name -- 每个功能模块必须有自己的名字, 用于区分共享变量.

            local ctx = ctxs[module_name] or {}  -- 这里需要使用local定义ctx.
            local f = module[handler_name]
            local err = f and f(ctx) -- 如果函数存在就执行函数, 函数返回值应该是nil, 否则报错.
            ctxs[module_name] = ctx

            if err then
                ngx.log(ngx.ERR, module_name, err)
            end
        end
    end

    -- 这样赋值操作后, 之后的对handler_name的检索就不需要走__index方法了, 起到代码加速的作用.
    _M[handler_name] = func

    -- 给检索handler_name的table(就是nginx配置文件中的prject1),
    -- 返回一个函数指针(地址)func, 这个func指向的函数包含并执行了项目所有功能模块的handler_name函数,
    -- 最终通过nginx配置文件的project1.handler_name()的小括号来触发了对这些功能模块的handler_name函数的执行.
    return func
end
})

return _M
```

## `nginx.conf`的相关配置

```nginx
# block: http {}
    lua_package_path '/usr/local/etc/openresty/lua/?.lua;;';

    init_by_lua_block {
        project1 = require "project1" -- 项目project1以模块的形式进行加载.
        -- project2 = require "project2" -- 如果有多个项目, 在这里依次require加载.

        -- 这里的init_master会被当作index去project1这个table里去检索,
        -- 最终返回的是一个函数(闭包, 把init_master这个参数封装进去了),
        -- 然后通过()对该函数进行执行,
        -- 至此, 项目project1的所有功能模块的init_master函数就都执行完了.
        project1.init_master()
       }

    init_worker_by_lua_block {
        project1.init_worker()
    }

# block: server { location {} }
   # 项目project1的url路由.
   location /project1/ {
        rewrite_by_lua_block { project1.rewrite() }
        access_by_lua_block { project1.access() }
        content_by_lua_block { project1.content() }
        header_filter_by_lua_block { project1.header_filter() }
        body_filter_by_lua_block { project1.body_filter() }
        log_by_lua_block { project1.log() }
   }

   # 项目project2的url路由(如果有)
   #location /project2/ {
   #}
```
