---
title: "关于import的正确用法"
date: 2017-09-27T22:24:28+08:00
lastmod: 2017-09-27T22:24:28+08:00
categories:
    - python
tags:
    - python
    - import
draft: false
---

写一个中大型项目, 往往需要在项目根路径下安排多层级的子目录和文件, 然后各文件之间通过`import`来相互连接.

各python文件之间如何通过`import`连接?

我们要遵从一个规范:

所有python文件都要以项目根目录为起始点来`import`, 当然除了目录下的`__init__.py`文件, 该文件一般是通过`from . import module`来使本目录和本目录下的`module`文件做关联.

`import`其他相关规范也需要注意:

1. 所有`import`尽量放在代码文件的头部位置
2. 每行`import`只导入一个对象
3. 当我们使用`from xxx import xxx`时, `import`后面跟着的对象要是一个`package`(包对应代码目录)或者`module`(模块对应代码文件), 不要是一个类或者一个函数
4. 标准模块、第三方模块、自定义模块要按这个先后顺序排列, 三类之间使用一个空行分隔.

比如项目根路径为:

`/path/project`

项目的组织结构:

```
├── main.py
├── module.py
├── package1
│   ├── __init__.py
│   ├── module.py
│   └── sub_package
│       ├── __init__.py
│       └── module.py
└── package2
    ├── __init__.py
    └── module.py
```

项目文件内容:

`main.py`

```python
#!/usr/bin/env python3

import module
import package1
import package2

module.who()
package1.module.who()
package2.module.who()
package1.sub_package.module.who()
package1.module.call_projectroot_module()
package1.module.call_package2_module()
```

`module.py`

```python
def who():
    print('I am projectroot.module.who()')
```

`package1/module.py`

```python
import module
import package2

def who():
    print('I am projectroot.package1.module.who()')

def call_projectroot_module():
    module.who()

def call_package2_module():
    package2.module.who()
```

`package1/sub_package/module.py`

```python
def who():
    print('I am projectroot.package1.sub_package.module.who()')
```

`package1/sub_package/__init__.py`

```python
from . import module
```

`package1/__init__.py`

```python
from . import module
from . import sub_package
```

`package2/module.py`

```python
def who():
    print('I am projectroot.package2.module.who().')
```

`package2/__init__.py`

```python
from . import module
```

运行项目入口文件

```terminal
python3 main.py
```

输出结果为:

```vim
I am projectroot.module.who()
I am projectroot.package1.module.who()
I am projectroot.package2.module.who().
I am projectroot.package1.sub_package.module.who()
I am projectroot.module.who()
I am projectroot.package2.module.who().
```

以上正确运行的约束条件为:

* 运行项目的入口程序文件, 必须在项目的根路径下;
* `import`的对象必须从项目根路径为起始点;
* 如果`import`的对象为目录(`package`), 则在目录下必须有`__init__.py`文件, 且该文件中通过`from . import module`的形式引入该目录下相关模块, 这样才可以使用相关模块, 否则虽然`import package`时不会报错, 但使用`package`下面的模块时就会提示找不到相关模块;
* 如果`import`的对象为`目录.文件`(package.module)这种形式, 目录下可以没有`__init__.py`文件;
* 如果以`from 目录 import 文件`这种形式, 目录下也可以没有`__init__.py`文件;

值得解释的一个问题:

上面的python3环境并没有将项目根目录加入到`sys.path`中, `sys.path`指的是`import`时python的查找路径, 为什么子目录中的python文件从项目根路径为起始点`import`不会报错？

解答:

首先我们知道`sys.path`中存储的是`import`时python查找库的搜索路径, 通过打印该变量内容, 我们可知, `''`空字符串是第一个路径, 这个空字符串代表的是当前python文件的所在目录, 不论是绝对路径执行还是相对路径执行, 这个值都是一样的, 都是当前python文件所在的目录.

然后再回答一下上面的问题, 因为我们是以项目根路径下的python文件作为项目的执行文件的, 该文件的`import`相当于把其他当前目录或各级子目录中的各个python文件中的代码(包括`import`代码)都加载到了当前文件中, 所以子目录文件中的`import`也是从项目根路径下开始的, 当然不会报错.

当然了, 我们要是单独执行子目录中的python文件, 就会报找不到模块的错误了.

还有一个问题, 以上我们部署到生成环境当然没问题, 问题是在项目开发过程中, 我们肯定是需要单独执行子目录中的python文件的, 这个时候改怎么配置呢？

解答:

把项目根目录加入到`sys.path`中, 可通过如下方式设置:

```bash
# 设置系统环境变量
export PYTHONPATH="/project/root/path"

# 如果是virtualenv环境, 加入到activate文件末尾即可
cat >>py37env/bin/activate <<EOF
export PYTHONPATH="/project/root/path"
EOF
```

这样子目录python文件在当前目录下找不到对象时, 就会去项目根路径下去找了.
