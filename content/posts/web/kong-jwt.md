---
title: "使用Kong的JWT插件"
date: 2018-01-10T14:44:38+08:00
lastmod: 2018-01-10T14:44:38+08:00
categories:
    - Web
tags:
    - web
    - gateway
keywords:
    -
draft: false
---

## Kong的JWT插件介绍

官方文档:
https://docs.konghq.com/hub/kong-inc/jwt/

需要先详细阅读该文档.

需要注意的几点:

- 用户的基础信息和权限信息存储在kong本地的数据库中,
  我们可以写一个权限的后台管理系统(用户表等权限表单独建立),
  通过调用kong提供的管理api来管理同步这些用户信息.

- 每个用户都有`key`和`seceret`字段, `key`表示`JWT`的`payload`字段中的签发者,
  `seceret`表示用于生成`JWT`第三个字段(签名)的密钥.

- 该`JWT`插件只验证`JWT`合法性, 不签发`JWT`, 但是提供用于签发`JWT`所需的`seceret`、
  签发者(`key`)、用户信息等的`API`.

```bash
# {consumer}可以是用户名或者是用户ID
curl -X GET http://kong:8001/consumers/{consumer}/jwt
```

输出:

```json
{
    "data": [
        {
            "consumer_id": "39f52333-9741-48a7-9450-495960d91684",
            "id": "3239880d-1de5-4dbc-bccf-78f7a4280f33",
            "created_at": 1491430568000,
            "key": "c5a55906cc244f483226e02bcff2b5e",
            "algorithm": "RS256",
            "secret": "b0970f7fc9564e65xklfn48930b5d08b1"
        }
    ],
    "total": 1
}
```

- 签发`JWT`的环节我们可以自己选择:
  - 可以在`kong`上写一个插件用于签发`JWT`
  - 可以在后端签发`JWT`
  - 可以在前端签发`JWT`(安全问题, 不推荐)

## JWT鉴权方案

### 企业内部的管理系统

`VUE -> LDAP -> KONG -> 各种后端API`

签发`JWT`的环节可以在`KONG`上, 也可以在后端服务上.

大概流程介绍:

1. 用户输入用户名和密码, `VUE`通过`LDAP`进行认证, 成功则拿到用户基础信息,
   表明基础认证成功.

2. 然后携带用户信息请求`KONG`上的用于签发`JWT`的接口(`lua`插件实现),
或者某个后端接口, 该接口也可以接入`KONG`, 但`KONG`不做鉴权.

3. 签发`JWT`和返回用户目录权限信息
    - 签发`JWT`的环节在`KONG`上的情况:
该`Lua`接口从`KONG`的数据库中取出用户的`key`(签发者)和`secret`(密钥), 结合用户名、过期时间等信息签发`JWT`.
    - 签发`JWT`的环节在后端服务的情况:
后端从`KONG`提供的接口获取用户的`key`和`secret`, 同时该接口从本地数据库中拿到用户的目录权限等信息, 以`JSON`格式统一响应给`VUE`.

4. `VUE`拿到`JWT`, 将`JWT`写入到浏览器本地的`localstorage`和`cookie`,
同时根据拿到的用户目录权限信息展示该用户拥有权限的前端页面.

5. `VUE`在请求其他后端接口的时候可以通过`url`参数和`Authorization`头携带`JWT`,
也可以通过`cookie`来携带`JWT`(KONG默认没有开启从cookie读取jwt, 可配置).

6. `KONG`接收到请求, 如果请求的`api`开启了`JWT`, 则会进行`JWT`校验,
签名校验和过期时间校验都通过后将转发该`api`到相应的后端服务.

> 注意: <br>
> 验证JWT是否过期也可以在VUE处先做了, 但还是建议在后端做, 因为可以加入一些其他的逻辑,
> 比如额外引入Redis来做用户登录过期判断, 从而可以将JWT的过期时间设置的比较短, 提高安全性.

### 对外Web应用

`VUE -> KONG -> 各种后端API`

整体流程和上述一致, 说一下不一样的地方:

- 不需要`LDAP`认证, 多了一个用户注册环节
- 用户管理在后端服务上服务上做, 包括签发`JWT`.
