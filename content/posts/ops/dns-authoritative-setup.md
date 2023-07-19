---
title: "搭建权威DNS的逻辑思路"
date: 2014-10-12T17:38:53+08:00
lastmod: 2014-10-12T17:38:53+08:00
categories:
  - OPS
tags:
  - dns
  - setup
toc:
  enable: false
draft: false
---

如果公司想要搭建自己的`权威DNS集群`用于管理公司所有域名的解析, 需要先注册一个一级域名, 比如说`sre.im`, 然后确定`权威DNS的域名`和对应的`VIP`的关系, 比如:

```text
ctc1.dns.sre.im, 对应vip地址为123.2.2.2
cuc1.dns.sre.im, 对应vip地址为231.3.3.3
等等...
```

每个`vip`对应一组服务器, 通过`ospf协议`做负载均衡, 每台服务器的 DNS 配置完全一致.

服务器上的配置为:

```text
zone sre.im
sre.im 600 IN NS ctc1.dns.sre.im
sre.im 600 IN NS cuc1.dns.sre.im
ctc1.dns.sre.im 600 IN A 123.2.2.2
cuc1.dns.sre.im 600 IN A 231.3.3.3
```

以上`NS记录`可以没有, 但必须有`A记录`. 一般都会在这里加入`NS记录`, 这样当用户查询的时候就可以直接缓存在`递归DNS`本地, `递归DNS`不用每次都去上一级`权威DNS`获取列表, 如果有`权威DNS`横向扩容的需求, 可以直接在这里加`NS`和`A`记录解析, 不用上报给上一级`权威DNS`添加新的`胶水记录`就可以.

然后向`COM权威DNS`申请添加胶水记录:

```text
sre.im 172800 IN NS ctc1.dns.sre.im
sre.im 172800 IN NS cuc1.dns.sre.im
ctc1.dns.sre.im 172800 IN A 123.2.2.2
cuc1.dns.sre.im 172800 IN A 231.3.3.3
```

这样我们自建的`权威DNS`就可以对我们购买的所有域名做解析管理. 比如说我们注册了一个域名`360.cn`, 想让我们`自建权威DNS`管理它的解析, `自建权威DNS`配置为:

```text
zone 360.cn
360.cn IN NS ctc1.dns.sre.im
360.cn IN NS cuc1.dns.sre.im
```

这里可以不加`NS记录`, 加的好处是可以在`递归DNS`本地缓存下来, `递归DNS`不用先去`CN权威DNS`上获取这个列表了, 减少了解析流程, 从而加速解析.

然后向`CN权威DNS`申请变更`360.cn`的`NS`记录, `CN权威DNS`配置:

```text
360.cn IN NS ctc1.dns.sre.im
360.cn IN NS cuc1.dns.sre.im
```

也就是每管理一个域名, 都要配置一个相应的`zone`, 同时配置该域名的`NS记录`. 有一个特殊情况, 如果该域名的`NS记录`是该域名的子域名, 则需要同时配置`NS记录`和`A记录`, 再申请把这对`胶水记录`添加到该域名的`上级权威DNS`.

如果要解析`权威DNS域名`本身的`A记录`, 比如`dig ctc1.dns.sre.im`, 则由`COM权威DNS`返回`sre.im`的所有 NS 服务器的 IP 地址, 然后`递归DNS`根据相关算法选择一个最快的 IP 去查询`ctc1.dns.sre.im`, 最后由我们`自建权威DNS`返回`A记录`结果.
