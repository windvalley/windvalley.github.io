---
title: "Linux命令行编辑Tips"
date: 2009-12-20T11:18:29+08:00
authors:
    - windvalley
categories:
    - Linux
tags:
    - linux
    - command
    - tips
keywords:
    -
draft: false
---

## 命令行的光标移动

`ctrl+b`: 向左移动一个字符.

`ctrl+f`: 向右移动一个字符.

`esc+b`: 向左移动一个单词, 注意, 每移动一次都需要重新按`esc`键.

`esc+f`: 向右移动一个单词, 注意, 每移动一次都需要重新按`esc`键.

`ctrl+a`: 光标移动到行首.

`ctrl+e`: 光标移动到行尾.

> 应避免使用方向键⬅️  或➡️  , 影响效率.

## 删除命令行字符

`ctrl+h` 或 `backspace`: 删除光标前的一个字符.

`ctrl+d`: 删除光标后的一个字符.

`ctrl+w`: 删除(剪切)光标前的一个单词(以空格作为单词分隔).

`ctrl+u`: 删除(剪切)光标到行首的所有字符.

`ctrl+k`: 删除(剪切)光标到行尾的所有字符.

`ctrl+y`: 粘贴之前剪切的内容到光标前.

## 撤销对命令行的编辑

`ctrl+-`: 撤销前一次对命令行的增改删.

## 清空屏幕内容

`ctrl+l` 或 `clear`

## 操作历史命令

`ctrl+p`: 显示上一条历史命令, 应避免使用⬆️.

`ctrl+n`: 显示下一条历史命令, 应避免使用⬇️.

`ctrl+r`: 搜索历史命令, 输入关键字后, 想删除关键字字符, 使用`ctrl+h`;
要执行匹配的命令行`enter`即可, 要让历史命令显示在命令行当不执行`esc`即可.

`ctrl+g`: 退出历史命令搜索模式.

`!!`: 执行上一条命令.

`^foo`: 删除上一条命令里的foo并执行.

`^foo^bar`: 把上一条命令里的foo替换为bar并执行.

`!curl`: 执行最近的以curl开头的命令.

`!curl:p`: 打印最近的以curl开头的命令, 但不执行.

`!$`或`$_`: 上一条命令的最后一个参数, 一般加在新命令后执行.

`!*`: 上一条命令的所有参数, 一般加在新命令后执行.

`!*:p`: 打印上一条命令的所有参数.
