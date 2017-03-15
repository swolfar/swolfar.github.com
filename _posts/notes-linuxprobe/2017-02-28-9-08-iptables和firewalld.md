---
layout: post
title: "iptables 与 firewalld 防火墙"
date: 2017-2-28
description: iptables 与 firewalld 防火墙
tags: 
 - linux
 - RHCE 
 - iptables
 - firewalld
---

# iptables 与 firewalld 防火墙

----------

**简述**
- 保证数据安全性
- 防火墙
- iptables 与 firewalld 的关系
- 防火墙配置工具：iptables、firewall-cmd、firewall-config、tcp_wrapper

----------

## 防火墙管理工具

依据策略对外部请求进行过滤，可监控每一个数据包，逐条匹配策略规则，直到符合某条策略规则为止，基于来源地址、请求动作、协议，最终仅让合法的用户请求流入到内网中，其余的均被丢弃。

Iptables 服务与 Firewalld 服务都不是真正的防火墙，它们都只是用来定义防火墙策略功能的“防火墙管理工具”而已，真正定义好的防火墙策略会被内核中的 netfilter 网络过滤器接收，从而真正实现了防火墙功能。

----------

## iptables 防火墙策略规则管理工具

**策略与规则链**

对防火墙策略的设置无非有两种，一种是“通”，一种是“堵”。

当防火墙的默认策略是拒绝的，咱们就要设置允许规则，否则谁都进不来了，而如果防火墙的默认策略是允许的，咱们就要设置拒绝规则，否则谁都能进来了，起不到防范的作用。

iptables 把对数据进行过滤或处理数据包的策略叫做规则，规则 → 链 → 表

**链（chains）**：

名称|作用
----|----
PREROUTING |在进行路由选择前处理数据包
INPUT |处理流入的数据包
FORWARD |处理转发的数据包
OUTPUT |处理流出的数据包
POSTROUTING |在进行路由选择后处理数据包

使用最多的就是 INPUT 数据链，这个链中定义的规则起到了保证咱们私网设施不受外网骇客侵犯的作用。

**数据处理方式**

参数|作用
----|----
ACCEPT|允许流量通过
LOG|记录日志信息
REJECT|拒绝流量通过，并作出回应
DROP|拒绝流量通过，不作回应

**基本命令参数**

“四表五链”

根据数据流量的源地址、目的地址、传输协议、服务类型等进行匹配，匹配方式自上而下，逐条匹配

参数	|作用
----|----
-P	 | 设置默认策略:iptables -P INPUT (DROP|ACCEPT)
-F | 清空规则链
-L	 | 查看规则链
-A | 在规则链的末尾加入新规则
-I num	|在规则链的头部加入新规则
-D num|	删除某一条规则
-s	|匹配来源地址 IP/MASK，加叹号"!"表示除这个 IP 外。
-d	|匹配目标地址
-i 网卡名称 	| 匹配从这块网卡流入的数据
-o 网卡名称 | 	匹配从这块网卡流出的数据
-p |	匹配协议,如tcp,udp,icmp
--dport num	| 匹配目标端口号（对外来数据即本机端口）
--sport num	| 匹配来源端口号（对外来数据即请求断端口）

`iptables` 命令

```
iptables -L  # 查看已有的防火墙策略

iptables -F  # 清空已有的防火墙策略

iptables -P INPUT DROP  # -P （policy） 修改 INPUT 链的默认策略
# 规则链的默认策略拒绝动作只能是 DROP，而不能是 REJECT。

iptables -I INPUT -p icmp -j ACCEPT  # 允许 icmp 协议数据包流入

iptables -D INPUT 1  # 删除 INPUT 链中的第 1 条策略

iptables -P INPUT ACCEPT  # 默认策略还原为允许 
```

防火墙策略是按照**从上至下顺序匹配**的，将限制较具体，较严格的规则放在限制小，范围大的规则之前。

```
# 设置 INPUT 链只允许指定网段访问本机的 22 端口，拒绝其他所有主机的数据请求流量：
iptables -I INPUT -s 192.168.10.0/24 -p tcp --dprot 22 -j ACCEPT
iptables -I INPUT -P tcp --dport 22 -j REJECT

或者
iptables -I INPUT ! -s 192.168.10.0/24 -p tcp --dport 22 -j REJECT
```

```
# 向 INPUT 链中添加拒绝所有人访问本机 12345 端口的防火墙策略：
iptables -I INPUT -p tcp --dport 12345 -j REJECT
iptables -I INPUT -p udp --dport 12345 -j REJECT
```

```
# 向 INPUT 链中添加拒绝来自于指定 192.168.10.5 主机访问本机 80 端口（web 服务）的防火墙策略：
iptables -I INPUT -s 192.168.10.5 -p tcp --dport 80 -j REJECT 
```

```
# 向 INPUT 链中添加拒绝所有主机不能访问本机 888 端口和 1000 至 1024 端口的防火墙策略：
iptables -A INPUT -p tcp --dport 888 -j REJECT
iptables -A INPUT -p udp --dport 888 -j REJECT
iptables -A INPUT -p tcp --dport 1000:1024 -j REJECT
iptables -A INPUT -p udp --dport 1000:1024 -j REJECT
```

让配置的防火墙策略永久的生效

```
service iptables save

# 查看服务对应的端口号
cat /etc/services
```

----------

## 8.3 firewalld 动态防火墙管理器服务

同时拥有命令行终端和图形化界面的配置工具

支持了动态更新技术，而且加入了“ zone 区域”的概念，预定义模板

根据生产场景的不同而选择合适的策略集合，实现了防火墙策略之间的快速切换。

常见的 zone 区域名称及应用可见下表（默认为 public）：

按默认信任级别排序（不受信到受信）

区域	|默认规则策略
----|-----------
drop|拒绝流入的数据包，除非与输出流量数据包相关。
block|拒绝流入的数据包，除非与输出流量数据包相关。
public|拒绝流入的数据包，除非与输出流量数据包相关或是 ssh,dhcpv6-client 服务则允许。
external|拒绝流入的数据包，除非与输出流量数据包相关或是 ssh 服务则允许。
dmz	|拒绝流入的数据包，除非与输出流量数据包相关或是 ssh 服务则允许。
work	|拒绝流入的数据包，除非与输出流量数据包相关或是 ssh,ipp-client 与 dhcpv6-client 服务则允许。
home	|拒绝流入的数据包，除非与输出流量数据包相关或是 ssh,mdns,ipp-client,samba-client 与 dhcpv6-client 服务则允许。
internal|等同于 home 区域
trusted	|允许所有的数据包。

----------

## 终端管理工具

firewall-cmd

参数	|作用
----|----
--state | 获取 firewalld 状态
--reload	| 让“永久生效”的配置规则立即生效，覆盖当前的。
--get-zones	|显示可用的区域。
--get-default-zone|	查询默认的区域名称。
--set-default-zone=<区域名称>|	设置默认的区域，永久生效。
--get-services	|显示预先定义的服务。
--get-active-zones|	显示当前正在使用的区域与网卡名称。
--add-source=	|将来源于此IP或子网的流量导向指定的区域。
--remove-source=|	不再将此IP或子网的流量导向某个指定区域。
--add-interface=<网卡名称>	|将来自于该网卡的所有流量都导向某个指定区域。
--change-interface=<网卡名称>|	将某个网卡与区域做关联。
--list-all|	显示当前区域的网卡配置参数，资源，端口以及服务等信息。
--list-all-zones	|显示所有区域的网卡配置参数，资源，端口以及服务等信息。
--add-service=<服务名>	|设置默认区域允许该服务的流量。
--add-port=<端口号/协议>	|允许默认区域允许该端口的流量。
--remove-service=<服务名>	|设置默认区域不再允许该服务的流量。
--remove-port=<端口号/协议>	|允许默认区域不再允许该端口的流量。
--permanent | 永久生效参数

----------

--state
--reload
--permanent
--panic-on
--panic-off
--set-
--zone

--add-  # 添加
--change- #改变
--get-   # 获取
--list-  # 列出
--query-  # 查询
--remove-  # 移除

----------

Firewalld 服务对防火墙策略的配置默认是当前生效模式（RunTime）

永久生效模式（permanent），设置的策略需要重启后才能自动生效，如果想让配置的策略立即生效的话需要手动执行一下--reload 参数。

一定要细心看到底是对 Runtime 还是 Permanent 模式操作，

`firewall-cmd` 命令

- 查看 Firewalld 服务当前所使用的zone区域：
```
firewall-cmd --get-default-zone
```

- 查询 eno16777728 网卡在 Firewalld 服务中的哪个 zone 区域：
```
firewall-cmd --get-zone-of-interface=eno16777728
```

- 将 Firewalld 防火墙服务中 eno16777728 网卡的默认区域修改为 external，重启后再生效：
```
firewall-cmd --permanent --zone=external --change-interface=eno16777728
firewall-cmd --get-zone-of-interface=eno16777728
firewall-cmd --permanent --get-zone-of-interface=eno16777728
firewall-cmd --reload
```

- 将 Firewalld 防火墙服务的当前默认 zone 区域设置为 dmz：
```
firewall-cmd --set-default-zone=dmz
```

- 启动/关闭 Firewalld 防火墙服务的应急状况模式，阻断一切网络连接（当远程控制服务器时请慎用。）：
```
firewall-cmd --panic-on
firewall-cmd --panic-off
```

- 查询在 public 区域中的 ssh 与 https 服务请求流量是否被允许：
```
firewall-cmd --zone=public --query-service=ssh
firewall-cmd --zone=public --query-service=https
```

- 将 Firewalld 防火墙服务中 https 服务的请求流量设置为永久允许，并当前立即生效：
```
firewall-cmd --add-service=https
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

- 将 Firewalld 防火墙服务中 http 服务的请求流量设置为永久拒绝，并当前立即生效：
```
firewall-cmd --permanent --remove-service=https
firewall-cmd --reload
```

- 将 Firewalld 防火墙服务中 8080 和 8081 的请求流量允许放行，但仅限当前生效：
```
firewall-cmd --add-port=8080-8081/tcp
firewall-cmd --list-ports
```

- 将原本访问本机 8888 端口号的请求流量转发到 22 端口号，要求当前和长期均有效：

> 流量转发命令格式：firewall-cmd [--permanent] [--zone=区域] --add-forward-port=port=源端口号:proto=协议:toport=目标端口号:toaddr=目标 IP 地址

```
firewall-cmd --add-forward-port=port=8888:proto=tcp:toport=22
firewall-cmd --list-forward-ports
```

- 在 Firewalld 防火墙服务中配置一条富规则，拒绝所有来自于 192.168.10.0/24 网段的用户访问本机 ssh 服务（22端口）：
```
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="192.168.10.0/24" service name="ssh" reject"
firewall-cmd --reload
```

----------

## 图形管理工具

firewall-config 命令是管理 Firewalld 防火墙策略的图形化工具

只要能实现需求的功能，无论用文本管理工具还是图形管理工具都是可以的。

----------

## SNAT 源地址转换协议

使得多个内网用户通过同一个外网 IP 地址上网。

生产环境中有多块网卡在同时提供着服务（这种情况很常见），对内网和对外网的网卡要选择的防火墙策略区域也不应是一样的。

----------

## 服务的访问控制列表

tcp_wrappers 红帽 RHEL7 系统中默认已经启用的一款流量监控程序

根据**来访主机地址**与**本机目标服务程序**做允许或拒绝操作。

两个层面的防火墙:
第一种是前面讲到的基于 **TCP/IP** 协议的流量过滤防护工具
第二种是 Tcp_wrappers 服务则是能够对**系统服务**进行允许和禁止的防火墙

控制列表文件修改后会立即生效

系统将会先检查允许策略规则文件（/etc/hosts.allow），如果匹配到相应的允许策略则直接放行请求

如果没有匹配则会去进一步匹配拒绝策略规则文件（/etc/hosts.deny）的内容，有匹配到相应的拒绝策略就会直接拒绝该请求流量

如果两个文件全都没有匹配到的话也会默认放行这次的请求流量。

常见的情况：

客户端类型	|示例	|满足示例的客户端列表
----------|-----|----------------------
单一主机	|192.168.10.10	|IP地址为 192.168.10.10 的主机。
指定网段	|192.168.10.	|IP段为192.168.10.0/24的主机。
指定网段	|192.168.10.0/255.255.255.0	|IP段为 192.168.10.0/24 的主机。
指定DNS后缀|	.linuxprobe.com	|所有DNS后缀为 .linuxprobe.com 的主机
指定主机名称|	www.linuxprobe.com	|主机名称为 www.linuxprobe.com 的主机。
指定所有客户端|	ALL	|所有主机全部包括在内。

两点原则必须要提前讲清楚：
第一，在写禁止项目的时候一定要写上的是服务名称，而不是某种协议的名称
第二，推荐先来编写拒绝规则，这样可以比较直观的看到相应的效果。

- 通过拒绝策略文件禁止所有访问本机 sshd 服务的请求数据

```
vim /etc/hosts.deny
---------------------
#
sshd:*

vim /etc/hosts.allow
------------------
#
sshd:192.168.10.2
```

----------

## 本章命令汇总

`iptables` 命令：命令行的防火墙策略管理工具

`firewall-cmd` 命令：动态防火墙管理器服务

`firewall-config` 命令：图形界面防火墙管理工具

`tcp_wrappers` : 流量监控程序，对系统服务进行允许和禁止的防火墙

----------

## 作业

1:红帽RHEL7系统中iptables已经被firewalld服务彻底取代？

答案:错，在系统中iptables和firewalld服务均可使用。

2:请您简述下防火墙拒绝规则中的DROP和REJECT动作有何不同之处？

答案:DROP动作是丢包，不响应，而REJECT动作是拒绝请求，回应给对方拒绝信息。

3:如何将iptables服务的INPUT规则链默认策略设置为DROP动作？

答案:执行命令iptables -P INPUT DROP后即可生效。

4:使用iptables命令配置禁止来自于192.168.10.0/24网段访问本机sshd服务（22端口）的防火墙策略应该怎么写？

答案:执行命令iptables -I INPUT -s 192.168.10.0/24 -p tcp --dport 22 -j REJECT后即可生效。

5:请您简述下firewalld防火墙管理工具的zone区域作用？

答案:我们可以依据不同的工作场景来调用不同的firewalld区域，实现对大量防火墙策略的快速切换。

6:如何在firewalld防火墙管理工具中将默认的zone区域设置为dmz？

答案:执行命令firewall-cmd --set-default-zone=public后即可生效。

7:如何让firewalld防火墙管理工具中永久生效策略（Permanent）立即生效可用呢？

答案:执行命令firewall-cmd --reload后即可生效。

8:使用SNAT源地址转换协议的目的是什么？

答案:SNAT源地址转换协议是为了解决IP地址资源匮乏问题而设计的技术协议，SNAT技术能够使得多个内网用户通过一个外网IP地址上网。

9:Tcp_wrappers服务分别有允许策略配置文件和拒绝策略配置文件，请问匹配顺序是怎么样的？

答案:Tcp_wrapper会依次匹配允许策略配置文件→拒绝策略配置文件，如果都没有匹配到也会默认放行流量。

----------

2017-2-28 by Achxku







