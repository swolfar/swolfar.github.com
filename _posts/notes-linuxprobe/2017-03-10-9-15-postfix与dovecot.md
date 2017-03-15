---
layout: post
title: "使用 postfix 与 dovecot 收发电子邮件"
date: 2017-3-10
description: 使用 postfix 与 dovecot 收发电子邮件
tags: 
 - linux
 - RHCE
 - postfix
 - dovecot 
---

# 使用 postfix 与 dovecot 收发电子邮件

----------

**简述**
- 电子邮局系统的组成角色
- MUA、MTA 与 MDA 的作用
- SMTP、POP3 与 IMAP4 邮局协议
- postfix 与 dovecot 服务程序的使用
- 部署基础电子邮局系统
- 设置用户别名邮箱

----------

## 电子邮局系统

邮件应用协议包括:

> 简单邮件传输协议(SMTP,Simple Mail Transfer Protocol)，用来发送或中转发出的电子邮件，占用 tcp 25 端口。
> 
> 第三版邮局协议(POP3,Post Office Protocol 3)，用于将服务器上把邮件存储到本地主机，占用 tcp 110 端口。
> 
> 第四版互联网信息访问协议(IMAP4,Internet Mail Access Protocol)，用于在本地主机上访问邮件，占用 tcp 143 端口。

**SMTP**

> SMTP 的全称是 “Simple Mail Transfer Protocol”，即简单邮件传输协议。它是一组用于从源地址到目的地址传输邮件的规范，通过它来控制邮件的中转方式。SMTP 协议属于 TCP/IP 协议簇，它帮助每台计算机在发送或中转信件时找到下一个目的地。SMTP 服务器就是遵循 SMTP 协议的发送邮件服务器。

**POP3**

> POP3 是 Post Office Protocol 3 的简称，即邮局协议的第 3 个版本,它规定怎样将个人计算机连接到 Internet 的邮件服务器和下载电子邮件的电子协议。它是因特网电子邮件的第一个离线协议标准,POP3 允许用户从服务器上把邮件存储到本地主机（即自己的计算机）上,同时删除保存在邮件服务器上的邮件，而 POP3 服务器则是遵循 POP3 协议的接收邮件服务器，用来接收电子邮件的。

**IMAP**

> IMAP 全称是 Internet Mail Access Protocol，即交互式邮件存取协议，它是跟 POP3 类似邮件访问标准协议之一。不同的是，开启了 IMAP 后，您在电子邮件客户端收取的邮件仍然保留在服务器上，同时在客户端上的操作都会反馈到服务器上，如：删除邮件，标记已读等，服务器上的邮件也会做相应的动作。所以无论从浏览器登录邮箱或者客户端软件登录邮箱，看到的邮件以及状态都是一致的。

![](http://i.imgur.com/dscK9Nn.png)

电子邮件系统(E-mail，即Electronic mail system)由三部分组成：

> 用户代理 MUA(Mail User Agent): 用于收发邮件。
> 
> 邮件传输代理 MTA(Mail Transfer Agent): 将来自于 MUA 的邮件转发给指定用户。
> 
> 邮件投递代理 MDA(Mail Delivery Agent): 将来自于 MTA 的邮件保存到本机的收件箱中。

![](http://i.imgur.com/OdW0LNe.png)

搭建企业级的电子邮件系统，请考虑下面几点：

> 反垃圾与反病毒模块: 阻止垃圾邮件或病毒邮件对企业邮箱的干扰。
> 邮件加密: 保证邮件内容不被嗅探、篡改。
> 邮件监控审核: 监控全体职员邮件中有无敏感词，透露企业资料等。
> 稳定性: 有较好的防 DDOS 攻击的能力，保证系统在线率等。

----------

## 部署基础电子邮局系统

一个基础的电子邮局系统至少需要有 SMTP 服务器、POP3/IMAP 服务器

需要使用到的软件：

> Postfix: 提供邮件发送服务，即 SMTP。
> 
> Dovecot: 提供邮件收取服务，即 POP3。
> 
> OutLook Express: 客户端收发邮件的工具。

![](http://i.imgur.com/fQhsBZF.png)

步骤：
1. 配置 DNS、主机名称
2. 配置 Postfix 服务程序
 - 停止 iptables 防火墙
 - 安装 postfix 邮局服务程序
 - 查看 Postfix 服务程序主配置文件：/etc/postfix/main.cf
 - 创建邮局帐号
 - 启动 Postfix 服务程序
3. 配置 Dovecot 服务程序
 - 安装 Dovecot 服务程序
 - 修改 Dovecot 程序主配置文件
 - 配置邮件的格式与存储路径
 - 创建邮件的存储目录
 - 启动 Dovecot 服务程序
4. 用户使用邮局系统

- 配置本地主机名

```
vim /etc/hostname
-----------
mail.linuxprobe.com
```

若要为用户提供 linuxprobe 域的电子邮局系统，则需先在 DNS 服务器中增加 A 记录和 MX 记录

```
@     IN MX 10 mail.linuxprobe.com.
mail  IN A     192.168.10.10
```

- 配置 Postfix 服务程序

停止 iptables 防火墙

```
systemctl disable iptables
```

安装 postfix 邮局服务程序

```
yum install postfix
```

Postfix 邮局服务程序的配置文件如下

文件|	作用
----|-----
/usr/sbin/postfix	|主服务程序
/etc/postfix/master.cf	|master主程序的配置文件。
/etc/postfix/main.cf	|postfix服务的配置文件。
/var/log/maillog	|记录邮件传递过程的日志。

第1步:查看 Postfix 服务程序主配置文件

参数	|作用
-----|-----
myhostname	|邮局系统的主机名。
mydomain	|邮局系统的域名。
myorigin	|从本机寄出邮件的域名名称。
inet_interfaces	|监听的网卡接口。
mydestination	|可接收邮件的主机名或域名。
mynetworks	|设置可转发那些主机的邮件。
relay_domains	|设置可转发那些网域的邮件

编辑 Postfix 服务程序的主配置文件

```
vim /etc/postfix/main.cf
-------------
# 修改第 76 行的邮局主机名。
myhostname = mail.linuxprobe.com

# 修改第 83 行的邮局域名。
mydomain = linuxprobe.com

# 修改第 99 行的寄出邮件域名，$mydomain 的值已在上面定义。
myorigin = $mydomain

# 修改第 116 行的监听网卡。
inet_interfaces = all

# 修改第 164 行的可接收邮件的主机名和域名。
mydestination = $myhostname, $mydomain
```

第2步:创建邮局帐号

```
useradd boss
echo "linuxprobe" | passwd --stdin boss
```

第3步:启动Postfix服务程序

-  配置 Dovecot 服务程序

第1步:安装 Dovecot 服务程序

```
yum install dovecot -y
```

第2步:修改 Dovecot 程序主配置文件

```
vim /etc/dovecot/dovecot.conf
---------------------
# 修改第24行的支持邮局协议。
protocols = imap pop3 lmtp

# 然后追加允许明文认证（25行）。
disable_plaintext_auth = no

# 修改第48行的允许登录网段地址，全部允许即为（0.0.0.0/0）。
login_trusted_networks = 192.168.10.0/24
```

第3步:配置邮件的格式与存储路径

编辑 dovecot 的配置文件：/etc/dovecot/conf.d/10-mail.conf

```
vim /etc/dovecot/conf.d/10-mail.conf
------------------
mail_location = mbox:~/mail:INBOX=/var/mail/%u
```

第4步:创建邮件的存储目录

```
su - boss
mkdir -p mail/.imap/INBOX # 在家目录创建
```

第5步:启动Dovecot服务程序

- 用户使用邮局系统

----------

##  设置用户别名邮箱

**aliases** 别名机制

查看 aliases 别名机制的配置文件：`cat /etc/aliases`

如发送给 xxoo@linuxprobe.com 的邮件，均保存到 root@linuxprobe.com 的邮箱中，则添加：

xxoo: root

编辑 /etc/aliases 文件后需要执行命令 "`newaliases`"
----------

## 本章命令汇总

`mail` 命令：查看和发送邮件

`newaliases` 命令：更新邮件用户别名

----------

## 作业

1:邮件应用协议包括：

答案:SMTP、POP3 和 IMAP4。

2:简述 MUA、MTA、MDA 的用处：

答案:MUA 用于收发邮件、MTA 用于转发邮件、MDA 用于保存邮件。

3:如何在 dovecot 程序的配置文件中限制允许连接的主机？

答案:修改 login_trusted_networks 参数的值。

4:使用 outlook 连接后提示找不到服务器或连接超时，最可能是什么原因？

答案:极大可能是 DNS 问题，请 ping mail.linuxprobe.com。

5:定义邮件用户别名邮箱后让其立即生效？

答案:执行 newaliases 命令。

----------

2017-03-10 by Achxku














































