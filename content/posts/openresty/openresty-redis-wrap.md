---
title: "为OpenResty封装一个更易用更健壮的redis"
date: 2019-06-29T16:48:06+08:00
draft: false
authors:
    - windvalley
categories:
    - OpenResty
tags:
    - redis
    - lua
    - openresty
keywords:
---

## 功能

- [x] 封装一个代码清晰简单易用的redis

## 完整代码

```lua
-- @author: wxg


local redis = require "resty.redis"
local table_remove = table.remove
local setmetatable = setmetatable


local ok, table_new = pcall(require, "table.new")
if not ok or type(table_new) ~= "function" then
    table_new = function(_, _) return {} end
end


local _M = table_new(0, 54)

_M.name = "redis_wrap"
_M._VERSION = "0.1"

local mt = { __index = _M }


local host_default = "127.0.0.1"
local port_default = 6379
local db_index_default = 0
local auth_default = nil
local timeout_default = 1000 -- 1 second
local keepalive_default = "on"
local keepalive_max_idle_timeout_default = 10000 -- 10 seconds
local keepalive_pool_size_default = 100 -- count


-- 不对外提供接口的函数不使用_M.func的命名(为了更规范、更清晰),
-- 而是命名一个纯本地函数，注意self参数是经过传参传进来的。
local redis_connect = function(self)
    local red = redis:new()
    red:set_timeout(self.timeout)

    local ok, err = red:connect(self.host, self.port)
    if not ok then
        return nil, err
    end

    -- 因为self.auth我们赋值了nil，而_M的元表的__index属性对应的是函数，
    -- 这时当我们检索self.auth时，由于值为nil，所以被认为是不存在该字段，
    -- 故去元表中查询auth字段，从而返回了一个函数，
    -- 所以此处如果通过self.auth来取auth的值，取出来的是一个函数，
    -- 在接下来的使用中会报错。综上，需要使用rawget这种方式来取字段值为nil的字段。
    local auth = rawget(self, "auth")
    if auth and auth ~= "" then
        local count, err = red:get_reused_times()
        if count == 0 then
            local ok, err = red:auth(auth)
            if not ok then
                return nil, err
            end
        end
    end

    if self.db_index > 0 then
        local ok, err = red:select(self.db_index)
        if not ok then
            return nil, err
        end
    end

    return red, nil
end


local redis_release = function(self, red)
    local keepalive = rawget(self, "keepalive")
    local keepalive_max_idle_timeout = rawget(self, "keepalive_max_idle_timeout")
    local keepalive_pool_size = rawget(self, "keepalive_pool_size")
    if keepalive == keepalive_default then
        local ok, err = red:set_keepalive(keepalive_max_idle_timeout,
                                          keepalive_pool_size)
        if not ok then
            return nil, err
        end
        return ok, nil
    end

    local ok, err = red:close()
    if not ok then
        return nil, err
    end

    return ok, nil
end


local do_cmd = function(self, cmd, ...)
    local reqs = rawget(self, "_pipe_reqs") -- 使用rawget取字段值的原因同上。
    if reqs then
        reqs[#reqs+1] = {cmd, ...} -- 这种方式效率比table.insert高。
        return
    end

    local red, err = redis_connect(self) -- 将self传递给redis_connect本地函数。
    if not red then
        return nil, err
    end

    -- Execute redis command.
    local method = red[cmd]
    local result, err = method(red, ...)
    if not result then
        return nil, err
    end

    local ok, err = redis_release(self, red)
    if not ok then
        return nil, err
    end

    return result, nil
end


-- 需要对外暴露API的函数使用_M.func的方式，不能使用本地函数的方式。
_M.init_pipeline = function(self, n)
    self._pipe_reqs = table_new(n or 4, 0)
end


_M.cancel_pipeline = function(self)
    self._pipe_reqs = nil
end


_M.commit_pipeline = function(self)
    local reqs = self._pipe_reqs
    if reqs == nil or #reqs == 0 then
        return {}, "no pipeline"
    end

    self._pipe_reqs = nil  -- reset

    local red, err = redis_connect(self)
    if not red then
        return nil, err
    end

    red:init_pipeline()
    for _, cmd in ipairs(reqs) do
        local func = red[cmd[1]]
        table_remove(cmd, 1)

        func(red, unpack(cmd))
    end

    local results, err = red:commit_pipeline()
    if not results then
        return {}, err
    end

    local ok, err = redis_release(self, red)
    if not ok then
        return nil, err
    end

    return results, nil
end


-- 由于redis的subscribe命令和常规的命令的处理流程不一样，所以也需要封装一下。
_M.subscribe = function(self, channel)
    local red, err = redis_connect(self)
    if not red then
        return nil, err
    end

    local res, err = red:subscribe(channel)
    if not res then
        return nil, err
    end

    res, err = red:red_reply()
    if not res then
       return nil, err
    end

    res, err = red:unsubscribe(channel)
    if not res then
        return nil, err
    elseif res[1] ~= "unsubscribe" then
        repeat
            res, err = red:read_reply()
            if not res then
                return nil, err
            end
        until res[1] == "unsubscribe"
    end

    local ok, err = redis_release(self, red)
    if not ok then
        return nil, err
    end

    return res, nil
end


--[[ opts i.e., mast be a table, and items can be none or many.
opts = {
    host = "127.0.0.1",
    port = 6380,
    db_index = 1,
    auth = "your password",
    timeout = 2000, -- 2 seconds
    -- If this value is not "on",
    -- keepalive_max_idle_timeout and keepalive_pool_size will be "nil".
    keepalive = "off",
    keepalive_max_idle_timeout = 20000, -- 20 seconds
    keepalive_pool_size = 200,
}
]]
_M.new = function(self, opts)
    local opts = opts or {}
    local host = opts.host or host_default
    local port = opts.port or port_default
    local db_index = opts.db_index or db_index_default
    local auth = opts.auth or auth_default
    local timeout = opts.timeout or timeout_default
    local keepalive = opts.keepalive or keepalive_default
    local keepalive_max_idle_timeout = opts.keepalive_max_idle_timeout
                                       or keepalive_max_idle_timeout_default
    local keepalive_pool_size = opts.keepalive_pool_size
                                or keepalive_pool_size_default

    if keepalive ~= keepalive_default then
        keepalive_max_idle_timeout = nil
        keepalive_pool_size = nil
    end

    local red = {
        host      = host,
        port      = port,
        db_index  = db_index,
        auth      = auth,
        timeout   = timeout,
        keepalive_max_idle_timeout = keepalive_max_idle_timeout,
        keepalive_pool_size = keepalive_pool_size,
        _pipe_reqs = nil, -- For pipeline requests to redis.
    }

    setmetatable(red, mt)
    return red
end


-- self是table自身隐藏的代表table本身的属性。
-- 从这里将self一层一层传递给上面的各种本地函数。
setmetatable(_M, {__index = function(self, cmd)
    local method =
        function(self, ...)
            return do_cmd(self, cmd, ...)
        end

    _M[cmd] = method
    return method
end})


return _M
```

## 使用方法举例

操作方法和官方redis库基本保持一致，只是省去了设置超时时间、创建连接、设置keepalive连接池、关闭连接等步骤，大大的方便我们的操作。

### redis常规命令用法

```lua
local redis = require "redis_wrap"

-- redis配置参数列表，如果不想开启keepalive，需要去掉下表中的keepalive两个元素，
-- 然后新加一个：keepalive = "off" 即可。
local redis_opts = {
    db_index = 1,
    timeout = 2000, -- 2 seconds.
    keepalive_max_idle_timeout = 10000, -- 10 seconds.
    keepalive_pool_size = 20,
}

-- 获取redis对象
local red = redis:new(redis_opts)

-- set 操作
local ok, err = red:set("dog", "an animal")
if not ok then
    ngx.say("failed to set dog: ", err)
    return
end

-- get 操作
local res, err = red:get("dog")
if not res then
    ngx.say("failed to get dog: ", err)
    return
end
```

### pipeline requests用法

```lua
local redis = require "redis_wrap"

local redis_opts = {
    db_index = 1,
    timeout = 2000, -- 2 seconds.
    keepalive_max_idle_timeout = 10000, -- 10 seconds.
    keepalive_pool_size = 20,
}

local red = redis:new(redis_opts)

-- 参数可以省略，加参数是在已知命令条目数量的情况下，为了高效处理。
red:init_pipeline(4)

red:set("dog", "111")
red:set("cat", "222")
red:get("cat")
red:get("dog")

local results, err = red:commit_pipeline()
if not results then
    ngx.say("failed to commit the pipelined requests: ", err)
    return
end
```
