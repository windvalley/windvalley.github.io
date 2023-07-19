---
title: "OpenResty编码规范"
date: 2019-06-01T23:52:42+08:00
authors:
    - windvalley
categories:
    - OpenResty
tags:
    - conventions
    - lua
    - openresty
draft: false
---

### 变量名

所有变量都使用local声明, 限制作用域. 变量名小写, 使用下划线连接.

### 常量名

常量名全部大写, 多个单词通过下划线`_`连接.

### 文件名

lua文件名使用小写字母, 多个单词通过下划线`_`连接.

### 函数

- 函数名称全部小写, 使用下划线`_`进行连接

- 单个函数不要过长, 尽量拆分为多个子函数.

### 对函数缓存

使用本地变量来缓存函数(函数指针, 函数的内存地址), 加速之后的请求的调用.

这里要强调一点, 标准API(函数)对应的本地变量别名一定不能随性命名, 会给团队其他的开发人员造成困惑, 需要统一和标准API名字一样, 只是把点替换成下划线, 其他保持不变, 比如：

```lua
local ngx_var = ngx.var
local ngx_ctx = ngx.ctx
local ngx_log = ngx.log
local setmetatable = setmetatable
local tonumber = tonumber
```

### 空行与空格

- 文件级别的函数之间空2个空行
- 代码不同意群的block之间空一个空行
- 各种运算符前后空一个空格, 逗号后空一个空格

### 记录版本号

模块要有版本号和模块名字, 比如：

```lua
local _M = { _VERSION = '0.09' }
_M.name = "redis"
```

如果不是模块, 而是代码文件, 也要加上版本信息：

需要使用注释的方式标明, 如果使用`local _VERSON = 0.1`这种定义本地变量的方式, luacheck检查的时候会报定义的变量未使用的错误.

```lua
-- _VERSION = 0.1
```

### 代码长度

控制单行代码的长度, 不要超过`80`个字符. 如果单行代码过长, 可以直接换行(注意lua的换行不能使用`\`, 直接换行即可), 第二行注意要缩进, 保持代码美观.

### 注释

代码中晦涩难懂的地方, 一定要加上注释.

单行注释使用`-- 后接一个空格`.

多行注释使用`--[[]]`, 一定要注意`--`和`[[`之间无空格, 否则无法换行. 举例：

```lua
-- this is pine

--[[ this is pine
this is apple
]]
```

如果多行注释中存在`[[`字符, 需要这样:

```lua
--[=[
this is [[apple]]
]=]
```

### 字符串引用

- lua中的字符串最好都统一使用`""`包含(当然有些时候使用`[[]]`是必须的), 就像python中都习惯统一使用`''`一样.

- 特别长的字符串使用`[[]]`来代替`""`, 可以直接折行, `""`无法在字符串内部直接折行.

### 模块

建议通过模块的方式编写项目具体功能, 然后通过require的方式关联, 清晰简洁. 模块变量名统一使用`_M`, 尽量不要自定义名字.

不要直接在模块`_M`中定义函数, 会导致内存增加, 可通过`function _M.funcname()`的方式定义或先定义本地函数, 最后通过`_M.funcname = funcname`的方式关联.

```lua
-- 通过元表实现lua的面向对象编程.

local _M = {}
local mt = { __index = _M }

_M.new = function()
    data = {
        foo = 1,
        bar = {},
    }
    return setmetatable(data, mt)
end

_M.func1 = function(self)
    self:_func2()
end

return _M
```

不想对外暴露的模块函数需要使用一个下划线修饰, 这样开发者一看代码就明白哪些模块函数是要给外部调用的, 哪些不是. 接着上面的代码举例：

```lua
_M._func2 = function(self)
     return self.foo
end

-- 为什么不直接使用本地函数呢, 比如:
local func2 = function()
    return self.foo
end
-- 很明显, 本地函数无法调用模块类的内部数据foo, 但有的场景下是可以通过传参的形式解决这个问题的.
```

### 记录日志

在可能出错的地方记录日志到nginx的`error.log`文件, 比如：

```lua
local ok, err = red:connect(addr)
if not ok then
    ngx.log(ngx.ERROR, "msg: ", err)
end
```

### ngx.exit

使用`ngx.exit(0)`、`ngx.exit(403)`等操作时, 前面使用`return`修饰.

```lua
return ngx_exit(500)
return ngx.exit(0)
```

### 静态代码检查工具

lua代码尽量通过`luacheck`和`lj-releng`这两个lua的静态代码检测工具的规范检测, 注意有时候`luacheck`的要求可能较苛刻, 对于有些场景可能检测的结果也不一定就很对, 所以不必严格遵守, 能做到提醒我们的作用就很好. 尽量完全处理`lj-releng`的提示信息.

使用这两种代码规范的检查工具, 不仅对我们写出赏心悦目、方便阅读的代码风格有好处, 而且还可以帮助我们发现代码中潜在的问题或错误.

持续更新...
