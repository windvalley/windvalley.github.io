---
title: "Linux下Iptables使用详解"
date: 2015-10-11T10:32:22+08:00
lastmod: 2015-10-11T10:32:22+08:00
categories:
    - OPS
tags:
    - iptables
    - linux
draft: false
---
## iptables架构图

![iptables 架构图](../../imgs/iptables_netfilter_arch.png)

> 1. 上图是我使用processon.com在线绘制的.
> 2. 上图中INPUT链的DNAT功能是CentOS7才开始有的.
> 3. security表用于SELinux, 一般忽略即可.
> 4. 参考链接: https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture

`iptables`是用户设置防火墙规则的工具, 位于用户空间.

`netfilter`是真正的软件防火墙, 位于内核空间, 是内核的一个数据包的处理模块.

`iptables/netfilter`组成Linux平台下的包过滤防火墙, 算是一种网络层的防火墙/路由器, 主要功能: 封包过滤、NAT、数据包修改等.

## 表(功能)在链中的优先顺序

1. `raw`  # 作用仅是通过动作`NOTRACK`使符合规则的封包不被连接跟踪(`ip_conntrack`)
2. `mangle`  # 拆包修改再重新封包, 即修改数据包
3. `nat`  # 地址转换, 连接跟踪
4. `filter`  # 数据包过滤
5. `security`  # 开启selinux才会用到, 忽略

## 看表支持哪些链(关卡), 或者说看链都支持哪些表(功能)

`iptables -t 表名 -nvL`

## 数据包目标地址是本机地址所经过的链(关卡)顺序

1. 网卡
2. `PREROUTING`(数据包从网卡进入之后和在进入路由表之前所经过的链)
3. 路由选择
4. `INPUT`(数据包通过路由表后经过的链, 在`INPUT`链中匹配完规则放行数据包后, 内核将其拆包检查4层端口, 然后交由上层用户空间的相关进程)
5. 用户空间(具体应用)
6. 路由选择
7. `OUTPUT`(由本机应用产生的数据包先经过此链)
8. `POSTROUTING`(到达出口网卡之前的链)
9. 网卡

## 数据包目标地址非本机(转发)所经过的链(关卡)顺序

1. 网卡
2. `PREROUTING`
3. 路由选择
4. `FORWARD`(数据包通过路由表后经过的链)
5. `POSTROUTING`
6. 网卡

> 注意: 从`PREROUTING`链出去的包是要先经过路由判断, 然后选择去`INPUT`链还是去`FORWARD`链

## iptables常用命令参数

- -P 设置链的默认策略
- -F 清空表中链的规则
- -N 创建表的自定义链
- -E 重命名自定义链
- -X 删除自定义的空链
- -Z 数据包和流量计数器都清零
- -I 默认在链首插入规则, 也可指定行插入
- -A 在链的尾部追加
- -R 修改原有规则, 必须严格指明原匹配条件
- -D 删除规则
- -S 打印规则语句
- `-nvL` 打印规则详细信息
- -x 显示计数器的精确值, 比如不显示为`K`或这`M`, 而是直接显示原始`BYTE`
- `--line-number|--line`  对每个链的规则打印序号
- `-t 表名 -F 链名`   清除某表中的某链的全部规则
- `-t 表名 -F` 清除某表的全部规则
- `-t mangle -P INPUT DROP`   为某表的某链设置默认策略
- -i 指定包的流入网卡, 只能用于`PREROUTING,INPUT,FORWARD`链
- -o 指定包的流出网卡, 只能用于`OUTPUT,FORWORD,POSTROUTING`链

> `FORWARD`链可同时使用`-i`和`-o`选项

## 处理动作(-j)

### 内建处理动作

- `ACCEPT` 通过
- `DROP`  丢弃
- `REJECT`  拒绝
- `SNAT,DNAT,MASQUERADE,REDIRECT,RETURN,MARK,LOG,UNTRACK`

### 自定义处理动作

- 自定义的链

### 内建处理动作详细介绍

#### LOG动作

默认将日志记录在`/var/log/message`, 可以自定义记录文件:

```bash
#vim /etc/rsyslog.conf
kern.warning /var/log/iptables.log
#service rsyslog restart
```

`--log-level`选项可以指定记录日志的级别,
`emerg, alert, crit, error, warning, notice, info, debug`

`--log-prefix`选项给日志条目添加标签, 便于区分日志, 方便过滤日志.
举例:
`-j LOG --log-prefix "access-22-port"`

#### SNAT动作

`centos6`只可存在于`POSTROUTING`链, `centos7`只存在于`POSTROUTING`和`INPUT`链).

`-j SNAT --to-source 123.23.1.1`  # 表示将匹配到的报文的源IP修改为`123.23.1.1`

用途: 共享公网IP上网, 隐藏原主机地址.

#### MASQUERADE动作(相当于动态的SNAT)

`-o eth1 -j MASQUERADE`  # 会动态的将源地址转换为指定网卡上的可用公网IP地址, 这样就不担心公网IP地址变更了.

可以把`MASQUERATE`理解为动态的、自动化的`SNAT`, 如果用于上公网的外网IP地址不会频繁变更, 没有必要使用`MASQUERADE`, 因为`SNAT`更加高效.

用途: 源地址要转换为的公网IP地址会经常动态变化.

#### DNAT动作

只可存在于`PREROUTING`, `OUTPUT`链.

`-j DNAT --to-destination 10.1.0.2:8080` # 表示将匹配的报文的目标地址与端口修改为`10.1.0.2:8080`

用途: 通过公网IP访问内网IP服务.

#### REDIRECT动作

只存在于`PREROUTING`, `OUTPUT`链.

`--dport 80 -j REDIRECT --to-ports 8080`  # 将目标端口由`80`改为`8080`

用途: 在本机进行端口映射.

#### RETURN动作(返回主链继续匹配)

从默认链里可以跳转到自定义链里, 这个自定义链就是子链.

从子链`RETURN`后, 会回到默认链里触发跳转到子链的那条规则, 然后从那条规则的下一条规则继续匹配.

如果`RETURN`不是在子链中, 而是在默认链中, 那么就以该默认链的默认策略进行, 这个默认策略指的是通过`-P`设置的默认链的默认策略.

### 自定义链详解

#### 使用自定义链的原因

当默认链的规则非常多, 就不方便管理了, 所以对这些规则分类管理, 也就有了自定义链.

自定义链需要默认链的引用才能生效, 可以被同一个表中的多个默认链同时引用.

#### 创建自定义链

`iptables -t filter -N WEB`

#### 使用默认链来引用自定义链

`iptables -t filter -I INPUT -p tcp --dport 80 -j WEB`  # 经过`filter`表的`INPUT`链中的目标端口是`80`的数据包全部去匹配自定义链WEB中的规则

`iptables -t filter -I INPUT -j WEB`  # 经过`filter`表的`INPUT`链的所有数据包全部去匹配WEB链中的规则

#### 重命名自定义链

`iptables -E WEB WEB_1`

#### 删除自定义空链

`iptables -X WEB`

## 数据包被跟踪连接的四种状态

`NEW,ESTABLISHED,RELATED,INVALID`, 在使用`RAW`表的`NOTRACK`动作后, 会增加一个状态: `UNTRACKED`.

除了本机产生的数据包由`NAT`表的`OUTPUT`链处理连接跟踪外, 其他数据包都是在`NAT`表的`PREROUTING`链中进行处理的.

如果发送一个流的初始化数据包, 状态就会在`NAT`表的`OUTPUT`链里被设置为`NEW`, 当收到回应的数据包时, 状态就会在`NAT`表的`PREROUTING`链里被设置为`ESTABLISHED`；如果第一个数据包不是本机生成的, 那就会在`NAT`表`PREROUTING`链里被设置为`NEW`状态, 所以所有状态的改变和计算都是在`NAT`表中的`PREROUTING`链和`OUTPUT`链里完成的.

`NEW`   # `NEW`状态的数据包说明这个数据包是收到的第一个数据包.

`ESTABLISHED`  # 只要发送并接到应答, 一个数据连接就从`NEW`变为`ESTABLISHED`, 而且该状态会继续匹配这个连接后继数据包.

`RELATED`  # 当一个连接和某个已处于`ESTABLISHED`状态的连接有关系时, 就被认为是RELATED, 也就是说, 一个连接想要是`RELATED`的, 首先要有个`ESTABLISHED`的连接, 这个`ESTABLISHED`连接再产生一个主连接之外的连接, 这个新的连接就是`RELATED`.

`INVALID`  # 说明数据包不能被识别属于哪个连接或没有任何状态.

`UNTRACKED`   # 数据包不要被跟踪

`-m state --state NEW,ESTABLISHED`

## 练习

### 实例1

在CentOS7中像在CentOS6中那样使用service操作iptables.

```bash
yum install -y iptables iptables-services
systemctl stop firewalld
systemctl disable firewalld
systemctl start iptables
systemctl enable iptables

# 将系统中的规则保存到指定文件
iptables-save >/etc/sysconfig/iptables-1

# service iptables save 会默认将文件保存到/etc/sysconfig/iptables

# 将某文件的规则重载到系统中
iptables-restore < /etc/sysconfig/iptables-1
```

### 实例2

使用iptables把Linux配置成网络防火墙(非主机防火墙).

1, 使内核支持数据包转发

临时生效

`echo 1 > /proc/sys/net/ipv4/ip_forward` 或 `sysctl -w net.ipv4.ip_forward=1`

永久生效

配置`/etc/sysctl.conf`文件(centos7中配置`/usr/lib/sysctl.d/00-system.conf`文件), 在配置文件中将`net.ipv4.ip_forward`设置为`1`.

2, 在`filter`表中的`FORWARD`链中设置规则, 使用白名单机制, 配置规则要考虑数据包的两个方向

```bash
# 示例为允许网络内主机访问网络外主机的web服务与sshd服务.
iptables -P FORWARD ACCEPT
iptables -A FORWARD -s 10.1.0.0/16 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -d 10.1.0.0/16 -p tcp --sport 80 -j ACCEPT
iptables -A FORWARD -s 10.1.0.0/16 -p tcp --dport 22 -j ACCEPT
iptables -A FORWARD -d 10.1.0.0/16 -p tcp --sport 22 -j ACCEPT
iptables -A FORWARD -j DROP

# 可以使用state扩展模块, 对上述规则进行优化, 使用如下配置可以省略许多"回应报文放行规则".
iptables -P FORWARD ACCEPT
iptables -A FORWARD -s 10.1.0.0/16 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -s 10.1.0.0/16 -p tcp --dport 22 -j ACCEPT
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -j DROP
```

### 实例3

解决`ip_conntrack: table full, dropping packet`的问题.

当服务器必须启用防火墙, 且服务器数据包流量过大时, 就会出现这种问题, 解决方法有如下两个, 第一种方法对服务器数据包流量还不是特别大的情况下有长期效果, 当服务器流量过大时, 这种方法只能是短暂的缓解, 无法根治. 推荐第二种方法, 可根本解决问题.

1, 扩容, 即增大相关参数数值(针对CentOS6及以上系统)

```bash
sysctl -w net.netfilter.nf_conntrack_max=1655360
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=1200

# 以上是临时生效, 重启系统后将失效, 可以写在/etc/sysctl.conf文件里使永久生效:
cat >> /etc/sysctl.conf <<EOF
net.netfilter.nf_conntrack_max = 1655360
net.netfilter.nf_conntrack_tcp_timeout_established = 1200
EOF
```

2, 对造成服务器流量大的业务数据包设置不追踪规则, 比如web服务

```bash
iptables -t raw -A PREROUTING -p tcp --dport 80 -j NOTRACK
iptables -t filter -A FORWARD -m state --state UNTRACKED -j ACCEPT  # 用于该主机转发数据包的情况
```

## 一些TIPS

1. 注意规则的顺序, 更严格的规则应该放到前面, 因为只要匹配到一条规则就直接执行该规则的动作了, 不再进行后续规则的匹配(`LOG`动作的规则除外).

2. 将预计会匹配频率高的规则优先放到前面, 这样可以减少匹配条目, 提高访问性能.

3. 如果将`iptables`所在防火墙作为网络防火墙使用, 配置规则时要考虑数据包的方向, 双向都要考虑好.

4. 配置`iptables`白名单时, 不要将链的默认策略设置为`DROP`, 因为当链中的规则被清空时, 管理员的请求也会被`DROP`掉, 比如管理员执行了`iptables -F`则直接被挡在门外了, 因为`-F`不会清除链的默认策略. 正确做法是链的默认策略设置为`ACCEPT`, 链的最后一条规则设置所有包都`DROP`, 来实现白名单机制.
