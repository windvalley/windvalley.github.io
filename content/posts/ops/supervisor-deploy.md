---
title: "使用Supervisor管理后台常驻进程"
date: 2016-03-06T13:50:31+08:00
lastmod: 2016-03-06T13:50:31+08:00
categories:
    - OPS
tags:
    - ops
keywords:
    -
draft: false
---

## 功能介绍

- `supervisor`可以把我们要管理的应用程序(进程)转成daemon程序(常驻后台运行)

- 可以方便的通过`supervisorctl`命令对进程进行开启、关闭、重启等操作

- 管理的进程一旦崩溃会自动重启，保证程序在各种原因下宕机后的自我修复

- 适用平台: 类Unix系统, 无法在Windows系统上适用

## 安装

```bash
pip3 install supervisor
```

通过pip安装完成后, 会在python3的安装路径的bin目录下生成如下几个命令:

- echo_supervisord_conf 用于生成superviord服务的配置文件样例.

```bash
echo_supervisord_conf > /etc/supervisord.conf
```

- supervisord 用于启动supervisord服务.

```bash
supervisord  # 若不使用-c参数指定配置文件, 将自动加载/etc/supervisord.conf
```

- supervisorctl 用于管理需要管理的进程, 比如启动、关闭、重启、加载配置等等.

```bash
# 交互模式
supervisorctl
supervisor> help
supervisor> status
supervisor> help tail

# 命令行模式
supervisorctl stop appname
supervisorctl restart appname # 重启某个进程, 注意不会重新加载/etc/supervisord.conf配置文件
supervisorctl reload # 重新加载/etc/supervisord.conf配置文件
```

我们可以将Python3的`bin`目录放到环境变量`PATH`里, 这样我们就可以直接使用`bin`下的各种命令了.

## 配置

`cat /etc/supervisord.conf`

```ini
[unix_http_server]
file = /var/run/supervisord.sock
chmod = 0777
chown= root:root
;username = user
;password = 123

;[inet_http_server]
; Web管理界面设定
;port = 127.0.0.1:9001
;username = user
;password = 123

[supervisorctl]
; 必须和'unix_http_server'里面的设定匹配
serverurl = unix:///var/run/supervisord.sock
;username = chris
;password = 123
;prompt = mysupervisor

[supervisord]
; supervisord本身的运行日志记录位置, default $CWD/supervisord.log
logfile=/var/log/supervisord/supervisord.log
; 单个日志文件大小上限, default 50MB
logfile_maxbytes=50MB
; 日志保留多少个备份, default 10)
logfile_backups=10
; 日志级别, default info, 可选: critical, error, warn, info, debug, trace, or blather
loglevel=info
; pidfile, default $CWD/supervisord.pid
pidfile=/var/run/supervisord.pid
; 如果值为true则supervisord在前台运行, 而不是daemonizing, default false
nodaemon=false
; 要使supervisord能启动成功需要拥有的最少文件描述符资源, default 1024
minfds=10240
; 要使supervisord能启动成功需要的最少进程描述符资源, default 200
minprocs=200
; default is current user, required if root
user=root
childlogdir=/var/log/supervisord/

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

; 管理的某个应用进程的配置，可以添加多个program
[program:idcbwbill]
; 最好是写绝对路径的命令
command=/home/ops/idcbwbill_web/idcbwbill
; 先切换到目标目录, 再执行上面的命令, 有些应用只有这样才不会出错.
directory=/home/q/system/idcbwbill_web/
; superviord启动时自动启动该应用程序
autostart = true
; 如果该应用进程意外宕机, 将自动重启, 值为true将无条件重启, default unexpected
autorestart=unexpected
; 进程正常退出的返回码列表(逗号分隔), 配合autorestart使用, default 0
exitcodes=0
; 如果该应用启动失败或宕机, 间隔几秒进行重启, defalut 1
startsecs = 3
; 启动该应用失败重试的次数, defalut 3
startretries = 3
; 启动该应用进程的系统用户
user = root
; 如果设置为true, 错误信息也将写到stdout_logfile指定的日志文件中, default false
redirect_stderr = true
; 该应用程序产生的日志存放位置
stdout_logfile = /var/log/supervisord/idcbwbill.log
```

## 启动

```bash
/usr/local/python3/bin/supervisord
```

看应用程序是否已启动:

```bash
ps aux | grep supervisor
ps axu | grep idcbwbill
```

查看日志:

```bash
tail -f /var/log/supervisord/supervisord.log
tail -f /var/log/supervisord/idcbwbill.log
```

应用进程的管理, 请参看前面supervisorctl用法.

supervisord开机自启动:

```bash
echo /usr/local/python3/bin/supervisord >>/etc/rc.local
```

## 官方文档

> http://supervisord.org/index.html
