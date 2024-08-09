---
title: "Git常用命令速查"
date: 2015-05-16T12:26:50+08:00
lastmod: 2015-05-16T12:26:50+08:00
categories:
  - OPS
tags:
  - git
keywords:
  -
draft: true
---

## 基础常用命令

### config

`git init`

会在当前目录(工作区)下生成`.git`文件夹, 所有的代码版本信息全部存储在这里.

```bash
git config --global user.name 'foo'
git config --global user.email 'foo@sre.im'
git config --global --list
```

配置全局 user 信息, 以上命令会自动生成`~/.gitconfig`文件.

```bash
git config --local user.name 'bar'
git config --local user.email 'bar@sre.im'
```

配置当前仓库的 user 信息, 在当前仓库不希望使用用户全局配置的情况下使用.

### add

`git add .`或`git add *`、`git add somefiles`

把目录下所有新增的或有修改的文件, 或指定某个或某些文件添加到 git 版本库的暂存区.

`git add -u`

将工作区所有变化的文件添加到暂存区, 注意对新增文件不起作用.

`git rm somefiles` 或 `git mv somefiles somedir`

删除或移动某些文件, 已直接添加到暂存区，可直接`git commit -m""`.

`git status`

查看当前 git 状态信息.

### commit

`git commit -m"fix some bug."`

将暂存区的文件 commit 到版本库的当前分支, 默认为 master 分支.

`git commit -am'变更的注释信息'`

其中的-a 参数是 add, 加上这个参数可以省略掉被修改的文件由工作区加入到版本库暂存区的步骤,注意-a 对新增文件不起作用.

`git commit -amend`

修改最近一次 commit 的注释.

`git reset HEAD~` 或 `git reset HEAD^`

撤销最近一次的 commit, 撤销`git add`, 仅工作区改动保留.

`git reset --soft HEAD~`

撤销最近一次的 commit, 仅仅撤销 commit, 不撤销`git add`,
也就是工作区和暂存区都保留.

`git reset --hard HEAD~`

撤销最近一次的 commit, 撤销`git add`, 工作区改动删除,
也就是完全回到了上一个 commit 版本.

### 版本恢复

`git log` 或 `git log --pretty=oneline`

查看 commit 历史，确定回退到哪个版本.

`git reflog`

查看全部 commit 历史版本，确定回到未来哪个版本.

`git reset --hard HEAD^`

回退到上一个版本.

`git reset --hard e93f611` 或 `git reset --hard HEAD@{1}`

回退到历史某个版本或恢复到未来某个版本.

### 窗口模式

`gitk`(只展示当前分支) 或 `gitk --all`(展示所有分支)

工作区内, 通过窗口模式管理 git 仓库.

## 查看历史版本

```bash
git log  # 查看当前分支的历史版本
git log --oneline
git log --pretty=oneline
git log -n4 或 git log -4 # 最近4个版本
git log --all # 查看所有分支的历史版本
git log test2 # 查看test2分支的历史版本
git log --oneline --all --graph # 图形化
git help log # 查看所有关于log的命令行帮助
git help --web log # 调用浏览器查看关于log的帮助文档
git reflog  # 查看全部commit历史
```

## 分支管理

### 创建分支

```bash
git branch temp # 基于当前commit创建分支
git branch temp2 ea0b75a12002 # 基于历史版本创建分支
git checkout -b test   # 基于当前版本创建分支test并直接切换到分支test
git checkout -b test2  ea0b75a12002  # 基于历史某个版本创建分支test2并切换到test2
git checkout -b fix-test2 test2 # 基于已有的test2分支创建新的分支fix-test2

git branch -av  # 查看有多少分支, 当前属于哪个分支
```

### 删除分支

`git branch -d fix-test1 # -d删除不掉的情况, 用-D`

### 合并分支

比如在 tmp1 分支开发完成后, 要将 tmp1 的合并到 master 分支上, 则:

```bash
git checkout master # 切换到master分支
git merge tmp1 # 将tmp1分支merge到当前分支master
git log --oneline
```

## 版本比较

```bash
git diff HEAD HEAD~2  # 比较当前版本和祖父版本的差异
git diff ea0b75a1 2a02b5a1 # 比较两个commit版本的差异
```
