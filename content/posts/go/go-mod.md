---
title: "Go的包管理工具: go mod"
date: 2019-03-09T10:59:45+08:00
lastmod: 2019-03-09T10:59:45+08:00
categories:
    - Go
tags:
    - go
keywords:
    -
draft: false
---

## 版本要求和环境变量

> `go version` >= 1.11

相关的环境变量: `GO111MODULE`

查看该环境变量的当前值: `go env GO111MODULE`

- 结果为空或`auto`, 这是默认值, 表示在非`$GOPATH/src`目录下才启用`go mod`功能.
- 值为`off`, 则表示不启用该功能,
  使用旧有的`vendor`目录或`$GOPATH/src`目录查找依赖.
- 值为`on`, 则表示任何情况都启用该功能,
  此时将不会再去`$GOPATH/src/`目录下寻找依赖.

如果想根据自己的情况改变这个值, 比如设置为`on`:

`echo "export GO111MODULE=on" >> ~/.bash_profile`

一般我们保持默认值即可.

## 初始化一个module

`go mod init yourmodulename`

使用举例:

创建项目目录

```bash
mkidr app_name
cd app_name
go mod init app_name  # 这里的module名字app_name可以和项目目录名字不一致
```

以上会在当前项目目录下生成一个文件`go.mod`, 内容如下:

```text
module app_name

go 1.13
```

> `go.mod`文件一旦创建, 它的内容将会被`go toolchain`全面掌控,
`go toolchain`会在各类命令执行时, 比如`go get`、`go build`、`go mod`等,
同步修改和维护`go.mod`文件.

## 管理项目的依赖包

`go run *.go`

写好代码后, 执行`go run *.go`或`go build`等命令会自动将依赖的包下载、解压来使用, 非常智能.

`go mod tidy`

也可以先执行这个命令, 如果本机还没有相关的依赖包,
就会自动将依赖的zip压缩包下载到`$GOPATH/pkg/mod/cache/download/`目录下,
然后将下载的压缩包解压到`$GOPATH/pkg/mod/`目录下;
最后生成`go.sum`文件.

另外, 如果本项目移除了一些依赖包, 或更换了依赖包的版本, 执行`go mod tidy`,
就会自动同步变更`go.mod`和`go.sum`文件.

`go.mod`

```go
module app_name

go 1.13

require (
	github.com/fsnotify/fsnotify v1.4.7
	github.com/influxdata/influxdb v1.7.9
	github.com/soniah/gosnmp v1.22.0
	github.com/spf13/viper v1.6.1
)
```

`go.sum`

```go
cloud.google.com/go v0.26.0/go.mod h1:aQUYkXzVsufM+DwF1aE+0xfcU+56JwCaLick0ClmMTw=
github.com/BurntSushi/toml v0.3.1 h1:WXkYYl6Yr3qBf1K79EBnL4mak0OimBfB0XUf9Vl28OQ=
github.com/BurntSushi/toml v0.3.1/go.mod h1:xHWCNGjB5oqiDr8zfno3MHue2Ht5sIBksp03qcyfWMU=
github.com/OneOfOne/xxhash v1.2.2/go.mod h1:HSdplMjZKSmBqAxg5vPj2TmRDmfkzw+cTzAElWljhcU=
github.com/alecthomas/template v0.0.0-20160405071501-a0175ee3bccc/go.mod h1:LOuyumcjzFXgccqObfd/Ljyb9UuFJ6TxHnclSeseNhc=
github.com/alecthomas/units v0.0.0-20151022065526-2efee857e7cf/go.mod h1:ybxpYRFXyAe+OPACYpWeL0wqObRcbAqCMya13uyzqw0=
github.com/armon/consul-api v0.0.0-20180202201655-eb2c6b5be1b6/go.mod h1:grANhF5doyWs3UAsr3K4I6qtAmlQcZDesFNEHPZAzj8=
github.com/beorn7/perks v0.0.0-20180321164747-3a771d992973/go.mod h1:Dwedo/Wpr24TaqPxmxbtue+5NUziq4I4S80YR8gNf3Q=
github.com/beorn7/perks v1.0.0/go.mod h1:KWe93zE9D1o94FZ5RNwFwVgaQK1VOXiVxmqh+CedLV8=
github.com/cespare/xxhash v1.1.0/go.mod h1:XrSqR1VqqWfGrhpAt58auRo0WTKS1nRRg3ghfAqPWnc=
...
```

> `go.mod`: <br>
> 该文件是使用`go mod`管理项目所必须的最重要的文件,
> 因为它描述了当前项目的元信息, 每一行都以一个动词开头, 目前有以下`5`个动词: <br>
> `module` - 用于定义当前项目的模块路径; <br>
> `go` - 用于设置预期的 Go 版本; <br>
> `require` - 用于设置一个特定的模块版本; <br>
> `exclude` - 用于从使用中排除一个特定的模块版本; <br>
> `replace` - 用于将一个模块版本替换为另外一个模块版本.

---

> `go.sum`: <br>
> 该文件详细罗列了当前项目直接或间接依赖的所有模块版本,
> 并写明了那些模块版本的`SHA-256`哈希值,
> 以备`Go`在今后的操作中保证项目所依赖的那些模块版本不会被篡改. <br>
> `go mod`安装依赖包的原则是先拉取最新的`release tag`, 若无`tag`,
> 则拉取最新的`commit`. 最后会自动生成一个`go.sum`文件来记录`dependency tree`.

## 更新依赖包

`go list -m -u all`: 检查可以升级的依赖包.

`go get -u`: 升级所有依赖包.

`go get package@version`: 升级某个依赖包到指定的版本号.

## vendor模式介绍

`go mod vendor`

自动在项目目录下创建`vendor`目录, 并把相关依赖包拷贝至此目录.

然后通过设置环境变量`GOFLAGS=-mod=vendor`来启用`vendor`模式, 此时,
依赖包将会优先从该目录查找.

> 注意: `go mod`不推荐使用vendor模式.

## go mod 其他命令介绍

`download`
将依赖包下载到`$GOPATH/pkg/mod/cache/download/`

`edit`
编辑`go.mod`文件, 具体用法文档: `go help mod edit`

`graph`
打印包的依赖关系.

`verify`
检查我们当前项目(包)所依赖的所有的包是否从下载下来之后没有修改过,
如果没修改过就输出`all modules verified`, 否则输出有变更的包的名称.

`why`
解释为什么需要依赖包, 具体说明: `go help mod why`

## 总结

使用`go mod`管理项目, 最常用的命令是`go mod init yourmodulename`和`go mod tidy`;

依赖包默认存储在`$GOPATH/pkg/mod/`目录下;

最好不要使用`vendor`模式, 更不要将`vendor`目录提交到版本控制;

需要提交`go.mod`到版本控制, 可以忽略`go.sum`.
