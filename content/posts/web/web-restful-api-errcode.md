---
title: "Restful API 错误码设计参考"
date: 2017-12-18T09:40:19+08:00
lastmod: 2017-12-18T09:40:19+08:00
categories:
  - Web
tags:
  - web
  - restful
  - api
keywords:
    -
draft: false
---

## 微博

新浪微博开放平台错误码设计较为传统, 以数字组合代表一定的错误含义, 不太易读.

详见: [微博开放平台错误码说明](https://open.weibo.com/wiki/Error_code)

## 360

360点睛营销开放平台的错误码也是类似微博开放平台, 以数字作为错误码, 不过感觉没有章法可循, 比较乱.

详见: [点睛开放平台错误码总揽](http://open.e.360.cn/api/errorCode.html)

## 阿里云

阿里云的API错误码是以英文简写代替数字, 个人感觉更加易读和可维护, 推荐.

详见: [阿里云API错误中心](https://error-center.aliyun.com/)

## 腾讯云

腾讯云的API错误码和阿里云类似使用英文简写表示, 不过划分为两类: 公共错误码和业务错误码.

详见: [腾讯云云服务器API错误码](https://cloud.tencent.com/document/api/213/30435)

## 百度云

百度云的每个项目的错误码设计规范都不太一样, 有的项目用数字表示错误码, 有的项目用英文简写,
可见这是不同团队之间做的不同选择.

详见:

[图像搜索错误码](https://cloud.baidu.com/doc/IMAGESEARCH/s/sk2k1sor0)(数字方式)

[MapReduce BMR错误码](https://cloud.baidu.com/doc/BMR/s/Wjwvxvxn7)(英文简写方式)

## 总结

每个公司的每个项目的具体错误码定制都是不一样的, 没有统一的规范,
我们可以博采众长, 根据项目实际情况, 设计符合自己项目要求的错误码规范.
