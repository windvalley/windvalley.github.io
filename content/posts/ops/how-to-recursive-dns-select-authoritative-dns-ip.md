---
title: "递归DNS是如何选择权威DNS的IP去解析域名的"
date: 2015-05-13T21:11:03+08:00
lastmod: 2015-05-13T21:11:03+08:00
categories:
    - OPS
tags:
    - dns
    - how
    - ops
draft: false
---

我们知道权威DNS为了加速解析、抗攻击、高可用等原因, 一般都是一个分布式集群,
部署了很多个节点, 每个节点同样也是一个集群, 对外暴露一个`VIP`.
所以`权威DNS`会对应多个`VIP`,
那么`递归DNS`解析域名的时候是怎样选择使用哪个`权威DNS IP`来加速解析的呢?

下面我们以解析`360.webcdn.qhcdn.com`域名为例进行简单的讲解.

## 先看递归dns解析`360.webcdn.qhcdn.com`的过程

先清除递归dns本地缓存: `rndc flush`

抓包分析:

```text
172.24.251.155.20489 > 192.33.4.12.53: A? 360.webcdn.qhcdn.com.  # 查询根
192.33.4.12.53 > 172.24.251.155.20489:  q: A? 360.webcdn.qhcdn.com. # 根返回com权威NS和A记录列表
com. [2d] NS j.gtld-servers.net.,
com. [2d] NS c.gtld-servers.net.,
com. [2d] NS a.gtld-servers.net.,
com. [2d] NS e.gtld-servers.net.,
com. [2d] NS b.gtld-servers.net.,
com. [2d] NS i.gtld-servers.net.,
com. [2d] NS m.gtld-servers.net.,
com. [2d] NS g.gtld-servers.net.,
com. [2d] NS k.gtld-servers.net.,
com. [2d] NS d.gtld-servers.net.,
com. [2d] NS f.gtld-servers.net.,
com. [2d] NS l.gtld-servers.net.,
com. [2d] NS h.gtld-servers.net.,
a.gtld-servers.net. [2d] A 192.5.6.30,
b.gtld-servers.net. [2d] A 192.33.14.30,
c.gtld-servers.net. [2d] A 192.26.92.30,
d.gtld-servers.net. [2d] A 192.31.80.30,
e.gtld-servers.net. [2d] A 192.12.94.30,
f.gtld-servers.net. [2d] A 192.35.51.30,
g.gtld-servers.net. [2d] A 192.42.93.30,
h.gtld-servers.net. [2d] A 192.54.112.30,
i.gtld-servers.net. [2d] A 192.43.172.30,
j.gtld-servers.net. [2d] A 192.48.79.30,
k.gtld-servers.net. [2d] A 192.52.178.30,
l.gtld-servers.net. [2d] A 192.41.162.30,
m.gtld-servers.net. [2d] A 192.55.83.30,
a.gtld-servers.net. [2d] AAAA 2001:503:a83e::2:30,
b.gtld-servers.net. [2d] AAAA 2001:503:231d::2:30,
c.gtld-servers.net. [2d] AAAA 2001:503:83eb::30,
d.gtld-servers.net. [2d] AAAA 2001:500:856e::30,
e.gtld-servers.net. [2d] AAAA 2001:502:1ca1::30,
f.gtld-servers.net. [2d] AAAA 2001:503:d414::30,
g.gtld-servers.net. [2d] AAAA 2001:503:eea3::30,
h.gtld-servers.net. [2d] AAAA 2001:502:8cc::30,
i.gtld-servers.net. [2d] AAAA 2001:503:39c1::30,
j.gtld-servers.net. [2d] AAAA 2001:502:7094::30,
k.gtld-servers.net. [2d] AAAA 2001:503:d2d::30,
l.gtld-servers.net. [2d] AAAA 2001:500:d937::30,
m.gtld-servers.net. [2d] AAAA 2001:501:b1f9::30
172.24.251.155.47449 > 192.31.80.30.53: A? 360.webcdn.qhcdn.com.  # 随机向COM的一个权威IP发起查询
192.31.80.30.53 > 172.24.251.155.47449: q: A? 360.webcdn.qhcdn.com.  # COM返回qhcdn.com权威NS和A记录
qhcdn.com. [2d] NS ns1.qhcdn.com.,
qhcdn.com. [2d] NS ns2.qhcdn.com.,
qhcdn.com. [2d] NS ns3.qhcdn.com.,
qhcdn.com. [2d] NS ns4.qhcdn.com.,
qhcdn.com. [2d] NS ns5.qhcdn.com.,
ns1.qhcdn.com. [2d] A 123.125.82.30,
ns2.qhcdn.com. [2d] A 180.153.232.30,
ns3.qhcdn.com. [2d] A 54.229.66.110,
ns4.qhcdn.com. [2d] A 42.236.9.239,
ns5.qhcdn.com. [2d] A 36.110.213.30
172.24.251.155.5435 > 54.229.66.110.53: A? 360.webcdn.qhcdn.com. # 随机向qhcdn.com的一个权威IP发起查询
54.229.66.110.53 > 172.24.251.155.5435: q: A? 360.webcdn.qhcdn.com. # 最终返回结果
360.webcdn.qhcdn.com. [1m] A 104.192.110.245
```

## 分析递归DNS怎么在NS列表中选择去哪一个NS发起查询

以下是递归DNS查询`360.webcdn.qhcdn.com`的结果:

```text
360.webcdn.qhcdn.com. [1m] A 104.192.110.245
qhcdn.com. [2d] NS ns1.qhcdn.com.,
qhcdn.com. [2d] NS ns2.qhcdn.com.,
qhcdn.com. [2d] NS ns3.qhcdn.com.,
qhcdn.com. [2d] NS ns4.qhcdn.com.,
qhcdn.com. [2d] NS ns5.qhcdn.com.,
ns1.qhcdn.com. [2d] A 123.125.82.30,
ns2.qhcdn.com. [2d] A 180.153.232.30,
ns3.qhcdn.com. [2d] A 54.229.66.110,
ns4.qhcdn.com. [2d] A 42.236.9.239,
ns5.qhcdn.com. [2d] A 36.110.213.30
```

简单说明:

```text
123.125.82.30  0.01744  # 这个是1中递归DNS首次查询使用的权威IP
54.229.66.110  0.338466 # A记录过期后, 进行第2次查询
180.153.232.30 0.025254 # A记录过期后, 进行第3次查询
42.236.9.239 0.030101  # A记录过期后, 进行第4次查询
36.110.213.30 0.038917 # A记录过期后, 进行第5次查询
```

由上可见, 递归DNS会在之后的查询中把其他NS都随机查询一遍, 同时记录下往返时间.

以下是A记录过期后的第6次到第N次查询用到的NS和往返时间：

```text
123.125.82.30 0.018925
180.153.232.30 0.025916
42.236.9.239 0.025255
123.125.82.30 0.025157
36.110.213.30 0.029854
180.153.232.30 0.023313
42.236.9.239 0.032326
123.125.82.30 0.018438
123.125.82.30 0.016495
180.153.232.30 0.024991
123.125.82.30 0.018657
36.110.213.30 0.042611
123.125.82.30 0.021091
42.236.9.239 0.027226
180.153.232.30 0.02127
123.125.82.30 0.022811
180.153.232.30 0.02783
123.125.82.30 0.021247
42.236.9.239 0.027053
123.125.82.30 0.020548
123.125.82.30 0.016898
123.125.82.30 0.020782
180.153.232.30 0.02466
36.110.213.30 0.033314
123.125.82.30 0.023268
42.236.9.239 0.025731
180.153.232.30 0.018577
123.125.82.30 0.015048
123.125.82.30 0.017281
123.125.82.30 0.01648
123.125.82.30 0.015888
123.125.82.30 0.013337
123.125.82.30 0.020723
123.125.82.30 0.024408
180.153.232.30 0.021204
42.236.9.239 0.028445
42.236.9.239 0.026762
123.125.82.30 0.020545
180.153.232.30 0.023225
36.110.213.30 0.038586
54.229.66.110 0.322662
123.125.82.30 0.018463
180.153.232.30 0.027169
42.236.9.239 0.02367
123.125.82.30 0.021552
36.110.213.30 0.043138
```

对以上进行分析:

`awk '{a[$1]+=$2;b[$1]++}END{for(i in a)print i,a[i]/b[i],b[i]}' ns.txt|sort -k3nr`

输出:

```text
123.125.82.30     0.0193688     23
180.153.232.30    0.0239463     11
42.236.9.239      0.0273966     9
36.110.213.30     0.0377367     6
54.229.66.110     0.330564      2
```

通过以上分析可知, 如果有5个NS, 那么前5次查询, 肯定都会覆盖到, 然后递归DNS在以后的每次查询则会根据某种算法, 往返时间最短的NS访问次数会最多, 其他NS也都会随机访问到, 并且访问次数和往返时间成反比, 即往返时间越大访问次数越少.

> 参考资料: <br>
> http://docstore.mik.ua/orelly/networking_2ndEd/dns/ch02_06.htm <br>
> http://www.sigcomm.org/sites/default/files/ccr/papers/2012/April/2185376-2185387.pdf
