---
title: "使用shellcheck工具对Shell代码进行检查"
date: 2014-03-22T18:58:30+08:00
lastmod: 2014-03-22T18:58:30+08:00
categories:
    - Shell
tags:
    - shell
draft: false
---

为了能写出规范、健壮的Shell代码, 推荐使用shellcheck对我们的Shell代码做检查.

github地址: `https://github.com/koalaman/shellcheck`

> 有很多小星星✨, 可见这个工具好处多多😄.

## 使用方式1

直接将`shell`代码粘贴到下面的网址进行检查.

`https://www.shellcheck.net`

显然这种方式不太推荐.


## 使用方式2

安装到操作系统.

```bash
# macos
brew install shellcheck

# centos:
yum install -y epel-release
yum install -y ShellCheck
```

使用方法:

`shellcheck yourscript.sh`

## 使用方式3

安装vim插件[`syntastic`](https://github.com/vim-syntastic/syntastic).

该插件几乎支持所有主流语言的语法检查, 详见github上的文档说明.

推荐使用这种方式, 在我们使用vim编写代码的过程中,
每次进行`:w`保存时都会进行检查, 给出风格或错误提示, 使编码调试效率大幅提高.

```bash
# 安装语法检查插件syntastic
mkdir -p ~/.vim/autoload ~/.vim/bundle &&
    curl -LSso ~/.vim/autoload/pathogen.vim https://tpo.pe/pathogen.vim

cat >> ~/.vimrc <<EOF
execute pathogen#infect()
EOF

cd ~/.vim/bundle &&
    git clone --depth=1 https://github.com/vim-syntastic/syntastic.git
```

vim命令行模式下: `:Helptags` 不报错, 则说明安装成功.

使vim命令行模式下`:wq`退出文件时不做检查, 只在`:w`时才做检查.

```bash
cat >>~/.vimrc <<EOF
" :wq退出文件时不检查语法或规范错误
let g:syntastic_check_on_wq = 0
EOF
```
