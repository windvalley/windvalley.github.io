---
title: "聊一聊HTTP访问劫持"
date: 2016-02-22T21:50:25+08:00
authors:
    - windvalley
categories:
    - OPS
tags:
    - problems
keywords:
    - 劫持
    - DNS劫持
    - HTTP劫持
draft: false
---

## 劫持的分类与原理简单介绍

### 1. 路由劫持

在运营商机房的近用户端路由器上接入一个分光器, 将访问流量分光到一台服务器用于数据采集与分析, 这台服务器的作用是分析访问流量特征, 将访问条目的访问频次和大小进行统计, 对于满足条件的TOP N链接会标记为强制缓存, 这样用户再访问这些链接的时候, 将会被这台服务器提前响应(由于这台服务器距离用户更近, 真正的ICP内容服务器的响应将会被用户端抛弃), 给用户一个新的下载链接(302重定向方式), 用户会无感知的访问运营商的缓存服务器(正向代理方式).

### 2.域名劫持

* 在运营商本地DNS上代替授权DNS给出伪造的解析记录.
* 分光方式提前响应DNS解析记录, 由于维护成本低, 方便快捷, 基本上用的这种方式, 也叫DNS投毒.

域名劫持通过以上方式的一种将域名权威解析IP变更为运营商的缓存服务器IP(缓存方式为正向代理)来实现劫持.

## 劫持的原因

* 二级运营商(一般指非电信和联通的其他宽带运营商)和电信联通的互联带宽结算费用严重不对等, 二级运营商通常要为出网带宽向电信或者联通支付巨额的费用(1Gb带宽40万到100万不等), 通过劫持缓存大流量文件, 节省出网带宽, 同时增强本网用户体验(在劫持不出现问题的情况下).
* 不少宽带运营商为了向宽带用户推广大宽带产品, 通常会劫持一些通用的测速软件的测速链接, 以提高测速效果, 给用户大带宽的假象.
* 页面内强制嵌入广告、更换渠道推广文件, 以获取非法利益.
* 工信部的区域宽带提速项目, 比如2015年的工信部西南三省宽带提速项目.

另外, 绝大部分劫持现象出现在二级运营商, 一般运营商不会自己做劫持项目, 而是会外包给第三方公司去做, 比如世纪互联、网宿、华为等等, 做这种项目的公司还是比较多的.

## 针对劫持的有效处理方法

### 1.被动处理模式

* 分析出劫持过程, 判断劫持IP的归属运营商, 如果ICP和劫持运营商有关系往来, 比如租用了运营商的机房、有其他的合作交流等, 可以直接联系运营商接口人说明情况, 提供劫持证据, 一般都能很快的得到解决; 如果ICP没有劫持运营商的联系方式, 可整理劫持证据, 发邮件投诉CERT, 解决时间上可能会稍慢, 但一定会得到解决.
* 很多劫持都是由用户发现并投诉到ICP的, ICP可让用户同步投诉到自己的宽带运营商, 如果没有效果, 可在当地的通管局官网上投诉, 一般都会得到解决.

### 2. 主动处理模式, 建立防劫持机制

* 通过https方式绕过运营商劫持, 对于页面图片类等小文件特别适合, 可以考虑全站部署, 但是对于大文件下载, https会可能降低性能, 可以有选择性的部署.
* 分析路由劫持的特点(对相同URL的访问量达到阈值才会劫持), 通过客户端和服务端的配合, 使用户下载相同文件的文件名每次都唯一, 可有效绕过运营商的劫持策略.鉴权模式.
* 可采用`https_dns`方式代替用户localdns做域名解析, 有效防止域名劫持并可做到更精确的调度.适用于app或PC的端产品, 不适用于浏览器.
