---
title: "如何进行C100K性能测试"
date: 2019-07-11T11:51:34+08:00
lastmod: 2019-07-11T11:51:34+08:00
categories:
    - OPS
tags:
    - performance
    - test
draft: false
---

## 测试前环境准备

- 关闭SElinux: `sestatus | grep -q disabled || setenforce 0`
- 设置最大打开文件数: `sysctl -w fs.file-max=6552016`
- 设置单个进程最大打开文件数: `ulimit -n 32768`
- 使用`c1000k`工具查看当前系统最大支持的并发连接数

```bash
wget --no-check-certificate https://github.com/ideawu/c1000k/archive/master.zip
unzip master.zip
cd c1000k-master
make

./server 7000 &
./client 127.0.0.1 7000
```

## 使用`siege`压测工具

### 安装siege

```bash
wget http://download.joedog.org/siege/siege-latest.tar.gz

cd siege-4.0.4

# --with-ssl选项使siege支持测试https链接
./configure --prefix=/usr/local/siege --with-ssl=/usr/local/openresty/openssl/

make
make install
```

### 配置siege

```bash
cd /usr/local/siege

# 查看配置, 可知默认配置文件为:/usr/local/siege/etc/siegerc
./siege -C

# 修改几处默认配置
vim /usr/local/siege/etc/siegerc
verbose = false   # 不显示响应信息
limit = 5000    # 将siege默认的client线程数限制由255改为5000
```

### 使用siege对接口进行压测

```text
./siege -c500 -r100 -b http://some-api

-c 表示模拟500个用户进行请求

-r 表示进行多少次测试

-b 表示压测(请求和请求之间没有延迟), 如果是`-i`, 则表示模拟用户请求(请求和请求之间有延迟)
```

### 输出

```text
** SIEGE 4.0.4
** Preparing 500 concurrent users for battle.
The server is now under siege...
Transactions:                  50000 hits # 服务端成功处理的请求数
Availability:                 100.00 %    # 成功率
Elapsed time:                   4.80 secs  # 测试总用时
Data transferred:               2.24 MB    # 测试总传输数据量
Response time:                  0.04 secs  # 请求平均响应时间
Transaction rate:           10416.67 trans/sec # 每秒并发处理数
Throughput:                     0.47 MB/sec # 每秒吞吐量
Concurrency:                  400.88  # CPU同一时间点最大处理数
Successful transactions:       50000  # 成功的请求数
Failed transactions:               0  # 失败的请求数
Longest transaction:            1.10  # 最长的一次响应用时
Shortest transaction:           0.00  # 最短的一次响应用时
```

以上测试结果指标, 重点关注: `每秒并发处理数`、`成功率`、`请求平均响应时间`

## 使用wrk压测工具

### 安装

```bash
wget -c https://github.com/wg/wrk/archive/4.1.0.tar.gz -O 4.1.0.tar.gz
tar zxf 4.1.0.tar.gz
cd wrk-4.1.0/
make -j24
```

### 开始压测

`./wrk -t12 -c400 -d10s http://some-api`

### 输出

```text
Running 10s test @ http://some-api
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     3.85ms    7.03ms 206.87ms   96.68%   # 平均延迟3.85ms
    Req/Sec    10.13k     1.15k   15.47k    87.45%
  1220141 requests in 10.10s, 264.08MB read   # 10秒共处理了122万次查询
Requests/sec: 120801.15   # 每秒并发12万
Transfer/sec:     26.15MB
```

## 工具选择

由`siege`和`wrk`工具的对比可知, 在相同测试环境下, `siege`的测试能力远低于`wrk`, 无法通过`siege`进行准确的测出服务并发能力, 使用`wrk`是更好的选择.

## 注意事项

网络延迟对测试结果影响巨大, 所以尽量在同一局域网内测试, 测试客户端服务器到待测试的服务器之间ping尽量在`1ms`以内, 否则压测的结果会和服务器实际承受能力相差几十倍甚至更多.

可以尝试把`wrk`和服务端程序都部署在同一台性能比较好的机器上.
比如, 我们在 Nginx 中开启20个worker, 剩下的4个 CPU 资源分给 `wrk`. 这样一来,
就只有本地的网络通信, 可以把网络对测试结果的影响降到最低.
