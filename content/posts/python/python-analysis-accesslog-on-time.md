---
title: "使用Python实时分析处理Nginx访问日志"
date: 2015-01-16T14:15:07+08:00
lastmod: 2015-01-16T14:15:07+08:00
categories:
  - Python
tags:
  - python
draft: true
---

```python
#!/usr/bin/env python

logfile = '/usr/local/nginx/logs/access.log'
# 打开日志文件
accesslog = open(logfile,'r')

# 定义一个空列表, 用于临时存放日志
loglist = []

# 获取当前时间的小时整点的时间戳, 比如现在是12:23, 则获取12点整的时间戳
now_timelist = list(time.localtime(time.time()))
now_timelist[4] = 0
now_timelist[5] = 0
now_hour_zero = now_timelist
old_timestamp = time.mktime(now_hour_zero)

# 每次处理多长时间的日志
time_interval = 600

# 通过while循环实时读取日志的行
while True:
    # 获取文件位置偏移量, 首次读取之前, 获取的是0
    location_offset = accesslog.tell()
    # 读取一行
    line = accesslog.readline()

    # 如果行为空, 说明到行尾了, 日志文件轮转了,
    # 需要重新加载日志文件, 然后从新开始循环
    if not line:
        # 关闭文件
        accesslog.close()
        # 打开轮转后的新access.log文件
        accesslog = open(logfile,'r')
        continue

    linelist = line.split()
    linelist_check = line.split('"')

    # 验证日志文件是否成功写完这行日志, 避免只写了一行的一部分就获取。
    if len(linelist_check) != 19:
        # 回到指定的偏移位置, 即此行首, 然后再读取一次, 直到行验证通过
        accesslog.seek(location_offset)
        continue

    # 获取当前日志打点的时间的时间戳
    try:
        now_time = linelist[4][1:]
        x = datetime.datetime.strptime(now_time, '%d/%b/%Y:%H:%M:%S')
        now_timestamp = time.mktime(x.timetuple())
    except Exception as e:
        pass

    # 在指定时间范围内的日志都会追加到一个列表里
    if int(now_timestamp-old_timestamp) <= time_interval:
        loglist.append('{0} {1} {2}'.format(linelist[0],linelist[1],linelist[10]))

    else:
        # 此处为对间隔时间内收集的日志列表做自定义分析处理
        log_resolv_threads(loglist)

        # 重置日志列表, 下个循环将开始收集新的间隔时间的日志到列表中
        loglist = []
        old_timestamp = now_timestamp
        # 回到指定的偏移位置, 即当前处理行的行首, 也就是下一次循环重新处理一遍当前行
        accesslog.seek(location_offset)
```
