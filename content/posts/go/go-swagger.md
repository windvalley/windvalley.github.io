---
title: "为基于Gin的Web项目生成Swagger在线API文档"
date: 2018-12-31T10:24:36+08:00
lastmod: 2018-12-31T10:24:36+08:00
categories:
  - Go
tags:
  - go
  - web
  - gin
keywords:
  -
draft: true
---

## 安装 swag 命令

```bash
go get -u github.com/swaggo/swag/cmd/swag
```

> 注意: <br>
> 确保提前配置了 Go 环境.

```bash
# Go
GOPATH=$HOME/go
GOBIN=$GOPATH/bin
PATH="$GOBIN:$PATH"
GOPROXY=https://goproxy.io
export GOPATH GOBIN PATH GOPROXY
```

验证是否安装成功:

```bash
swag -v
```

## 安装 gin-swagger

```bash
go get -u github.com/swaggo/gin-swagger
go get -u github.com/swaggo/files
```

## 为 swagger 添加访问路由

路由文件: `router/router.go`

```go
package router

import (
	"net/http"

	"github.com/gin-gonic/gin"
    ginSwagger "github.com/swaggo/gin-swagger"
	"github.com/swaggo/gin-swagger/swaggerFiles"

    _ "yourapp/docs"  // for swagger
)

func Group() {
	runmode := config.Conf().Runmode

	switch runmode {
	case "release":
		gin.SetMode(gin.ReleaseMode)
	case "test":
		gin.SetMode(gin.TestMode)
	case "debug":
		gin.SetMode(gin.DebugMode)
	default:
		panic("unknown runmode")
	}

	router := gin.New()
	if runmode == "debug" {
		router.Use(gin.Logger())
	}
	router.Use(gin.Recovery())

    // 只在debug和test模式下启用swagger
	if runmode == "debug" || runmode == "test" {
		// NOTE: execute `swag init` in project root dir after updating docs.
		// path: /doc/index.html
		router.GET("/doc/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
	}

    ...
}
```

## 写文档注释

为`main`函数写文档注释: `main.go`

```go
// @title xxx管理系统 API
// @version 0.1
// @description  ### 项目功能:
// @description - xxx
// @description - xxx

// @contact.name API维护者
// @contact.url http://www.swagger.io/support
// @contact.email i@sre.im

// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @host xxx.sre.im:8080
// @BasePath /v1/xxx
func main() {

    ...
```

为其他`API`方法写文档注释:

```go
// @Summary xxx
// @Description xxx
// @Tags idcs
// @Accept  json
// @Produce  json
// @Param   id path uint true "2"
// @Param   idccode query string false "shxx"
// @Param   billmethod query string false "95"
// @Success 200 {string} string
// @Router /idcs/{id} [put]
func UpdateIDC(c *gin.Context) {
    ...
```

`Param`的类型有:

```text
query
path
header
body
formData
```

`Param`的数据类型有:

```text
string (string)
integer (int, uint, uint32, uint64)
number (float32)
boolean (bool)
user defined struct
```

## 生成文档注释

在项目根目录下执行:

```bash
swag init
```

会自动生成如下目录和文件:

```text
docs/
├── docs.go
├── swagger.json
└── swagger.yaml
```

每次文档有更新, 都需要重新执行一次:

```bash
swag init
```

## 访问 swagger 在线 API 文档

编译项目并运行后, 访问:

```
http://localhost:8080/swagger/index.html
```

## 官方文档

> https://github.com/swaggo/swag/blob/master/README.md <br> > https://swaggo.github.io/swaggo.io/declarative_comments_format/general_api_info.html
