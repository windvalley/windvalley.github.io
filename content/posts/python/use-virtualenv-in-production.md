---
title: "如何将Virtualenv用于生产环境"
date: 2014-10-14T14:26:16+08:00
lastmod: 2014-10-14T14:26:16+08:00
categories:
  - Python
tags:
  - python
draft: false
---

## 应用场景

一台服务器上运行多个 Python 项目, 每个项目都需要有独立的 Python 虚拟环境, 互不干扰.

## 项目组织

服务器上统一的干净的 Python 路径为:
`/usr/local/python/`

项目 1 的根路径举例:
`/usr/local/apps/project1/`

项目 1 的虚拟环境:

注意虚拟环境和服务器上统一的 Python 环境是有关联的, 不能独立存在.

`/usr/local/apps/projects1/pyenv`

项目 1 的入口程序文件举例:

入口程序文件必须在项目根路径下.

文件名: `/usr/local/apps/project1/main.py`

文件内容:

```python
#!/usr/bin/env python
import lib.xxx
```

项目 1 的其他程序目录或文件举例:

文件名:
`/usr/local/apps/project1/lib/xxx.py`

## 运行项目

### 方法 1: 通过 shell 脚本包装环境

文件名: `/usr/local/apps/project1/main_start.sh`

文件内容:

```bash
#!/bin/bash

script_dir=$(dirname $(readlink -f ${BASH_SOURCE[0]}))
cd $script_dir
source pyenv/bin/activate

./main.py

exit 0
```

### 方法 2: 直接运行 python 项目的入口文件

#### 运行方式 1

`/usr/local/apps/project1/pyenv/bin/python /usr/local/apps/project1/main.py`

#### 运行方式 2

将`/usr/local/apps/project1/main.py`文件的第一行修改为:
`#!/usr/local/apps/project1/pyenv/bin/python`

然后这样运行: `/usr/local/apps/projects/main.py`

#### 运行方式 3

将`/usr/local/apps/projects/main.py`文件的第一行修改为:
`#!./pyenv/bin/python`

然后这样运行: `cd /usr/local/apps/projects && ./main.py`
