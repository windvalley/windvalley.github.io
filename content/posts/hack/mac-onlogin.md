---
title: "实现Mac开机自动启动指定服务"
date: 2018-06-09T15:35:44+08:00
authors:
  - windvalley
categories:
  - Hack
tags:
  - mac
  - tips
keywords:
  -
draft: false
---

本文主要讲的是 macOS 重新启动后, 自动运行我们指定的服务, 实现开机自启动.

比如我们要实现让 hugo 本地 sever 开机自启动.

> macOS 的开机自启说明文档参考:
> `man launchd`

## 创建并配置服务的 plist 文件

```bash
cat > ~/Library/LaunchAgents/localhost.hugo.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>KeepAlive</key>
  <true/>
  <key>Label</key>
  <string>localhost.hugo</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/hugo</string>
    <string>serve</string>
    <string>-D</string>
    <string>--disableFastRender</string>
    <string>-e</string>
    <string>production</string>
    <string>--logFile=process.log</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>WorkingDirectory</key>
  <string>/Users/xinggang/github/sre.im</string>
</dict>
</plist>
EOF
```

> 注意:

> 服务的启动命令不能放置到后台, 比如启动命令结尾不能加`&`;

> `KeepAlive`设置为`true`, 系统默认每 10 秒会检查一次服务进程的存活,
> 发现无相关进程会重新启动.

> `Label`表示服务的名称, 一般我们以服务 plist 配置文件的名称即可, 不可加 plist 扩展名.

> `ProgramArguments`表示带参数的命令行, 命令必需使用绝对路径, 每个参数独立一行;

> `WorkingDirectory`表示命令在哪个目录下运行, 即工作目录,
> 这个路径也必需是绝对路径;

> `RunAtLoad`表示在将该配置文件注册(load)到系统中时就运行服务的启动命令;

> 最后再强调一遍, 命令的路径和工作目录都必需是绝对路径, 使用`$HOME`和`~`都不可以.

## 将配置文件注册到系统中

```bash
launchctl load ~/Library/LaunchAgents/localhost.hugo.plist
```

查看是否已注册成功: `launchctl list|grep hugo`

如果注册成功, 执行`ps aux|grep hugo`可发现 hugo 本地 server 已经启动了.

取消注册的命令: `launchctl unload ~/Library/LaunchAgents/localhost.hugo.plist`

> 注意:

> `launchctl load`这个步骤可以没有, 重启后 macOS 会自动注册并启动.

> 如果我们不想让 hugo 开机自启动了, 需要将该文件`localhost.hugo.plist`移除.

## 总结

有很多 brew 安装的软件, 比如 mysql, 通过`brew services start mysql`,
即可实现开机自启, 这个原理和上面介绍的方式是一致的. 我们可以在
`~/Library/LaunchAgents`目录中找到 mysql 的 plist 文件.

对于其他无法通过`brew services start`的方式来实现开机自启的自定义服务,
我们就可以按照上面介绍的方式进行了, 非常友好方便.
