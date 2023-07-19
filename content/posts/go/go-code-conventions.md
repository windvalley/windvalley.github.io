---
title: "Go语言编码规范"
date: 2018-03-27T09:42:23+08:00
authors:
  - windvalley
categories:
  - Go
tags:
  - Go
  - conventions
draft: false
---

## 命名规范

### 文件名

- 整个应用或包的主入口文件应当是`main.go`或与应用名称简写相同.

- 文件名如果包含多个单词通过下划线连接.

### 包名

- 包名应该简短、清晰、有意义, 应全部是小写字母, 不要使用下划线或者驼峰方式连接.
  Go 中很多项目的包的名字是一样的, 但包的路径必需唯一.
  看看官方库中的例子, 宁愿多个单词直接相连, 也不使用下划线或者驼峰方式相连:

  ```text
  /src/cmd/addr2line
  /src/cmd/objdump
  /src/image/internal/imageutil
  /src/index/suffixarray
  ```

- 包名必须和目录名一致, 尽量采取有意义、简短的包名, 不要和标准库冲突.

- 包名以及包所在的目录名, 不要使用复数, 例如: 应该是 `net/url`, 而不是 `net/urls`.

- 项目名可以看作是特殊的包名, 可以通过中划线来连接多个单词.

### 常量

- 常量名必须遵循驼峰式, 首字母根据访问控制决定使用大写或小写.

- 如果是枚举类型的常量, 需要先创建相应类型:

  ```go
  type Scheme string

  const (
      HTTP  Scheme = "http"
      HTTPS Scheme = "https"
  )
  ```

### 变量

- 变量为单独的单词时, 可导出变量开头字母大写, 私有变量全小写;
  专有名词时, 可导出变量全大写, 私有变量时全小写.

- 多单词组成的变量名称遵循驼峰法, 可导出(公有)变量开头字母要大写,
  不可导出(私有)变量开头字母小写. 比如: 可导出变量为 `UserName`, 对应私有变量为 `userName`.
  不过对于专有名词, 比如 API、ID、HTTP、IP 等, 需要这样处理,
  举例: 私有变量 apiClient, 对应可导出变量为 APIClient.

  常见的专有名词列表:

  ```go
  // A GonicMapper that contains a list of common initialisms taken from golang/lint
  var LintGonicMapper = GonicMapper{
  	"API":   true,
  	"ASCII": true,
  	"CPU":   true,
  	"CSS":   true,
  	"DNS":   true,
  	"EOF":   true,
  	"GUID":  true,
  	"HTML":  true,
  	"HTTP":  true,
  	"HTTPS": true,
  	"ID":    true,
  	"IP":    true,
  	"JSON":  true,
  	"LHS":   true,
  	"QPS":   true,
  	"RAM":   true,
  	"RHS":   true,
  	"RPC":   true,
  	"SLA":   true,
  	"SMTP":  true,
  	"SSH":   true,
  	"TLS":   true,
  	"TTL":   true,
  	"UI":    true,
  	"UID":   true,
  	"UUID":  true,
  	"URI":   true,
  	"URL":   true,
  	"UTF8":  true,
  	"VM":    true,
  	"XML":   true,
  	"XSRF":  true,
  	"XSS":   true,
  }
  ```

- 若变量类型为 `bool` 类型, 则名称应以 `Has`, `Is`, `Can` 或 `Allow` 开头, 例如：

  ```go
  var hasConflict bool
  var isExist bool
  var canManage bool
  var allowGitHook bool
  ```

### 函数或方法名

- 命名规则基本遵从变量的命名规则.

- 函数尽量少用全局变量, 应该通过参数传递, 使每个函数都是"无状态"的, 减少耦合, 方便分工和单元测试; 如果函数参数比较多, 可以将相关参数定义成结构体传递.

### 结构体名

- 结构体名不应该是动词, 应该是名词.
- 其他命名规则同变量的命名规则.

### 接口名

- 为了更方便区分, 可导出接口名前统一加大写`I`, 私有接口名前统一加小写`i`, 然后紧跟具体对象名称的大写字母, 比如:

  ```go
  // 可导出接口
  type IDemo interface {
    Create()
    Update()
    Get()
    Delete()
  }

  // 私有接口
  type iDemo interface {
    Create()
    Update()
    Get()
    Delete()
  }
  ```

## import 包导入规范

一个源代码文件导入的包(`package`)可以分为 4 类:

- 标准库(`package`)
- 第三方库(`package`)
- 组织内(公司内)其他项目的包(`package`)
- 当前项目的子包(`package`)

导入规范:

- 每类之间使用空行分隔
- 不要使用`.`来导入包
- 不要使用相对路径导入子包(比如: `../subpackage`), 示例:

  ```go
  import (
      // 标准库
    "fmt"
    "html/template"
    "net/http"
    "os"

      // 第三方库
    "github.com/codegangsta/cli"
    "gopkg.in/macaron.v1"

      // 组织内或公司内其他项目的包
    "github.com/gogits/git"
    "github.com/gogits/gfm"

      // 当前项目的子包
    "github.com/gogits/gogs/routers"
    "github.com/gogits/gogs/routers/repo"
    "github.com/gogits/gogs/routers/user"
  )
  ```

## 注释规范

- 所有导出对象(大写字母开头)都需要注释说明其用途, 非导出对象根据复杂情况进行选择性注释.

- 包、函数、方法和类型的注释说明需要是一个完整的句子, 句子类型的注释首字母均需大写, 短语类型的注释首字母需小写.

- 注释的单行长度不要超过`120`个字符.

- 只使用单行注释符号`//`, 尽量不使用多行注释符号`/* */`.

- 包级别的注释就是对包的介绍, 只需在同个包的任一源文件中说明即可有效.

- 当某个部分等待完成时, 可用`TODO:`开头的注释来提醒维护人员.

- 当某个部分存在已知问题进行需要修复或改进时, 可用`FIXME:`开头的注释来提醒维护人员.

- 当需要特别说明某个问题时，可用`NOTE:`开头的注释.

- 在多段注释之间可以使用空行分隔加以区分, 如下所示:

  ```go
  // WithCancel returns a copy of parent with a new Done channel. The returned
  // context's Done channel is closed when the returned cancel function is called
  // or when the parent context's Done channel is closed, whichever happens first.
  //
  // Canceling this context releases resources associated with it, so code should
  // call cancel as soon as the operations running in this Context complete.
  func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
  	if parent == nil {
  		panic("cannot create context from nil parent")
  	}
  	c := newCancelCtx(parent)
  	propagateCancel(parent, &c)
  	return &c, func() { c.cancel(true, Canceled) }
  }
  ```

- 包注释, 包注释统一用 `//` 进行注释, 格式为 `// Package 包名 包描述`, 例如:

  ```go
  // Package context defines the Context type, which carries deadlines,
  // cancellation signals, and other request-scoped values across API boundaries
  // and between processes.
  package context
  ```

## 函数或方法定义规范

- 函数或方法的顺序一般需要按照依赖关系排序, 被依赖的函数放到前面.

- 函数参数传入变量和返回变量都以小写字母开头.

- 传入参数数量不能超过 5 个, 超过则使用 `struct` 替代.

- 多返回值最多返回 3 个, 超过三个请使用 `struct`.

- 应以对象第一个英文首字母的小写作为方法接收器的名称, 不要使用 `me`、`this`、`self`.

- 函数或方法的顺序一般需要按照依赖关系排序, 被依赖的函数放到前面.

- 结构附带的方法应置于结构定义之后, 按照所对应操作的字段顺序摆放方法.

- 如果一个结构拥有对应操作函数, 大体上按照 `CRUD`(增查改删) 的顺序放置结构定义之后.

- 如果一个结构拥有以 `Has`、`Is`、`Can` 或 `Allow` 开头的函数或方法,则应将它们置于所有其它函数及方法之前.

- 如果函数被一个变量引用作为一部分, 函数应该放到这个变量之后.

## 其他规范

- 单行代码不要超过`120`个字符, 如果超过需要进行分行.

- 变量声明尽量放在变量第一次使用的前面, 遵循就近原则.

- 如果魔法数字出现超过两次, 则禁止使用, 改用一个常量代替.

- 分组声明一般需要按照功能来区分, 而不是将所有类型都分在一组.

- 其他规范细节建议详细阅读: <https://github.com/xxjwxc/uber_go_guide_cn>

> 参考:
>
> https://golang.org/doc/code.html#PackageNames
>
> https://golang.org/doc/effective_go.html#package-names
>
> https://golang.org/doc/code.html
>
> https://golang.org/doc/effective_go.html
>
> https://github.com/unknwon/go-code-convention/blob/master/zh-CN/README.md
