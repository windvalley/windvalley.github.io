---
title: "递归DNS是如何解析域名的"
date: 2015-05-11T22:58:11+08:00
lastmod: 2015-05-11T22:58:11+08:00
categories:
    - OPS
tags:
    - dns
    - how
    - ops
draft: true
---

我们拿解析`www.360.cn`域名为例说明.

#### 1. 递归DNS从它本地配置的13台根权威DNS的一个`IP`发起查询请求`www.360.cn`.

#### 2. `根权威DNS`返回`CN权威DNS`的域名和IP地址, 并把它们缓存下来.

#### 3. `递归DNS`根据某种算法向`CN权威DNS`的某一个IP地址请求`www.360.cn`.

#### 4. `CN权威DNS`返回`360.cn权威DNS`的域名, 有如下6个域名:
```
dns2.360safe.com.
dns1.360safe.com.
dns7.360safe.com.
dns8.360safe.com.
dns9.360safe.com.
dns3.360safe.com.
```

但没有对应的IP地址, 因为这些域名是`com`域的, `CN权威DNS`没法对其他顶级域的域名配置`A记录`解析, 只能配置`NS记录`解析. `CN权威DNS`配置的相关记录举例:

```
zone 360.cn
360.cn NS dns1.360safe.com
360.cn NS dns2.360safe.com
```

这个配置一般是要求域名托管商帮忙提交的(一般都有自助web页面), 域名托管商有对`顶级域权威DNS`的部分写入权限.

然后`递归DNS`将这些`NS记录`缓存下来.

#### 5. `递归DNS`按照`dns1/2/3/7/8/9`的顺序向`根IP`查询这些`NS域名`的`A记录`, 比如选择了查询`dns2.360safe.com`, 过程如下:

根权威IP返回com的权威NS和A记录, 递归DNS随机向com的一个权威IP发起查询`dns2.360safe.com`, com权威IP返回`360safe.com`权威NS和A记录(此处成对出现的NS记录和A记录就是所谓的胶水记录, 因为`360safe.com`的NS域名是`360safe.com`自身的子域名, 所以还需要配置这些NS域名的A记录, 否则会发生查询死循环. `com权威DNS`上配置的相关胶水记录举例:

```
zone 360safe.com
360safe.com NS dns2.360safe.com
dns2.360safe.com A 36.110.213.6
```

递归DNS随机向`360safe.com`的一个权威IP发送查询`dns2.360safe.com`的A记录.

在递归DNS随机向`360safe.com`的一个权威IP发送完查询`dns1/2/3/7/8/9`的UDP包后, 在没有全部得到响应的情况下, 同时又随机向`360safe.com`的一个权威IP发送查询`www.360.cn`的UDP包.

`360safe.com`权威IP几乎在同时收到了`dns1/2/3/7/8/9.360safe.com`的A记录结果和`www.360.cn`的A记录结果.

