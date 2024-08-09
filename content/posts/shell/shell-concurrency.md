---
title: "Shell并发处理任务"
date: 2011-09-28T13:20:11+08:00
authors:
  - windvalley
categories:
  - Shell
tags:
  - shell
  - concurrency
draft: true
---

## 功能说明

- [x] 多进程并发运行程序，始终保持指定进程数运行

## 方法 1

```bash
handle_threads(){
    local threads_num=$1;shift
    local object_list=$1;shift
    local command="$*"

    [ -z "$command" ] && {
        echo "Usage: $FUNCNAME threads_num object_list command";return 1; }

    tmp_fifofile="/tmp/$$.fifo"
    mkfifo $tmp_fifofile
    exec 6<> $tmp_fifofile
    for ((i=0;i<$threads_num;i++));do echo;done >&6

    for object in $object_list;do
      read -u6
      {
          $command $object
          echo >&6
      } &
    done

    wait
    exec 6>&-
    rm -f $tmp_fifofile
}
```

使用方法举例:

```terminal
# 100是指保持并发处理的进程数,
# "obj1 ..."是要处理的对象集合,
# foo.sh是处理对象的程序(可以是函数也可以是脚本)
handle_threads 100 "obj1 obj2 obj3 ..." func
```

## 方法 2

```bash
handle_threads() {
    local threads_num=$1;shift
    local object_list=$1;shift
    local command="$*"
    [ -z "$command" ] && { echo "Usage: $FUNCNAME threads_num object_list command";return 1;}

    for object in $object_list;do
        {
            $command $object
        } &

        while :; do
            sum=$(ps axu|grep -v grep|grep -c ${command%% *})
            [ $sum -ge $threads_num ] && continue || break
        done
        # while语句段也可以用如下until替代
        #sum=$(ps axu|grep -v grep|grep -c ${command%% *})
        #until [ $sum -lt $threads_num ];do
        #    sum=$(ps axu|grep -v grep|grep -c ${command%% *})
        #done
    done
}
```

使用方法举例:

```terminal
handle_threads 100 "obj1 obj2 obj3 ..." func
```
