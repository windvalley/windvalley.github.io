---
title: "RESTful API规范"
date: 2018-12-30T13:49:46+08:00
lastmod: 2018-12-30T13:49:46+08:00
categories:
    - Web
tags:
    - web
    - api
    - restful
keywords:
    -
draft: false
---

## 什么是Web API

简单的说就是如果一个URL返回的不是HTML, 而是某种格式化的数据字符串, 比如`JSON`、`XML`等, 那么这个URL就可以说是一个`Web API`.

`Web API`就是把Web应用的功能封装起来, 通过操作API就可以使用该Web应用提供的数据和功能.

## 什么是REST

`REST`(`Representational State Transfer`)是一种软件架构模式的风格,
也可以说是一种设计`Web API`的架构模式.

`REST`架构的主要原则:

- 所有事物都被抽象为资源
- 每个资源都有唯一的资源标识符
- 同一个资源可以有多种表现形式, 比如json、xml等
- 对资源的所有操作都不会改变资源标识符
- 对资源的所有操作都是无状态的

符合`REST`原则的架构方式就可以称为`RESTful`.

## RESTful API规范

### URL设计规范

1. URL表示资源, 应该使用名词, 且是复数形式, 比如:

```text
/users
/articles
```

而不要写成:

```text
/create-users  # 成为动词了
/article  # 单数形式了
```

2. 多个单词组成的资源名称, 应该使用`-`连接, 比如:

```text
/china-provinces
```

而不要写成:

```text
/chinaProvinces  # 不要使用大写字母
/china_provinces  # 不要使用下划线等其他特殊字符
```

3. 避免通过多级URL嵌套资源

    到表示资源的这级路径后, 只允许继续扩展1级路径表示具体的某个资源,
不允许继续扩展路径表示其他含义, 应使用参数代替. 比如:

```text
# authors表示资源路径用户, 继续扩展1级2表示具体的某个用户,
# 再扩展2级路径articles/3表示某个具体用户的某篇具体文章.
# 从articles开始的路径都是不对的.
/authors/2/articles/3
```

应该改为:

```text
/authors/2?aurticles=3
```

也就是说除资源路径开始最多只能扩展一级路径, 表示具体的某个资源,
其他功能都使用参数来表示.

4. URL结尾不要有`/`

    我们设计的`URL`结尾不要加`/`.

    如果用户请求的url结尾有`/`, 使用重定向的方式去除`/`, 大部分Web框架会自动做这个处理.

    正确的写法:

```text
/authors
```

不要设计成:

```text
/authors/
```

5. 不要使用文件扩展名

比如应该设计成这样:

```text
/users
```

不要设计成这样:

```text
/users.xml
/users.json
```

6. 使用版本号

    比如:

```text
/v1/xxx-project/users
```

在URL路径开始部分加上版本号.

### 请求方法规范

用来操作URL的方法(`Method`)有如下:

- `GET`: 获取资源列表, 或单个资源
- `POST`: 创建资源
- `PUT`: 更新资源
- `PATCH`: 更新资源的某个或某些属性(字段)
- `DELETE`: 删除资源
- `OPTIONS`: 获取服务端允许对资源操作的方法有哪些.

`POST/PUT/PATCH`请求参数建议使用`JSON`格式将参数放到请求体中.

## 参考

> https://restfulapi.net/resource-naming/ <br>
> https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design <br>
> https://demo.doc.coding.io/ <br>
> https://help.coding.net/docs/management/api/import/openapi.html
