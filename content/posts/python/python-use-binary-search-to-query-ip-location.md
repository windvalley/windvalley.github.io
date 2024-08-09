---
title: "Python使用二分查找算法高效解析IP归属"
date: 2016-06-14T13:54:59+08:00
lastmod: 2016-06-14T13:54:59+08:00
categories:
    - Python
tags:
    - python
    - algorithm
draft: true
---

## ip库文件格式

`ipdb.txt`

```text
0 16777215 保留地址 未知
16777216 16777471 CLOUDFLARE.COM 未知
16777472 16778239 福建 电信
16778240 16779263 维多利亚州 gtelecom.com.au
16779264 16781311 广东 电信
16781312 16785407 日本 i2ts.com
16785408 16793599 广东 电信
16793600 16809983 未知 未知
16809984 16842751 未知 未知
16842752 16843007 福建 电信
16843008 16843263 CLOUDFLARE.COM 未知
16843264 16844799 福建 电信
16844800 16845055 广东 电信
16845056 16859135 广东 电信
16859136 16875519 东京都 i2ts.com
16875520 16908287 未知 未知
16908288 16908799 福建 电信
16908800 16909055 北京 联通
16909056 16909311 APNIC.NET 未知
16909312 16909567 SDNS.CN sdns.cn
```

## 二分查找算法Python版本

### 加载ip库文件到内存, 数据格式list:

```python
import IPy

with open(workdir + '/ipdb.txt', "r") as f:
    ipdb = f.readlines()

```

### 方法1

比较IP区间:

```python
# 方法1: 比较IP区间
def query_ip(ipdb_list, ipdb_length, ipaddr):
    query_ip = IPy.IP(ipaddr).int()
    low = 0
    height = ipdb_length - 1
    while low <= height:
        mid = low + ((height - low) >> 1)
        line_list = ipdb_list[mid].split()
        ip_start, ip_end = int(line_list[0]), int(line_list[1])
        if query_ip < ip_start:
            height = mid - 1
        elif query_ip > ip_end:
            low = mid + 1
        else:
            return line_list[2]
    return 'Not-Found'
```

### 方法2

将起始IP地址, 按照对应的整型值的大小关系, 从小到大进行排序, 然后查找最后一个小于等于给定IP整型值的起始IP地址.

```python
def query_ip(ipdb_list, ipdb_length, ipaddr):
    queryip = IPy.IP(ipaddr).int()
    low = 0
    height = ipdb_length - 1
    while low <= height:
        mid = low + ((height-low) >> 1)
        if int(ipdb[mid].split()[0]) > queryip:
            height = mid - 1
        else:
            if mid == n - 1 or int(ipdb[mid + 1].split()[0]) > queryip:
                return ipdb[mid].split()[2]
            else:
                low = mid + 1
    return 'Not-Found'
```
