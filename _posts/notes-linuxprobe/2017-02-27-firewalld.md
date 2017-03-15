---
layout: post
title: "官方 firewalld wiki 翻译"
date: 2017-2-27 
description: 官方 firewalld wiki 翻译
tags: 
 - linux
 - RHCE 
 - firewalld
---

# 官方 firewalld wiki 翻译
https://fedoraproject.org/wiki/Firewalld

----------

## 1 动态防火墙 firewalld 服务

----------

firewalld 提供了一个防火墙动态管理，支持网络/防火墙区域来定义网络连接或接口的信任级别。它支持 IPv4，IPv6 防火墙设置和以太网网桥，并且具有运行时和永久配置选项的分离。它还支持服务或应用程序直接添加防火墙规则的接口。

以前使用的 system-config-firewall / lokkit 防火墙模型是静态的，每次更改都需要完全重新启动防火墙。还包括防火墙内核模块卸载与重新加载所需的新选项。卸载防火墙内核模块会打破防火墙的运行状态，终断网络连接。

另一方面，firewalld（防火墙守护程序）动态管理防火墙并应用更改，而不需要重新启动整个防火墙。因此，不需要重新加载所有防火墙内核模块。但是使用 firewalld （防火墙守护程序）需要防火墙所有修改完成，同时确保守护程序中的状态和内核中的防火墙保持同步。防火墙守护程序无法解析由 ip * tables 和 ebtables 命令行工具添加的防火墙规则。

守护程序通过 D-BUS 提供有关当前活动的防火墙设置的信息，并接受通过使用 PolicyKit 身份验证的 D-BUS 的更改。

官方 firewalld 主页 [firewalld.org](http://www.firewalld.org/)

## 1.1 守护进程

----------

应用程序，守护程序和用户可以请求通过 D-BUS 启用防火墙功能。功能之一是预定义的防火墙功能，如服务，端口和协议组合，端口/数据包转发，伪装或 icmp 阻止。该功能可以启用一段时间，也可以再次禁用。

使用所谓的直接接口，其他服务（例如 libvirt）能够使用 iptables 参数和参数来添加自己的规则。

netfilter 防火墙帮助程序，例如用于 amanda，ftp，samba 和 tftp 服务，也由守护程序处理，只要它们是预定义服务的一部分。加载附加助手不是当前界面的一部分。对于某些帮助器，只有在模块处理的所有连接都关闭后才能卸载。因此，在这里连接跟踪信息很重要，需要考虑。

## 1.2 静态防火墙 ( system-config-firewall/lokkit )

----------

实际静态防火墙模型 system-config-firewall 和 lokkit 仍然可用，但是不能和 firewalld 同时使用。用户或管理员可以通过启用相应的服务来决定应该使用哪个防火墙解决方案。

计划在安装时或第一次引导时添加防火墙解决方案的选择器。另一个解决方案的配置将保持不变，只需切换到其他模型即可启用。

firewalld （防火墙守护程序）独立于 system-config-firewall，但不应同时使用。

## 1.3 使用静态防火墙规则与 iptables 和 ip6tables 服务

----------

如果要使用自己的静态防火墙规则与 iptables 和 ip6tables 服务，安装 iptables-services 和禁用 firewalld 并启用 iptables 和 ip6tables：

```
dnf install iptables-services  # 安装 iptables-services  DNF 与 yum
systemctl mask firewalld.service  # 使 firewalld 服务失效，mask > disabled
systemctl enable iptables.service
systemctl enable ip6tables.service
```

使用 /etc/sysconfig/iptables 和 /etc/sysconfig/ip6tables 配置文件配置静态防火墙规则。

`注意`：iptables 和 iptables-services 包不提供用于服务的防火墙规则。这些服务可用于兼容性以及希望使用自己的防火墙规则的人员。您可以通过安装和使用 system-config-firewall 来创建用于服务的规则。要能够使用 system-config-firewall，你必须停止 firewalld。

在创建用于服务的规则后，停止 firewalld 并启动 iptables 和 ip6tables 服务：

```
systemctl stop firewalld.service
systemctl start iptables.service
systemctl start ip6tables.service
```

## 1.4 什么是区域

----------

网络区域定义网络连接的信任级别。这是一对多关系，这意味着连接只能是一个区域的一部分，但区域可以用于许多网络连接。

1.4.1 **预定义服务**（Predefined services）

是一个端口和/或协议条目的组合的服务。（可选）可以添加 netfilter 帮助模块，还可以添加 IPv4 和 IPv6 目标地址。

1.4.2 **端口和协议**（Ports and protocols）

tcp 或 udp 端口的定义，其中端口可以是单个端口或端口范围。 

1.4.3 **ICMP阻塞**（ICMP blocks）

可以选择 Internet 控制报文协议的报文。这些报文可以是信息请求亦可是对信息请求或错误条件创建的响应。

1.4.4 **伪装**（Masquerading）

专用（私有）网络的地址映射到公有 IP 地址并隐藏在公有 IP 地址后面。这是一种地址转换的形式。

1.4.5 **端口转发**（Forward ports）

端口可以映射到另一个端口以及/或者其他主机。

## 1.5 哪些区域可用

----------

以下是由 firewalld 提供的区域，根据从 **untrusted 区域**（不受信任）到 **trusted 区域**（受信任）的默认信任级别排序：

1.5.1 **丢弃**（drop）

任何传入的网络数据包（外来访问）都被丢弃，不作回应。只有传出的网络连接（对外访问）是可行的。

1.5.2 **阻塞**（block）

任何传入的网络连接（外来访问）都会被拒绝，其中包含针对 IPv4 的 icmp-host-prohibited 消息和针对 IPv6 的 icmp6-adm-prohibited 消息。只允许由该系统初始化的网络连接。

1.5.3 **公开**（public）

在公共场所使用。你认为网络中其他的计算机不可信并且可能伤害你的计算机。只接受选定的传入连接（允许被选定的外来访问）。

1.5.4 **外部**（external）

用在路由器等启用了伪装的外部网络。你认为网络中其他的计算机不可信并且可能伤害你的计算机。只允许选中的连接接入（过滤外来访问）。

1.5.5 **隔离区**（dmz）

用以允许隔离区（dmz）中的电脑有限地被外界网络访问。只接受被选中的连接。

1.5.6 **工作**（work）

用在工作网络。你信任网络中的大多数计算机不会影响你的计算机。只接受被选中的连接。

1.5.7 **家庭**（home）

用在家庭网络。你信任网络中的大多数计算机不会影响你的计算机。只接受被选中的连接。

1.5.8 **内部**（internal）

用在内部网络。你信任网络中的大多数计算机不会影响你的计算机。只接受被选中的连接。

1.5.9 **受信任的**（trusted）

允许所有网络连接。

## 1.6 应该选用哪个区域

----------

例如，公共的 WIFI 连接应该主要为不受信任的，家庭的有线网络应该是可信任的。根据与你使用的网络最符合的区域进行选择。

根据使用的网络连接受信任程度来选用。

## 1.7 如何配置或添加区域

----------

你可以使用任何一种 firewalld 配置工具来配置或者增加区域，以及修改配置。工具有例如 firewall-config 这样的图形界面工具，firewall-cmd 这样的命令行工具，以及 D-BUS 接口。或者你也可以在配置文件目录中创建或者拷贝区域文件。@PREFIX@/lib/firewalld/zones 被用于默认和备用配置，/etc/firewalld/zones 被用于用户创建和自定义配置文件。

## 1.8 如何设置或更改连接的区域

----------

区域设置以 ZONE= 选项 存储在网络连接的 ifcfg 文件中。如果这个选项缺失或者为空，firewalld 将使用配置的默认区域。

如果这个连接受到 NetworkManager 控制，你也可以使用 nm-connection-editor 来修改区域。

## 1.9 由 NetworkManager 控制的网络连接

防火墙不能够通过 NetworkManager 显示的名称来配置网络连接，只能配置网络接口。因此在网络连接之前 NetworkManager 将配置文件所述连接对应的网络接口告诉 firewalld 。如果在配置文件中没有配置区域，接口将配置到 firewalld 的默认区域。如果网络连接使用了不止一个接口，所有的接口都会应用到 fiwewalld。接口名称的改变也将由 NetworkManager 控制并应用到 firewalld。

为了简化，自此，网络连接将被用作与区域的关系。

如果一个接口断开了，NetworkManager 也将告诉 firewalld 从区域中删除该接口。

当 firewalld 由 systemd 或者 init 脚本启动或者重启后，firewalld 将通知 NetworkManager 把网络连接增加到区域。

## 1.10 由脚本控制的网络

----------

对于由网络脚本控制的连接有一条限制：没有守护进程通知 firewalld 将连接增加到区域。这项工作仅在 ifcfg-post 脚本进行。因此，此后对网络连接的重命名将不能被应用到 firewalld。同样，在连接活动时重启 firewalld   将导致与其失去关联。现在有意修复此情况。最简单的是将全部未配置连接加入默认区域。

区域定义了本区域中防火墙的特性。

## 1.11 使用 firewalld

----------

你可以通过图形界面工具 firewall-config 或者命令行客户端 firewall-cmd 启用或者关闭防火墙特性。

1.11.1 **使用firewall-cmd**

命令行工具 firewall-cmd 支持全部防火墙特性。对于状态和查询模式，命令只返回状态，没有其他输出。

1.11.1.1 **一般应用**

- 获取 firewalld 状态

```
firewall-cmd --state  # 获取 firewalld 状态
```

- 返回 firewalld 的状态。可以使用以下方式获得状态输出：

```
firewall-cmd --state && echo "Running" || echo "Not running"
```

- 在 Fedora 19 中, 状态输出比此前直观:

```
# rpm -qf $( which firewall-cmd )
 firewalld-0.3.3-2.fc19.noarch
# firewall-cmd --state
 not running
```

- 在不改变状态的条件下重新加载防火墙：

```
firewall-cmd --reload # 只加载防火墙区域配置，不重新加载内核模块
```

- 如果你使用 --complete-reload，状态信息将会丢失。这个选项应当仅用于处理防火墙问题时，例如，状态信息和防火墙规则都正常，但是不能建立任何连接的情况。

```
firewall-cmd --complet-reload  # 连内核模块一起重新加载
```

- 获取支持的区域列表

```
firewall-cmd --get-zones  # 获取区域列表，这条命令输出用空格分隔的列表。
```

- 获取所有支持的ICMP类型

```
firewall-cmd --get-icmptypes  # 这条命令输出用空格分隔的列表。
```

- 列出全部启用区域的特性

```
firewall-cmd --list-all-zones
```

- 输出指定区域 <zone> 全部启用的特性。如果忽略区域，将显示默认区域的信息

```
firewall-cmd [--zone=<zone>] --list-all  # 命令格式
firewall-cmd --list-all   # 忽略区域 zone
firewall-cmd --zone=work --list-all  # 显示 work 区域的信息
```

- 获取默认区域的网络设置

```
firewall-cmd --get-default-zone  # 获取默认区域的网络设置
```

- 设置默认区域

```
firewall-cmd --set-default-zone=<zone>  # 命令格式
firewall-cmd --set-default-zone=home  # 设置默认区域为 home
```
流入默认区域中配置的接口的新访问请求将被置入新的默认区域。当前活动的连接将不受影响。

- 获取活动的区域

```
firewall-cmd --get-active-zones 
```

- 根据接口获取区域

```
firewall-cmd --get-zone-of-interface=<interface>  # 命令格式
firewall-cmd --get-zone-of-interface=eno16777728  # 根据网卡名称获取区域
```
这条命令将输出接口所属的区域名称。

- 将接口增加到区域

```
firewall-cmd [--zone=<zone>] --add-interface=<interface>  # 命令格式
firewall-cmd --add-interface=eno16777745  # 向默认区域添加接口（网卡）
firewall-cmd --zone=work --add-interface=eno16777734  # 向 work 区域添加接口（网卡）
```
如果接口不属于区域，接口将被增加到区域。如果区域被省略了，将使用默认区域。接口在重新加载后将重新应用。

- 修改接口所属区域

```
firewall-cmd [--zone=<zone>] --change-interface=<interface> #命令格式
firewall-cmd --change-interface=eno16777745  # 把接口（网卡）切换到默认区域
firewall-cmd --zone=work --change-interface=eno1677735 # 把接口切换到指定的 work 区域
```
这个选项与 --add-interface 选项相似，但是当接口已经存在于另一个区域的时候，该接口仍将被添加到新的区域。

- 从区域中删除一个接口

```
firewall-cmd [--zone=<zone>] --remove-interface=<interface> # 命令格式
firewall-cmd --remove-interface=eno16777745  # 从默认区域移除
firewall-cmd --zone=work --remove-interface=eno16777745 # 从指定区域移除
```

- 查询区域中是否包含某接口

```
firewall-cmd [--zone=<zone>] --query-interface=<interface>
```
返回接口是否存在于该区域。

- 列举区域中启用的服务

```
firewall-cmd [ --zone=<zone> ] --list-services
```

- 出现紧急状况，启用应急模式阻断所有网络连接

```
firewall-cmd --panic-on 
```

- 禁用应急模式

```
firewall-cmd --panic-off
```
在 0.3.0 之前的 Firewalld 版本中, panic 选项是 –enable-panic 与 –disable-panic.

- 查询应急模式

```
firewall-cmd --query-panic
firewall-cmd --query-panic && echo "On" || echo "Off"
```

## 1.11.1.2 处理运行中的区域 （runtime）

----------

运行时模式下对区域进行的修改不是永久有效的。重新加载或者重启后修改将失效。

- 启用区域中的一种服务

```
firewall-cmd [--zone=<zone>] --add-service=<service> [--timeout=<seconds>]
```
此举启用区域中的一种服务。如果未指定区域，将使用默认区域。如果设定了超时时间，服务将只启用特定秒数。如果服务已经活跃，将不会有任何警告信息。

- 例: 使区域中的 ipp-client 服务生效60秒:

```
firewall-cmd --zone=home --add-service=ipp-client --timeout=60
```

- 例: 启用默认区域中的http服务:

```
firewall-cmd --add-service=http
```

- 禁用区域中的某种服务

```
firewall-cmd [--zone=<zone>] --remove-service=<service>
```
此举禁用区域中的某种服务。如果未指定区域，将使用默认区域。

- 例: 禁止home区域中的http服务:

```
firewall-cmd --zone=home --remove-service=http
```
区域种的服务将被禁用。如果服务没有启用，将会有警告信息。

- 查询区域中是否启用了特定服务

```
firewall-cmd [--zone=<zone>] --query-service=<service>
```

- 启用区域端口和协议组合

```
firewall-cmd [--zone=<zone>] --add-port=<port>[-<port>]/<protocol> [--timeout=<seconds>]
```
此举将启用端口和协议的组合。端口可以是一个单独的端口 <port> 或者是一个端口范围 <port>-<port> 。协议可以是 tcp 或 udp。

- 禁用端口和协议组合

```
firewall-cmd [--zone=<zone>] --remove-port=<port>[-<port>]/<protocol>
```

- 查询区域中是否启用了端口和协议组合

```
firewall-cmd [--zone=<zone>] --query-port=<port>[-<port>]/<protocol>
```

- 启用区域中的 IP 伪装功能

```
firewall-cmd [--zone=<zone>] --add-masquerade
```
此举启用区域的伪装功能。私有网络的地址将被隐藏并映射到一个公有IP。这是地址转换的一种形式，常用于路由。由于内核的限制，伪装功能仅可用于IPv4。

- 禁用区域中的IP伪装

```
firewall-cmd [--zone=<zone>] --remove-masquerade
```

- 查询区域的伪装状态

```
firewall-cmd [--zone=<zone>] --query-masquerade
```

- 启用区域的 ICMP 阻塞功能

```
firewall-cmd [--zone=<zone>] --add-icmp-block=<icmptype>
```
此举将启用选中的Internet控制报文协议（ICMP）报文进行阻塞。ICMP报文可以是请求信息或者创建的应答报文，以及错误应答。

- 禁止区域的 ICMP 阻塞功能

```
irewall-cmd [--zone=<zone>] --remove-icmp-block=<icmptype>
```

- 查询区域的 ICMP 阻塞功能

```
firewall-cmd [--zone=<zone>] --query-icmp-block=<icmptype>
```

- 例: 阻塞区域的响应应答报文:

```
firewall-cmd --zone=public --add-icmp-block=echo-reply
```

- 在区域中启用端口转发或映射

```
firewall-cmd [--zone=<zone>] --add-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
```
端口可以映射到另一台主机的同一端口，也可以是同一主机或另一主机的不同端口。端口号可以是一个单独的端口 <port> 或者是端口范围 <port>-<port> 。协议可以为 tcp 或udp 。目标端口可以是端口号 <port> 或者是端口范围 <port>-<port> 。目标地址可以是 IPv4 地址。受内核限制，端口转发功能仅可用于IPv4。

- 禁止区域的端口转发或者端口映射

```
firewall-cmd [--zone=<zone>] --remove-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
```

- 查询区域的端口转发或者端口映射

```
firewall-cmd [--zone=<zone>] --query-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
```

- 例: 将区域 home 的 ssh 转发到 127.0.0.2

```
firewall-cmd --zone=home --add-forward-port=port=22:proto=tcp:toaddr=127.0.0.2
```

- 例：将 8888 端口转发到 22 端口

```
firewall-cmd --zone=public --add-forward-prot=port=8888:proto=tcp:toport=22
```

## 1.11.1.3 处理永久区域

----------

永久选项不直接影响运行时的状态。这些选项仅在重载或者重启服务时可用。为了使用运行时和永久设置，需要分别设置两者。 选项 --permanent 需要是永久设置的第一个参数。

- 查看支持永久生效的服务

```
firewall-cmd --permanent --get-services
```

- 查看支持永久生效的 ICMP 类型列表

```
firewall-cmd --permanent --get-icmptypes
```

- 查看支持永久生效的区域

```
firewall-cmd --permanent --get-zones
```

- 在永久生效区域中添加服务

```
firewall-cmd --permanent [--zone=<zone>] --add-service=<service>
```
此举将永久启用区域中的服务。如果未指定区域，将使用默认区域

- 禁用区域中的一种服务

```
firewall-cmd --permanent [--zone=<zone>] --remove-service=<service>
```

- 查询区域中的服务是否启用

```
firewall-cmd --permanent [--zone=<zone>] --query-service=<service>
```

- 例: 永久启用 home 区域中的 ipp-client 服务

```
firewall-cmd --permanent --zone=home --add-service=ipp-client
```

- 永久启用区域中的一个端口-协议组合

```
firewall-cmd --permanent [--zone=<zone>] --add-port=<port>[-<port>]/<protocol>
```

- 永久禁用区域中的一个端口-协议组合

```
firewall-cmd --permanent [--zone=<zone>] --remove-port=<port>[-<port>]/<protocol>
```

- 查询区域中的端口-协议组合是否永久启用

```
firewall-cmd --permanent [--zone=<zone>] --query-port=<port>[-<port>]/<protocol>
```

- 例: 永久启用/禁用 home 区域中的 https (tcp 443) 端口

```
firewall-cmd --permanent --zone=home --add-port=443/tcp
firewall-cmd --permanent --zone=home --remove-port=443/tcp
```

- 永久启用区域中的伪装

```
firewall-cmd --permanent [--zone=<zone>] --add-masquerade
```
此举启用区域的伪装功能。私有网络的地址将被隐藏并映射到一个公有 IP。这是地址转换的一种形式，常用于路由。由于内核的限制，伪装功能仅可用于 IPv4。

- 永久禁用区域中的伪装

```
firewall-cmd --permanent [--zone=<zone>] --remove-masquerade
```

- 查询区域中的伪装的永久状态

```
firewall-cmd --permanent [--zone=<zone>] --query-masquerade
```

- 永久启用区域中的 ICMP 阻塞

```
firewall-cmd --permanent [--zone=<zone>] --add-icmp-block=<icmptype>
```
此举将启用选中的 Internet 控制报文协议 （ICMP） 报文进行阻塞。 ICMP 报文可以是请求信息或者创建的应答报文或错误应答报文。

- 永久禁用区域中的 ICMP 阻塞

```
firewall-cmd --permanent [--zone=<zone>] --remove-icmp-block=<icmptype>
```

- 查询区域中的ICMP永久状态

```
firewall-cmd --permanent [--zone=<zone>] --query-icmp-block=<icmptype>
```

- 例: 阻塞公共区域中的响应应答报文:

```
firewall-cmd --permanent --zone=public --add-icmp-block=echo-reply
```

- 在区域中永久启用端口转发或映射

```
firewall-cmd --permanent [--zone=<zone>] --add-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
```
端口可以映射到另一台主机的同一端口，也可以是同一主机或另一主机的不同端口。端口号可以是一个单独的端口 <port> 或者是端口范围 <port>-<port> 。协议可以为 tcp 或udp 。目标端口可以是端口号 <port> 或者是端口范围 <port>-<port> 。目标地址可以是 IPv4 地址。受内核限制，端口转发功能仅可用于IPv4。

- 永久禁止区域的端口转发或者端口映射

```
firewall-cmd --permanent [--zone=<zone>] --remove-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
```

- 查询区域的端口转发或者端口映射状态

```
firewall-cmd --permanent [--zone=<zone>] --query-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
```

- 例: 将 home 区域的 ssh 服务永久转发到 127.0.0.2

```
firewall-cmd --permanent --zone=home --add-forward-port=port=22:proto=tcp:toaddr=127.0.0.2
```

## 1.11.1.4 直接选项

----------

直接选项可以更直接地访问防火墙。这些选项要求用户知道基本的iptables概念，即表（filter / mangle / nat / ...），链（INPUT / OUTPUT / FORWARD / ...），命令（-A / -D / -I / .. 。），参数（-p / -s / -d / -j / ...）和目标（ACCEPT / DROP / REJECT / ...）。直接选项只能作为最后的手段，当不可能使用例如--add-service = service或--add-rich-rule ='rule'。每个选项的第一个参数必须是 ipv4 或 ipv6 或 eb。使用 ipv4 它将为 IPv4（iptables（8）），与 ipv6 IPv6（ip6tables（8））和 eb 以太网网桥（ebtables（8））。

- 将命令传递给防火墙。
参数 <args> 可以是 iptables, ip6tables 以及 ebtables 的命令行参数。

```
firewall-cmd --direct --passthrough {ipv4 | ipv6 | eb} <args>
```

- 向表 table 中添加新链 chain。

```
firewall-cmd [--permanent] --direct --add-chain {ipv4 | ipv6 | eb} <table> <chain>
```

- 从表 table 中删除名为 chain 的链。

```
firewall-cmd [--permanent] --direct --remove-chain {ipv4 | ipv6 | eb} <table> <chain>
```

- 查询表 tabel 中是否存在名为 chain 的链

```
firewall-cmd [--permanent] --direct --query-chain {ipv4 | ipv6 | eb} <table> <chain>
```

- 将所有链添加到表 table 中作为空格分隔列表。

```
firewall-cmd [--permanent] --direct --get-chains {ipv4 | ipv6 | eb} <table>
```

- 使用优先级 priority 向表 table 中的 chain  添加参数 args 的规则。

```
firewall-cmd [--permanent] --direct --add-rule {ipv4 | ipv6 | eb} <table> <chain> <priority> <args>
```

- 从表 table 中的链 chain 中删除参数 args 的规则

```
firewall-cmd [--permanent] --direct --remove-rule {ipv4 | ipv6 | eb} <table> <chain> <args>
```

- 查询是否在表 table 中的链 chain 中存在参数 args 的规则。

```
firewall-cmd [--permanent] --direct --query-rule {ipv4 | ipv6 | eb} <table> <chain> <args>
```

- 将所有规则添加到表 table 中的 chain  作为换行符分隔的参数列表。

```
firewall-cmd [--permanent] --direct --get-rules {ipv4 | ipv6 | eb} <table> <chain>
```

## 1.12 当前 firewalld 的功能

----------

1.12.1 **D-BUS接口**（D-BUS Interface）

D-BUS 接口提供防火墙状态的信息，使防火墙的启用、停用或查询设置成为可能。

1.12.2 **区域**（Zones)

网络或者防火墙区域定义了连接的可信程度。firewalld 提供了几种预定义的区域。区域配置选项和通用配置信息可以在firewall.zone(5)的手册里查到。


1.12.3 **服务**(Services）

服务可以是一系列本地端口、目的地以及附加信息，也可以是服务启动时自动增加的防火墙助手模块。预定义服务的使用使启用和禁用对服务的访问变得更加简单。服务配置选项和通用文件信息在 firewalld.service(5) 手册里有描述。

1.12.4 **ICMP类型**（ICMP types）

因特网控制消息协议（ICMP）用于在因特网协议（IP）中交换信息以及错误消息。ICMP类型可以在firewalld中使用，以限制这些消息的交换。ICMP类型配置选项和通用文件信息在firewalld.icmptype（5）手册页中描述。

1.12.5 **直接接口**（Direct interface）

直接接口主要由服务或应用程序用于添加特定的防火墙规则。

1.12.6 **运行时配置**（Runtime configuration）

运行时配置不是永久性的，只会在重新加载时恢复。在重新启动或停止服务或系统重新启动后，这些选项将会消失。

1.12.7 **永久配置**（Permanent configuration）

永久配置存储在配置文件中，并将在每次机器引导或服务重新加载或重新启动时恢复。


1.12.8 **托盘小程序**（Tray Applet）

托盘小程序 firewall-applet 为用户可视化的显示防火墙状态和存在的问题。它也可以用于通过调用 firewall-config 配置设置。

1.12.9 **图形配置工具**（Graphical Configuration Tool）

配置工具 firewall-config 是防火墙守护程序的主要配置工具。它支持除直接接口之外的所有防火墙功能，这由添加规则的服务/应用程序处理。

1.12.10 **命令行客户端**（Command Line client）

命令行客户机firewall-cmd支持所有防火墙功能。对于状态和查询模式，没有输出，但命令返回状态。

对于离线使用，还有 firewall-offline-cmd。此命令行客户端正在直接创建 firewalld 配置文件，并且不使用firewalld 或 D-Bus 接口。例如，它在系统安装过程中用于从 kickstart 设置创建初始防火墙配置。

1.12.11 **对于ebtables的支持**（Support for ebtables）

要满足libvirt daemon的全部需求，在内核 netfilter 级上防止 ip*tables 和 ebtables 间访问问题，ebtables 支持是需要的。由于这些命令是访问相同结构的，因而不能同时使用

1.12.12 **/usr/lib/firewalld中的默认/备用配置**（Default/Fallback configuration in /usr/lib/firewalld）

该目录包含了由 firewalld 提供的默认以及备用的 ICMP 类型、服务、区域配置。由 firewalld 软件包提供的这些文件不能被修改，即使修改也会随着 firewalld 软件包的更新被重置。 其他的 ICMP 类型、服务、区域配置可以通过软件包或者创建文件的方式提供。

1.12.13 **/etc/firewalld中的系统配置设置**（System configuration settings in /etc/firewalld）

存储在此的系统或者用户配置文件可以是系统管理员通过配置接口定制的，也可以是手动定制的。这些文件将重载默认配置文件。

为了手动修改预定义的 icmp 类型，区域或者服务，从默认配置目录将配置拷贝到相应的系统配置目录，然后根据需求进行修改。

如果你加载了有默认和备用配置的区域，在 /etc/firewalld 下的对应文件将被重命名为 file.old 然后启用备用配置。

## 1.13 正在开发的特性 

----------

1.13.1 **富语言**（Rich Language）

丰富的语言提供了一种高级语言，能够为 IPv4 和 IPv6 提供更复杂的防火墙规则，而无需了解 iptables 语法。

Fedora 19 提供了带有 D-Bus 和命令行支持的富语言特性第2个里程碑版本。第3个里程碑版本也将提供对于图形界面 firewall-config 的支持。

对于此特性的更多信息，请参阅： firewalld Rich Language

1.13.2 **锁定**（Lockdown）

锁定特性为 firewalld 增加了锁定本地应用或者服务配置的简单配置方式。它是一种轻量级的应用程序策略。

Fedora 19 提供了锁定特性的第二个里程碑版本，带有 D-Bus 和命令行支持。第3个里程碑版本也将提供图形界面 firewall-config 下的支持。

更多信息请参阅： firewalld Lockdown

1.13.3 **永久直接规则**（Permanent Direct Rules）

这项特性处于早期状态。它将能够提供保存直接规则和直接链的功能。通过规则不属于该特性。更多关于直接规则的信息请参阅Direct options。

1.13.4 **从ip *tables 和 ebtables 服务迁移**（Migration from ip*tables and ebtables services）

此功能处于非常早的状态。它将提供一个转换脚本，尽可能从iptables，ip6tables和ebtables服务配置创建直接永久规则。这里的限制可能是集成到firewalld提供的直接链中。
这也需要从更复杂的防火墙配置进行大量测试。

## 1.14 计划和建议的功能

----------

1.14.1 **防火墙抽象模型**（Firewall Abstraction Model）

在 ip*tables 和 ebtables 防火墙规则之上添加抽象层使添加规则更简单和直观。要抽象层功能强大，但同时又不能复杂，并不是一项简单的任务。为此，不得不开发一种防火墙语言。使防火墙规则拥有固定的位置，可以查询端口的访问状态、访问策略等普通信息和一些其他可能的防火墙特性。

1.14.2 **支持conntrack**（Support for conntrack）

要终止禁用特性已确立的连接需要 conntrack 。不过，一些情况下终止连接可能是不好的，如：为建立有限时间内的连续性外部连接而启用的防火墙服务。

1.14.3 **用户交互模式**（User interaction mode）

这是防火墙中用户或者管理员可以启用的一种特殊模式。应用程序所有要更改防火墙的请求将定向给用户知晓，以便确认和否认。为一个连接的授权设置一个时间限制并限制其所连主机、网络或连接是可行的。配置可以保存以便将来不需通知便可应用相同行为。 该模式的另一个特性是管理和应用程序发起的请求具有相同功能的预选服务和端口的外部链接尝试。服务和端口的限制也会限制发送给用户的请求数量。


1.14.4 **用户策略支持**（User policy support）

管理员可以规定哪些用户可以使用用户交互模式和限制防火墙可用特性。

1.14.5 **端口元数据信息(由 Lennart Poettering 提议)**（Port metadata information (proposed by Lennart Poettering)）

拥有一个端口独立的元数据信息是很好的。当前对 /etc/services 的端口和协议静态分配模型不是个好的解决方案，也没有反映当前使用情况。应用程序或服务的端口是动态的，因而端口本身并不能描述使用情况。

此元数据信息可用于为防火墙形成简单的规则。这里有些例子：


>  允许外部访问文件共享应用程序或服务
 允许外部访问音乐共享应用程序或服务
 允许外部访问所有共享应用程序或服务
 允许外部访问torrent文件共享应用程序或服务
 允许外部访问http web服务

这里的元数据信息不只有特定应用程序，还可以是一组使用情况。例如：组“全部共享”或者组“文件共享”可以对应于全部共享或文件共享程序(如：torrent 文件共享)。这些只是例子，因而，可能并没有实际用处。

这里是在防火墙中获取元数据信息的两种可能途径：


> 第一种是添加到 netfilter (内核空间)。好处是每个人都可以使用它，但也有一定使用限制。还要考虑用户或系统空间的具体信息，所有这些都需要在内核层面实现。

> 第二种是添加到 firewall daemon 中。这些抽象的规则可以和具体信息(如：网络连接可信级、作为具体个人/主机要分享的用户描述、管理员禁止完全共享的应归则等)一起使用。

第二种解决方案的好处是不需要为有新的元数据组和纳入改变(可信级、用户偏好或管理员规则等等)重新编译内核。这些抽象规则的添加使得 firewall daemon 更加自由。即使是新的安全级也不需要更新内核即可轻松添加。



1.14.6 **sysctld**

现在仍有 sysctl 设置没有正确应用。一个例子是，在 rc.sysinit 正运行时，而提供设置的模块在启动时没有装载或者重新装载该模块时会发生问题。

另一个例子是 net.ipv4.ip_forward ，防火墙设置、libvirt 和用户/管理员更改都需要它。如果有两个应用程序或守护进程只在需要时开启 ip_forwarding ，之后可能其中一个在不知道的情况下关掉服务，而另一个正需要它，此时就不得不重启它。

sysctl daemon 可以通过对设置使用内部计数来解决上面的问题。此时，当之前请求者不再需要时，它就会再次回到之前的设置状态或者是直接关闭它。


1.15 防火墙规则

----------

netfilter 防火墙总是容易受到规则顺序的影响，因为一条规则在链中没有固定的位置。在一条规则之前添加或者删除规则都会改变此规则的位置。 在静态防火墙模型中，改变防火墙就是重建一个干净和完善的防火墙设置，且受限于 system-config-firewall / lokkit 直接支持的功能。也没有整合其他应用程序创建防火墙规则，且如果自定义规则文件功能没在使用 s-c-fw / lokkit 就不知道它们。默认链通常也没有安全的方式添加或删除规则而不影响其他规则。

动态防火墙有附加的防火墙功能链。这些特殊的链按照已定义的顺序进行调用，因而向链中添加规则将不会干扰先前调用的拒绝和丢弃规则。从而利于创建更为合理完善的防火墙配置。

下面是一些由守护进程创建的规则，过滤列表中启用了在公共区域对 ssh , mdns 和 ipp-client 的支持：

```
*filter
 :INPUT ACCEPT [0:0]
 :FORWARD ACCEPT [0:0]
 :OUTPUT ACCEPT [0:0]
 :FORWARD_ZONES - [0:0]
 :FORWARD_direct - [0:0]
 :INPUT_ZONES - [0:0]
 :INPUT_direct - [0:0]
 :IN_ZONE_public - [0:0]
 :IN_ZONE_public_allow - [0:0]
 :IN_ZONE_public_deny - [0:0]
 :OUTPUT_direct - [0:0]
 -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
 -A INPUT -i lo -j ACCEPT
 -A INPUT -j INPUT_direct
 -A INPUT -j INPUT_ZONES
 -A INPUT -p icmp -j ACCEPT
 -A INPUT -j REJECT --reject-with icmp-host-prohibited
 -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
 -A FORWARD -i lo -j ACCEPT
 -A FORWARD -j FORWARD_direct
 -A FORWARD -j FORWARD_ZONES
 -A FORWARD -p icmp -j ACCEPT
 -A FORWARD -j REJECT --reject-with icmp-host-prohibited
 -A OUTPUT -j OUTPUT_direct
 -A IN_ZONE_public -j IN_ZONE_public_deny
 -A IN_ZONE_public -j IN_ZONE_public_allow
 -A IN_ZONE_public_allow -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
 -A IN_ZONE_public_allow -d 224.0.0.251/32 -p udp -m udp --dport 5353 -m conntrack --ctstate NEW -j ACCEPT
 -A IN_ZONE_public_allow -p udp -m udp --dport 631 -m conntrack --ctstate NEW -j ACCEPT
```

使用 deny/allow 模型来构建一个清晰行为(最好没有冲突规则)。例如： ICMP块将进入 IN_ZONE_public_deny 链(如果为公共区域设置了的话)，并将在 IN_ZONE_public_allow 链之前处理。

该模型使得在不干扰其他块的情况下向一个具体块添加或删除规则而变得更加容易。

----------

2017-2-27 by Achkux

























