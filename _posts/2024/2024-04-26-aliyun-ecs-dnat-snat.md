---
layout: post
title: "简易跳板机连接无外网IP的ECS及其访问外网"
categories: Linux
tags: Linux ECS VPS 跳板机 SSH Tunnel
author: 玄玉
excerpt: 针对无外网IP的ECS，介绍本地SSH连接的方法，以及它访问外网的方法等。
published: true
---

* content
  {:toc}


背景：阿里云上有 2 台 CentOS7.9，一台分配了公网IP（192.168.1.1），另一台无公网IP（192.168.0.1）

## 简易跳板机

先在阿里云 192.168.0.1 机器上配置安全组：22 端口允许 192.168.1.1 访问

然后在本地 xshell 作如下 2 步配置即可：

1. 创建一个跳板机的 Session（可以随便给一个名字，这里就命名为跳板机），具体配置如图所示<br/>
   ![](https://gcore.jsdelivr.net/gh/jadyer/mydata/img/blog/2024/2024-04-26-aliyun-ecs-dnat-snat-01.png)<br/><br/>
   ![](https://gcore.jsdelivr.net/gh/jadyer/mydata/img/blog/2024/2024-04-26-aliyun-ecs-dnat-snat-02.png)<br/>
   最重要的就是：隧道的配置，这里是通过 192.168.1.1 机器侦听 2200 端口，来与 192.168.0.1 机器的 22 端口建立隧道
2. 创建一个内网机器的 Session<br/>
   ![](https://gcore.jsdelivr.net/gh/jadyer/mydata/img/blog/2024/2024-04-26-aliyun-ecs-dnat-snat-03.png)<br/>
   然后在 Authentication 处填入连接内网机器的用户名密码或 PublicKey 即可

实际使用时，通过 xshell 先连跳板机，再连内网机器，就行了（所以上面建了 2 个 xshell session）

关于这方面的更详细介绍，可参考这篇博客：[xshell tunnel的使用](https://www.cnblogs.com/oxspirt/p/10260053.html)

## 配置访问外网

192.168.0.1 默认是无法访问外网的

这里可以借助有公网的 192.168.1.1 来访问外网（二者处于同一个 VPC，但可以不在一个安全组内）

共有 3 步配置 **（全是在 192.168.1.1 上面配置，192.168.0.1 全程不需要任何配置）**：

1. 开启 ECS 的 IP 转发功能<br/>
   ```shell
   [root@dev02 ~]# vim /etc/sysctl.conf
   net.ipv4.ip_forward = 1             # 添加这条配置（如果已有，则直接修改）
   [root@dev02 ~]# sysctl -p           # 令修改生效
   ```
2. 设置 SNAT 规则：分别执行以下命令<br/>
   执行命令：`iptables -t nat -I POSTROUTING -s 192.168.0.0/24 -j SNAT --to-source 192.168.1.1`<br/>
   再把它保存在 iptables 配置文件中：`service iptables save`（否则 iptables 规则重启会清空）<br/>
   这个时候，查看 **/etc/sysconfig/iptables** 会发现它多了一条 SNAT 转发规则<br/>
   *-A POSTROUTING -s 192.168.0.0/24 -j SNAT --to-source 192.168.1.1*<br/>
   最后重启 iptables 使规则生效：`systemctl restart iptables.service`
3. 在阿里云后台自定义路由条目：<https://vpc.console.aliyun.com/vpc/cn-beijing/route-tables><br/>
   ![](https://gcore.jsdelivr.net/gh/jadyer/mydata/img/blog/2024/2024-04-26-aliyun-ecs-dnat-snat-03.png)<br/>
   名称可以随便写，除了图中圈红的两处，两台服务器还要位于同一个资源组下面，否则 ECS 实例处选择不到

现在，192.168.0.1 就可以访问外网了，同 192.168.1.1 相比，速度几乎无影响

```shell
# 这是在跳板机上 ping 的结果
[xuanyu@dev02 ~]$ ping www.baidu.com
PING www.a.shifen.com (220.181.38.149) 56(84) bytes of data.
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=1 ttl=54 time=5.09 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=2 ttl=54 time=5.08 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=3 ttl=54 time=5.11 ms

# 这是在无外网IP的机器上 ping 的结果
[xuanyu@prod01 ~]$ ping www.baidu.com
PING www.a.shifen.com (220.181.38.149) 56(84) bytes of data.
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=1 ttl=53 time=6.75 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=2 ttl=53 time=6.73 ms
64 bytes from 220.181.38.149 (220.181.38.149): icmp_seq=3 ttl=53 time=6.71 ms
```

### 可能遇到的问题

内网机器在 ping 外网时，可能会返回 icmp_seq=1 Destination Host Prohibited，而非 icmp_seq=1 ttl=53 time=6.75 ms

此时，需要修改 192.168.1.1：`vim /etc/sysconfig/iptables`（不用修改内网机器 192.168.0.1）

注释掉：**-A FORWARD -j REJECT --reject-with icmp-host-prohibited**

再重启：`systemctl restart iptables.service` 即可

### 关于新版防火墙

CentOS 7.x 开始默认使用的是 firewall 作为防火墙，我们需要把它改为 iptables 防火墙

注意：firewalld 和 iptables 的配置和规则都有所不同，二者不能同时启动

```shell
# 关闭 firewall
systemctl stop firewalld.service
systemctl disable firewalld.service
systemctl mask firewalld.service
# 安装 iptables
yum install -y iptables-services
# 启动 iptables
systemctl enable iptables
systemctl start iptables
# 查看防火墙状态
systemctl status iptables

# 设置防火墙开机启动
systemctl enable iptables.service
# 重启防火墙使配置生效
systemctl restart iptables.service
```