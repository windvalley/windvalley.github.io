---
draft: false
title: "Shell编码规范"
date: 2010-11-27T22:14:44+08:00
authors:
    - windvalley
categories:
    - Shell
tags:
    - shell
    - conventions
---

### 文件名字

- 要以`.sh`为扩展名
- 脚本名尽量表达脚本的用途
- 脚本要给执行权限, 并且推荐以`./foo.sh`方式执行

```terminal
foo.sh
chmod u+x foo.sh
./foo.sh
```

### 脚本头部

首行必须指定脚本解释器, 为了通用性更好, 建议使用`#!/usr/bin/env bash`, 而不是`#!/bin/bash`;

第二行添加一行空注释;

接下来是脚本文件名、功能描述、作者及联系方式、最后修改日期、版本号等.

```bash
#!/usr/bin/env bash
#
# Script Name: mysqldump.sh
# Description: Mysql of InnoDB or MyIsam Engines Backups by mysqldump.
#
# Author: xxx<xxx@gmail.com>
# Last Modified: 2012/01/12
# Version: 0.1
```

如果是简单脚本, 简单一点也可以:

```bash
#!/usr/bin/env bash
# mysqldump.sh
# author_name
# 2010/01/12
```

### 使用`set`相关命令

脚本头部之后空一行配置`set`相关命令:

- `set -o nounset`, 表示当引用未定义的变量时自动退出脚本
- `set -o errexit`, 表示当遇到命令运行失败时自动退出脚本, 注意由管道组成的整条命令,
   如果管道前命令失败, 管道后命令执行成功, 还是会认为整条命令是成功的
- `set -o errtrace`, 当函数里或者子shell里发生错误时自动退出脚本
- `set -o pipefail`, 捕获管道前的命令出现错误的情况, 配合`set -o errexit`使用
- `set -o xtrace`, 调试脚本使用, 默认要注释掉

以上非必须, 根据实际情况看是否需要使用。

```bash
#!/usr/bin/env bash
# foo.sh
# levin
# 2010/07/26

set -o nounset
set -o errexit
set -o errtrace
set -o pipefail
#set -o xtrace
```

### 变量的定义与引用

- 一般脚本头部之后空一行开始变量的定义
- 全局变量和常量使用大写字母加下划线的方式，
  局部变量使用小写字母加下划线的方式,
  变量名称尽量要表达其含义.
- 对于只读变量, 使用`readonly`修饰, 防止被重新赋值
- 函数内变量使用小写字母加下划线方式，且使用`local`修饰
- 要使变量在子进程中也生效, 需要使用`export`修饰
- 变量的引用方式最好加大括号, 比如`${var}`, 防止和其他字符串混在一起引发错误
- 引用变量时最好加上双引号`""`, 这样当变量值不小心是空时也能代表为空的字符串,
  避免引发未知错误.

示例：

```bash
#!/usr/bin/env bash
# mysqldump.sh
# levin
# 2012/01/12

# 大写表示全局变量
GLOBAL_VAR1=100
VAR_ZIP="foo.zip"
# 表示只读的全局变量
readonly FOO1="not change"
# 子shell可调用的全局变量
export FOO2="can be used in subshell"

[[ "$GLOBAL_VAR1" -lt 100 ]] && mv "$VAR_ZIP" ${var2}_20120102


while read line;do
    # 小写表示局部变量
    var1=$(echo $line|awk...)
done < file.txt

func1() {
    # 函数内部的局部变量
    local var1=xxx
    local var2=123
}

```

### 注释

- 说明性注释`#`和注释文字之间要有一个空格, 代码注释`#`和代码之间不要有空格
- 大段代码注释不要使用`#`, 使用`Here Document`用法

```bash
<<EOF
注释内容
EOF
```

- 尽量不使用中文注释, 用来避免在部分环境中变成乱码(不强制)

说明性注释:
`# this is a apple pin.`

代码行注释:

```bash
#var1=100
#var2=200
```

大段代码注释:

```bash
<<CODE 或者 <<'CODE'
while sleep 3;do
    echo "hello world"
done
CODE
```

### 每行长度

代码每行长度尽量不超过`80`个字符, 过长的情况要使用`\`分行.

### 函数

- 多使用函数, 使代码模块化, 除可重用外还增加可读性
- 函数使用`foo(){}`这种简洁的方式定义, 不使用`function`关键字方式,
  这种方式对老版bash兼容性不好
- 函数中的变量只在函数内起作用时, 记得加`local`来指定为局部变量,
  避免和函数外部变量混淆

```bash
foo(){
    local var=100
}
```

### 记录日志

关键操作要记录日志, 可记录执行时间点、什么操作、输出什么内容、成功还是失败等

```log
2010-07-20 16:40:57 [receive api data]: false 5499,192.168.0.1
2010-07-20 16:40:58 [excute result]: 5499,success
2010-07-20 16:42:58 [receive api data]: false 5500,10.128.0.1
2010-07-20 16:42:59 [excute result]: 5500,success
```

### 条件语句

- 简单的条件语句最好不要使用if语句, 使用 && ||方式代替, 尽量简洁.

```bash
[[ "$var" -lt 10 ]] && echo ok || echo error
```

- 字符串或者数字进行比较时, 统一使用`[[ ]]`,不使用过时的`[]`、`test`方式；字符串相等的比较使用`=`即可, 不用使用`==`。

```bash
var1=100
var2=200
foo1="hello"
foo2="world"

[[ "$var1" -eq "$var2" ]] && echo ok || echo err
[[ "$var1" -ne "$var2" ]] && echo ok || echo err
[[ "$foo1" = "$foo2" ]] && echo ok || echo err
[[ "$foo1" != "$foo2" ]] && echo ok || echo err
```

### 内容输出

输出大段文本内容或生成模版文件等情况, 避免使用`echo`, 尽量使用`here document`.

```bash
# 输出大段内容, 如果EOF使用了单引号扩起来, 则表示忽略变量引用, 完全原样输出。
cat <<EOF 或 <<'EOF'
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
EOF

# 生成模版文件
cat >/etc/my.cnf <<EOF
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
EOF
```

### 脚本路径问题

脚本中涉及到文件路径时, 最好以当前脚本的路径为基准, 去找其他路径,
这样也能避免以绝对路径方式执行脚本时出现一些不必要的错误.
可在脚本开始的位置定义脚本所在路径变量.

```bash
# 两种获取脚本路径的方法
ROOT_DIR=$(cd $(dirname ${BASH_SOURCE[0]}) && pwd)
ROOT_DIR=$(dirname $(readlink -f ${BASH_SOURCE[0]}))  # 不兼容macOS
```

### 进程锁文件

为了避免shell脚本被无意中重复执行造成错乱，对执行中的脚本加锁文件.

示例：

```bash
SHELL_NAME=$(basename $0)
LOCK_FILE="/var/run/${SHELL_NAME}.pid"

# 对于ctrl+c终止脚本的情况, 执行删除锁文件操作
trap "unlock;echo" 2 15

# 创建锁文件
lock(){
    echo $$ >$LOCK_FILE
}

# 删除锁文件
unlock(){
    rm -f $LOCK_FILE
}

# 发现锁文件, 则退出执行, 避免重复执行.
if_lock(){
    if [ -f "$LOCK_FILE" ];then
        pid=$(cat $LOCK_FILE)
        echo "Existing lock $LOCK_FILE: another copy is running as pid $pid"
        exit 1
    fi
}
```

### 其他

- 成对符号要一次性书写完成再书写内容
- 流程控制语句一次性书写完成再书写内容
- 通过缩进让代码更易读, 代码缩进最好是4个空格, 不要使用tab
- 在语句末尾和空行不要存在空格
- 为了便于程序阅读, 脚本里推荐使用长参数代替短参数的情况,
  可避免再通过`man page`查阅
- 代码尽量简洁, 能用一个命令解决的问题不要使用多条命令堆砌
- 提前考虑到可能出现的异常情况, 提前做好异常处理
- 考虑到性能问题, 读取大文件时不要使用`for`循环, 使用`while`循环替代
- 不要把密码硬编码到脚本里, 可通过配置文件或接口的方式引用
- 将命令的执行结果赋值给变量的情况, 尽量使用`$()`, 而不是使用反引号\`,
  因为反引号无法进行嵌套使用并且容易和`'`混淆.

持续更新...
