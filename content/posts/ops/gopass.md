---
draft: false

title: 使用 Gopass 管理你的密码
subtitle: ""
description: ""

date: 2022-12-11T22:53:45+08:00
lastmod: 2022-12-11T22:53:45+08:00

author: ""
authorLink: ""

tags: ["security", "password"]
categories: ["OPS"]

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

本文基于 MacOS 系统说明安装和使用 `gopass` 工具.

## 终端管理 gopass

### 安装 gopass

#### 二进制安装

```sh
mkdir tmp && cd tmp

# 从官网找最新稳定版下载链接进行下载.
wget -c https://github.com/gopasspw/gopass/releases/download/v1.15.0/gopass-1.15.0-darwin-amd64.tar.gz
tar zxf gopass-1.15.0-darwin-amd64.tar.gz
cp gopass /usr/local/bin/

# 清理文件
cd .. && rm -rf tmp
```

#### 配置 gopass 命令补全

```sh
# 针对使用zsh的用户
gopass completion zsh > $ZSH/cache/completions/_gopass
exec zsh
```

### 构建联动 github 的 gopass 仓库

#### 1. 在 github 上创建空的私有 pass 仓库

比如: `git@github.com:demo/pass.git`

#### 2. 本地创建联动 github 仓库的密码仓库

```sh
gopass --yes setup --remote git@github.com:demo/pass.git \
  --alias demo \
  --create --name "XG" \
  --email "demo@gmail.com"
```

输出:

```text
/'_ '\ /'_'\ ( '_'\  /'_' )/',__)/',__)
( (_) |( (_) )| (_) )( (_| |\__, \\__, \
'\__  |'\___/'| ,__/''\__,_)(____/(____/
( )_) |       | |
 \___/'       (_)

🌟 Welcome to gopass!
🌟 Initializing a new password store ...
Creating a new team ...
🌟 Configuring your password store ...
✅ Configuration written
[demo] Initializing your shared store ...
[demo] ✅ Done. Initialized the store.
[demo] Configuring the git remote ...
[demo] ✅ Done. Created Team "demo"
```

#### 3. 直接 clone 已有 github 密码仓库

如果 github 上已经有 `gopass` 仓库, 则忽略 1 和 2, 直接 clone 到本地即可.

```sh
gopass --yes setup --remote git@github.com:demo/pass.git \
  --alias demo --name "XG" --email "demo@gmail.com"
```

### 删除本地密码仓库的方法

仅在需要重新从零开始安装 `gopass` 的场景使用.

```sh
# 删除本地github密码仓库, demo是仓库别名.
gopass mounts remove demo
# 删除本地默认的root密码库.
rm -rf ~/.local/share/gopass
```

### 按需更改 gopass 配置文件

配置文件路径默认为: `~/.config/gopass/config`

```toml
# gopass setup 过程自动生成的.
[mounts]
path = /Users/xg/.local/share/gopass/stores/root
# gopass setup 过程自动生成的.
[mounts "demo"]
path = /Users/xg/.local/share/gopass/stores/demo

[core]
# 使用 gopass generate <pass-name> <pass-length> 生成密码后是否自动将生成的密码拷贝到系统剪贴板.
# 默认值: false
autoclip = true
# 如果autoclip参数为true, 或使用了-c参数, 生成密码后密码在剪贴板停留的最大时间.
# 默认值: 45秒
cliptimeout = 45
# 本地github密码库有新密码存储时, 是否自动git push推送到远程.
# 默认值: true
autosync = true
# 桌面终端是否给出必要的提示, 比如密码拷贝到了系统剪贴板, git push操作的结果等.
# 默认值: true
notifications = false
# 只输出安全的内容信息, 密码不输出, 如果使用 -f 参数, 还是会输出密码.
# 默认值: false
showsafecontent = true
# 使用 gopass show 命令时, 是否自动将密码拷贝到系统剪贴板.
# NOTE: 如果设置成true, tmux-gopass将无法正常使用, 因为输出的密码包含了提示信息.
# 默认值: false
showautoclip = false

[generate]
# gopass generate 生成的密码是否包含特殊字符.
# NOTE: 目前测试此项设置不生效.
# 默认值: fasle
symbols = true
# gopass generate 生成的密码的默认长度是多少.
# NOTE: 目前测试发现此项设置不生效, 而系统环境变量有效: export GOPASS_PW_DEFAULT_LENGTH=12
# 默认值: 24
length = 12
```

### gopass 常用命令行

```sh
# 列出所有密码条目名称
gopass ls

# 非交互式删除(-f)密码条目foo
gopass rm -f foo

# 通过vim编辑密码条目bar
gopass edit bar

# 新建密码项foo, 密码长度12, 自动生成密码
gopass generate foo 12

# 新建密码项servers/host1.sre.im, 交互式输入自定义密码
gopass insert servers/host1.sre.im

# 通过vim输入自定义密码
gopass insert -m servers/host1.sre.im

# 模糊查询密码项
gopass show foo

# 模糊查询密码项, 并将密码拷贝到系统剪贴板
gopass show -c foo

# 模糊查询密码项, 并只显示密码信息
gopass show -o foo

# 纯交互式创建密码项
gopass new

# 生成一组密码显示在屏幕上, 生成的密码至少包含一个特殊字符, 密码长度16(不输数值的话默认12)
# NOTE: 其他常用的参数
#   --ambiguous 不包含容易混淆的字符, 比如1和l
#   --no-capitalize 不包含大写字母
#   --no-numerals 不包含数字
#   --one-per-line 每行只显示一个密码
gopass pwgen --symbols 16
```

## 浏览器联动 gopass

Chrome 浏览器安装`GopassBridge`插件联动本地的`gopass`密码.

### 安装浏览器插件 GopassBridge

点击 <https://chrome.google.com/webstore/detail/gopass-bridge/kkhfnlkhiapbiehimabddjbimfaijdhk> 安装浏览器插件.

### 安装 gopass-jsonapi

在 <https://github.com/gopasspw/gopass-jsonapi/releases> 页面下载和`gopass`同版本的 `gopass-jsonapi` 工具.

```sh
mkdir tmp && cd tmp

wget -c https://github.com/gopasspw/gopass-jsonapi/releases/download/v1.15.1/gopass-jsonapi-1.15.1-darwin-amd64.tar.gz
cp gopass-jsonapi /usr/local/bin/

cd .. && rm -rf tmp
```

### 配置 gopass-jsonapi

```sh
# 根据提示进行配置, 最后会生成 ~/.config/gopass/gopass_wrapper.sh
gopass-jsonapi configure
```

配置完成后, Chrome 浏览器插件 `GopassBridge` 和本地的 `gopass` 服务就联动起来了.

此时如果 `gopass` 存储了某个网站的密码, 当通过浏览器访问这个网站登陆页的时候, 点击 `GopassBridge` 插件即可出现相关的条目供选择注入密码.

## Tmux 调用 gopass

### 安装配置 tmux-gopass 插件

使用 `Tmux Plugin Manager` 方式安装, 在 `~/.tmux.conf` 合适的位置添加如下配置:

```tmux
# 调用 gopass 保存的密码.
set -g @plugin 'haru-ake/tmux-gopass'
# 使用哪种过滤工具, 可选peco或fzf, 需提前安装peco或fzf.
set -g @gopass-filter-program 'fzf'
# 设置gopass密码窗格大小.
# NOTE: 如下2条配置默认只能配置一条, 另一条必须为空.
# 1) 固定大小, 比如: 10
set -g @gopass-pane-size ''
# 2) 按当前窗格的百分比
set -g @gopass-pane-percentage '40'
# 设置显示gopass密码条目窗口的快捷建:
# NOTE: 如下3条配置的快捷键如果相同, 只有@gopass-horizontal-split-pane-key 会生效.
# 1) 新建一个窗口显示gopass密码条目.
set -g @gopass-new-pane-key ''
# 2) 在当前窗格右侧新建一个窗格显示gopass密码条目.
set -g @gopass-horizontal-split-pane-key ''
# 3) 在当前窗格下方新建一个窗格显示gopass密码条目
set -g @gopass-vertical-split-pane-key 'g'
```

然后 `prefix + I` 进行安装插件和配置加载.

> 关于 tmux 的完整配置以及使用技巧, 请参考我的 dotfiles 项目: <https://github.com/vimhack/dotfiles>

### 使用快捷键调用 gopass

`prefix + g` 即可通过创建新的 pane 的方式来显示 gopass 密码条目, 然后选择需要的条目, 即可输出密码.

## 参考链接

> https://www.gopass.pw/
>
> https://github.com/gopasspw/gopass
>
> https://github.com/gopasspw/gopass/blob/master/docs/setup.md
>
> https://github.com/gopasspw/gopass/blob/master/docs/config.md
>
> https://github.com/gopasspw/gopass/blob/master/docs/features.md
>
> https://github.com/gopasspw/gopassbridge
>
> https://github.com/gopasspw/gopass-jsonapi
>
> https://github.com/haru-ake/tmux-gopass
