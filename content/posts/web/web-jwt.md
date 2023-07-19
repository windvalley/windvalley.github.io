---
title: "JWT(Json Web Token)原理讲解"
date: 2018-01-08T10:03:36+08:00
date: 2018-01-08T10:03:36+08:00
categories:
    - Web
tags:
    - web
keywords:
    -
draft: false
---

## JWT简单原理

`JWT`的原理是服务器通过用户认证以后, 会生成一个`JSON`对象响应给用户.

简单示例:

```json
{
 "username": "David",
 "role": "admin",
 "expire": "2018/10/01 00:00:00"
}
```

之后用户与服务端通信的时候, 都要携带这个`JSON`对象.
服务器完全只靠这个对象认定用户身份.
为了防止数据被篡改, 服务器在生成这个对象的时候, 会加上签名.

服务器不用再保存任何`Session`数据, 服务器变成无状态的了, 从而可以比较容易实现服务器扩展.

## JWT基本流程

1. 用户通过用户名和密码登录系统, 发出登录请求

2. 服务端收到请求数据, 通过查询数据库中的用户名和密码对用户身份进行验证

3. 服务端验证通过后, 为该用户生成`Token`, 响应给用户

4. 用户将`Token`保存起来, 如果使用的是浏览器,
   一般自动保存到`Cookies`(`Application-Storage-Cookies`)中;
   用户之后的请求需携带该`Token`,
   如果用户使用的是浏览器,
   一般浏览器会自动通过请求头`Authorization: Bearer $token`携带该`Token`进行请求.

5. 服务端收到请求后, 首先验证`Token`, 通过后返回数据, 没通过返回验证失败信息

> 注意: <br>
> 服务端不需要保存`Token`, 只需要对`Token`中携带的各种信息进行验证即可, 验证过程一般为: <br>
> 通过携带的加密算法信息和服务端保存的密钥实时计算签名,
> 然后和`Token`的签名(`Token`的第3个字段)进行比对看`Token`是否正确、
> 通过携带的过期时间信息和当前时间比对验证是否过期等. <br>
> 无论用户访问服务端的哪台服务器, 只要可以通过`Token`验证即可.

## JWT数据结构

服务器为用户生成的`Json Web Token`示例:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.cThIIoDvwdueQB468K5xDc5633seEFoqwxjF_xSJyQQ
```

这是一个很长的字符串, 中间用点`.`分隔成三个部分:

```text
Header(头部)
Payload(装载的内容)
Signature(签名)
```

### Header(头部)

`Header`部分是一个`JSON`对象, 描述`JWT`的元数据:

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

`alg`属性表示签名的算法(`algorithm`), 默认是`HMAC SHA256`(通常写成`HS256`).

`typ`属性表示这个令牌(`token`)的类型(`type`), `JWT`令牌固定写为`JWT`.

最后将该`JSON`对象使用`Base64URL`算法转成字符串, 就是上面`JWT`示例的第一个部分.

### Payload(装载的内容)

`Payload`部分也是一个`JSON`对象, 用来存放实际需要传递的数据.

`JWT`规定了7个官方字段, 供选用:

```text
iss (issuer): 签发者
sub (subject): 主题
aud (audience): 接收JWT的一方
exp (expiration time): 过期时间
nbf (Not Before): 生效时间
iat (Issued At): 签发时间
jti (JWT ID): 编号
```

除了官方字段, 你还可以在这个部分定义私有字段, 比如:

```json
{
 "sub": "123456789",
 "name": "David",
 "admin": true
}
```

这个`JSON`对象同样使用`Base64URL`算法转成字符串, 就是`JWT`示例的第二部分.

> 注意: <br>
> `JWT`默认是不加密的, 任何人都可以读到, 所以不要把私密信息放在这个部分.

### Signature(签名)

`Signature`部分是对前两部分的签名, 用来防止数据被篡改.

首先, 服务端需要自定义一个密钥(`secret`), 一般是写在配置文件中, 不能泄露给用户.

然后, 使用`Header`里指定的签名算法(默认是`HMAC SHA256`), 按照下面的公式产生签名:

```text
HMACSHA256(
    base64UrlEncode(header) + "." +
    base64UrlEncode(payload),
    secret)
```

算出签名以后, 把`Header`、`Payload`、`Signature` 三个部分使用`.`号分隔拼成一个字符串, 这个字符串就是`JWT`(`Json Web Token`), 然后响应给用户.

### Base64URL算法

`Header`和`Payload`转换为字符串的算法是`Base64URL`.
这个算法跟`Base64`算法基本类似, 但有一些小的不同.

`JWT`作为一个令牌(`token`), 有些场景下用户可能会将它作为请求`URL`的参数,
比如: `https://yourapipath/?token=xxx`.

由于`Base64`有三个字符`+`、`/`和`=`, 在`URL`里面有特殊含义, 所以需要被替换掉:
`=`被省略、`+`替换成`-`, `/`替换成`_`, 这就是 `Base64URL` 算法.

## JWT的使用方式

用户收到服务器返回的`JWT`, 需要保存起来, 如果客户端是浏览器一般会储存在`Cookie`里面或`localStorage`.

此后, 用户每次与服务器通信, 都要带上这个`JWT`, 有几种方式携带`JWT`:

- 放在`HTTP`请求头`Cookie`里(不能跨域)
- 放在`HTTP`请求头`Authorization`里:
```text
Authorization: Bearer <json web token>
```
- 放在`POST`请求的数据体里面

用户如果使用浏览器请求, 浏览器会自动通过`HTTP`请求头携带`JWT`进行请求.

## JWT的特点和注意事项

- `JWT`默认是不加密的, 但也是可以加密的, 生成`Token`以后, 可以用密钥再加密一次.
`JWT`在不加密的情况下, 不要将私密数据写入`JWT`.

- `JWT`不仅可以用于认证, 也可以用于交换信息. 有效使用`JWT`, 可以降低服务器查询数据库的次数.

- `JWT`的最大缺点是无法在使用过程中废止某个用户的`JWT`, 或者更改`JWT`的权限.
一旦`JWT`签发了, 在到期之前就会始终有效, 除非服务器部署额外的逻辑.

- `JWT`本身包含了认证信息, 一旦泄露任何人都可以获得该令牌的所有权限.
为了减少盗用, `JWT`的有效期应该设置的比较短.
对于一些比较重要的权限,
即使`JWT`没有过期也应该再次对用户进行认证(查询数据库进行用户名密码校验).

- 安全原因, `JWT`不应该使用`HTTP`协议明文传输, 要使用`HTTPS`协议传输.

## JWT适用场景

`JWT`适用于:

- `App`和`Web`的单页面应用, 比如各种前后端分离的`Web`项目.
- 供用户程序调用的`API`项目.

> 注意: <br>
> 由后端渲染`HTML`页面的场景(`MVC`架构), 建议使用`Cookie-session`方式认证.

## 参考

> https://jwt.io/
