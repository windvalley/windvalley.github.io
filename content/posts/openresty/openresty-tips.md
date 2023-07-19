---
date: 2019-04-28T11:58:13+08:00
title: "OpenResty编程tips"
description: ""
slug: ""
externalLink: ""
authors:
  - windvalley
categories:
  - OpenResty
tags:
  - openresty
  - lua
  - tips
keywords:
  -
draft: false
---

## Lua 中定义和调用函数时点号和冒号的区别

定义`table`中的函数时和调用`table`中的函数时, 冒号`:`和点号`.`的区别是什么.

### 定义 function

```lua
local _M = {}  -- 定义一个空table
local mt = { __index = _M } -- 定义一个元表mt,
                            -- 它的index方法是去table _M中去检索要找的元素.

-- 定义 _M 的初始化方法
_M.new = function(self)   -- 虽然定义时用的是., 但是参数中也可以没有self,
                          -- 因为我们这个方法中没有去调用_M自身的数据;
                          -- 如果有其他参数, 必须使用self占第一个位置.
    local s = {
        a = 1,
        b = 2,
        c = 3,
    }
    -- 给调用该new方法者返回table s, 并且设置table s的元表为mt,
    -- 也就是当在table s中检索不到元素时, 就去table _M中去检索,
    -- 还是检索不到就返回nil.
    return setmetatable(s, mt)
end


-- 注意冒号方式定义函数时, _M:new = function()这种写法是错误的,
-- 必须是function _M:new()才可以.


-- 定义_M的其他方法
function _M.get(self, name) -- 函数中不涉及到调用_M内部数据的话, 这里可以不用self.
    ngx.say(self.a, self.b, self.c) -- 通过.的方式调用_M内部数据的话,
                                    -- .后不能用参数变量name
    ngx.say(self[name])  -- 要通过函数参数变量来调用_M内部数据, 需要用这种方式
end

-- 和上面函数等价的冒号写法如下:
function _M:get(name)
    ngx.say(self.a, self.b, self.c)  -- 函数内语句和上面的完全一样
    ngx.say(self[name])
end
```

### 调用 function

养成调用方法统一使用`:`的习惯, 统一风格.

```lua
local foo = _M:new()
local bar = 'a'

ngx.say(foo:get(bar)) -- 冒号调用方法, 等价于下面的点号调用方法
ngx.say(foo.get(foo, bar))  -- 点号调用方法
```

输出:

```txt
123
1

123
1
```

## 如何正确使用 Lua 中的 local

- lua 的变量, 可以引用字符串、table、函数等各种 lua 对象, 默认是全局的.

- 定义全局变量的坏处：

  - 在不同阶段的 lua 代码中存在命名冲突；
  - 全局变量是同一个 worker 下的所有请求共享的, 所以会存在不同请求同时处理同一个变量造成 race condition(资源竞争), 导致意想不到的错误结果.
  - 由上可知, 定义变量时尽量使用`local`来声明.

- 使用`local`声明变量后, 变量的有效范围是当前`block`级别的, `block`可以是条件语句块、循环语句块、函数块、文件中的代码块.
  举个例子, 在条件语句块中`local`声明的变量, 在条件语句块之外是无法访问的, 如果需要访问, 就需要在条件语句块之外`local`声明该变量, 在条件语句块中直接使用该变量, 不可再次`local`声明该变量.

- 一个变量在相同`block`下只能被`local`声明一次.

- 使用 local 对 API 进行加速, 比如

```lua
local ngx_var = ngx.var
local ngx_ctx = ngx.ctx -- 注意这个不能写在模块级别, 要写在函数级别
```

## Lua 中 return 的用法详解

- 用在函数块最后一行, `return`主要是用于从函数中返回结果, 不会终止代码文件继续执行;

- 用在条件代码块时, `return`不论是否放在代码块的最后一行, 都会最后一个执行, 用于终止当前代码文件的运行, 代码文件后续的所有代码都不会执行了;

- 用于循环语句时, `return`用于终止当前代码文件的运行, 后面的所有代码都不会执行了, 而`break`是终止循环继续运行, 注意他们的区别;

- `return`不能直接用于代码文件级别;

- `do return end`一般用于调试代码的场景使用：
  - 用于函数中时, 可放置在函数中代码的中间, 这样函数剩余的代码部分不会被执行, 不会终止代码文件执行;
  - 用于条件、循环语句中时, 可放置在代码块的中间, 会在此处终止代码文件执行;
  - 直接用于代码文件级别, 会在`return`所在处终止代码文件执行.

> 注意: <br>
> 以上说的终止代码文件执行, 终止的是`return`所在的当前文件(模块)级别, 并不会影响其他执行阶段; <br>
> 至于是否会终止当前执行阶段, 就要看当前执行阶段的代码块中是否有`return`了. <br>
> 如果`return`只是出现在`require`的模块中, 这个`return`也不会影响到当前执行阶段.

## 使用 resty 写 Lua 脚本的技巧

### 脚本第一行

```lua
#!/usr/local/openresty/bin/resty 或 #!/usr/bin/env resty
```

### 指定 require 库时需要的搜索路径

```lua
#!/usr/local/openresty/bin/resty -I/usr/local/openresty/nginx/conf
```

或者

```lua
#!/usr/local/openresty/bin/resty

package.path = "/usr/local/openresty/nginx/conf/?.lua;" .. package.path
```

注意，下面这样不可以，会报错：

```lua
#!/usr/bin/env resty -I/usr/local/openresty/nginx/conf
```

### 中断脚本执行的方法, 用于调试

```lua
ngx.exit(0)
```

或者

```lua
do return end
```

### 书写版本和脚本名称, 养成好习惯

```lua
local name = "create_iplib_file"
local _VERSION = "0.1"
```

### print()

`print("")`在 resty 脚本中将不再是`ngx.log(ngx.NOTICE, "")`

而是等价于：

`io.stdout:write("")`

所以在`resty`脚本中可以直接使用`print`来打印输出到终端。

## 判断元素是否在 table 中

```lua
local item_if_in_table = function(item, tablename)
    for i=1, #tablename do
       if tablename[i] == item then
           return true
       end
    end
end
```

## 截取 table 中的元素

截取 table 中的前多少个元素，返回一个截取后的 table

```lua
local table_top_slice = function(table_name, top_number)
    local n = 0
    local result_table = {}
    for k, v in ipairs(table_name) do
        if n == top_number then
            break
        end
        table.insert(result_table, v)
        n = n + 1
    end

    return result_table
end
```

## 读取操作系统中的文件, 返回一个 table

```lua
local _M = {}

local function load_file()

    local iplib_file = ngx.config.prefix() .. "/conf/lua/lib/ipipnet.txt"
    local iplib_func = io.lines(iplib_file)
    local iplib_list = {}

    while true do
        local line = iplib_func()
        if line then
            table.insert(iplib_list, line)
        else
            break
        end
    end

    return iplib_list
end

-- model level variable, share data for all requests of the worker.
-- so, do not put this in the function level(request level).
local iplib_list = load_file()

_M.get_iplib_table = function()
    return iplib_list
end


return _M
```

## 解析不到 POST 上来的 body 体的问题

客户端 post 数据总是提示错误, `openresty lua`获取不到 body 体的问题.

```python
# python客户端
r = requests.post(server_api, headers=headers, json=json_post_data)

# openresty lua
ngx.req.read_body()
local body = ngx.req.get_body_data()
ngx.say('##' .. body .. '##')

# 解决方案
# 由于post数据较大造成
# 在nginx.conf中的location上下文中加入
client_max_body_size 1m;
client_body_buffer_size 1m;
```

## POST 体中包含中文的解析

客户端向 openresty post 的 json 字符串, 如果包含中文, 那么 openresty 需要进行两次`json.decode`才能转化为`lua table`数据结构.

```lua
ngx.req.read_body()
local body = ngx.req.get_body_data()
local body_data_ = json.decode(body)
local body_data = json.decode(body_data_)
```

## 注意 typo 错误带来的排查困扰

使用 redis 时, `error_log`日志始终报这个错误:

```text
[error] 19981#19981: *48463 attempt to send data on a closed socket: u:0000000000000000, c:0000000000000000, ft:0 eof:0,
```

困扰了很多天, 最后发现是因为: `red:set_timeout(2000)`这句拼写错误,
错误的写成了`red:set_timemout(2000)`.

## null 的比较

从 redis 读取的字符串如果通过`ngx.say`打印出的是`null`,
这个`null`在 lua 里是用`cjson.null`表示的, 所以这个时候我们如果做条件判断,
一定要和`cjson.null`进行比对, 否则会和预期不一样.

其他情况和`null`进行比较的时候要使用`ngx.null`替代, 具体情况具体分析.

## ngx.say 和 ngx.print 的使用阶段

`ngx.say`和`ngx.print`使用阶段为:
`rewrite_by_lua*, access_by_lua*, content_by_lua*`,
如果出现在`rewrite`, 就不会执行`access`和`content`阶段了,
但之后的`header_filter`、`body_filter`、`log`三个阶段还是会执行的,
只是这三个阶段不能执行`ngx.say`API, 其他代码还是会执行的.
同理出现在`access`阶段, 就不会执行`content`阶段了.
也就是说给客户端响应内容的只能是`rewrite`、`access`和`content`的阶段.

## ngx.var API 的几个用法

注意 ngx.var 几种不同用法的区别.

表示 http 头部时, 比如 content-type 头: `ngx.var.http_content_type`

表示 nginx 内置的变量时, 比如`$remote_addr`: `ngx.var.remote_addr`

表示在 nginx 配置文件中自定义的变量时,
比如`set $error_from "-"`: `ngx.var.error_from`

## require 一个模块得到的大都是一个 table 数据类型

`require`一个模块之后, 得到的大都是一个`table`数据类型(因为一般模块都是通过`table`来封装的), 比如:

```lua
local ngx_re = require "ngx.re"
ngx.say(type(ngx_re))
```

输出: `table`

另外需要注意, 函数是不能被序列化的(所谓序列化就是从 table 类型转化为 json 字符串类型), 所以如果 table 中包含了函数, 通过`cjson.encode`进行序列化是就会报错, 如下这样的 table:

```lua
local _M = {
    bar = function()
              return "hello world"
          end,
    foo = 2,
}
```

这个 table `_M`等价于:

```lua
local _M = { foo = 2 }
function _M.bar()  -- 注意这里, 由于前面声明_M变量时已经用了local, 所以这里不能再加local了.
    return "hello world"
end
```

## 使用 lua_cache_code 需要注意的

当`lua_cache_code`指令值是`off`的时候, `*_by_lua_file`指定的 lua 文件代码修改后,
可以不需要 reload openresty 重新加载即可立即生效.
但`*_by_lua`和`*_by_lua_block`指令对应的 lua 代码内容就不可以,
因为这些代码写在了 nginx 配置文件里面.
当然这个指令一般只用于在代码测试阶段, 可以省去反复 reload openresty 的麻烦,
不过要特别注意, 如果测试通过共享变量存储数据或是`lrucache`等,
这个指令一定要是`on`.

## ngx.timer.\* 的执行与请求无关

`ngx.timer.*`的执行是在独立的协程里完成的,
也就是说它的运行与当前的请求没有关系.

在事件循环中, Nginx 会找出到期的`timer`(即需要开始执行里面的回调函数了),
并在一个独立的协程中执行对应的 Lua 回调函数.

## 请求级别的变量放到函数内

当代码文件作为模块使用时, 谨记要把请求级别的变量放到函数内,
否则同一`worker`下的所有请求都会共享这个变量内容, 导致内容输出错误.

## table 作为字典使用时如何排序

`table`作为列表时, 是有序的, 作为字典使用时是无序的,
如果为了序列化后显示先后顺序, 可以这样做:

```lua
-- 也就是每一个键值对都加上一个{}包含住
value_table = {
     { cdn_type = cdn_type },
     { bandwidth_time = bandwidth_time },
     { bandwidth_total = bandwidth_total },
}
```

## table 中键值对的值是 nil 时需要注意的

如果`table`中的某些键值对的值是`nil`,
通过`cjson.encode(table)`是打印不出来该键值对的.

## 获取 table 的长度的方法

```lua
#tablename   -- 只能获取table是数组元素的, #也可以获取字符串的长度
table.nkeys(tablename)  -- 数组元素和键值对通用
```

## 判断一个 table 是否为空

`table.nkeys(tablename) ~= 0`

## http 库相关

`tokers/lua-resty-requests`库同时请求后端数超过 40 个左右的时候,
部分请求会超时报错: `lua tcp socket queued connect timed out`;

`ledgetech/lua-resty-http`这个库比较完善, 没有上面的问题.

前一个库每查询一次后端就新建一个`tcp`连接,
后一个库所有查询只用一个`tcp`连接, 高下立见.

## 使用 resty.mysql 库连接 mysql8.0 以上报错

`Client does not support authentication protocol requested by server; consider upgrading MySQL client: 1251 08004 sql`

解决方法:
`alter user 'root'@localhost IDENTIFIED WITH mysql_native_password by '123456';`

## 将某个文件加载到内存时如何指定文件位置

```lua
local ngx_config_prefix = ngx.config.prefix  -- 即启动openresty时-p指定的路径.

local iprepo_file = ngx_config_prefix() .. "etc/iprepo_v4_v6.txt"
```
