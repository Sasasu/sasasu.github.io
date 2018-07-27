---
title: "如何在公司办公室里远程工作"
date: 2018-07-23T09:40:29+08:00
draft: false
---

如果你家里用一个 NAS 的话会很自然地想到我能不能在地铁上用手机浏览阅览 NAS 上的内容. 如果你上班时必须使用公司发的 Mac 的话能随时访问家里运行 Linux 的笔记本就变成了一个很强的需求.
很自然的就会想到用 VPN, 先列一下需求再来决定该用什么样子的 VPN

1. 是个 VPN
1. 加密
1. 多平台支持
1. 可扩展性
1. 可批量部署
1. 起码能跑 50MiB/s, 要高于常见设备的 Wifi 速度
1. 能局域网和公网混用

使用这个 VPN 的流程应该是

1. 部署 VPN 服务器
1. 分发配置
1. 每个客户端启动 VPN client, 加一条 `/24` 的路由指向 VPN server
1. 某些情况下再加一条 `/0` 路由, 启动全局 VPN
1. 在两个 client 互相可见的情况下直接通讯, 不可见时才通过 VPN server 通讯

这第 5 条需求叫做 mesh net, 常见于无线通讯等两个 client 可能互相看不见的情况 [WIKI](https://en.wikipedia.org/wiki/Mesh_networking), 以前用 ZigBee 接触过这种网络结构, 但那时用的 SDK 封装的很完全, 我只管发包就好.

确定了需求之后选择软件

| VPN 名字  	| 为啥不行                              	|
|-----------	|---------------------------------------	|
| SS/V2Ray  	| 不是个 VPN                            	|
| 裸 IPsec  	| 山寨路由器不支持, 不知道怎么死的      	|
| OpenVPN   	| 会被墙                                	|
| L2TP      	| 部署复杂, 山寨路由器上速度只有 3MiB/s 	|
| WireGuard 	| 目前不支持 Mesh                       	|

最后选择了 [tinc](https://www.tinc-vpn.org/) 因为他们的 goals 就是我的需求

## 搭建 tinc 网络

tinc 没有客户端和服务器端, 每个 tincd 都会监听一个 udp 端口, 接受其他 tincd 的链接

一个标准并完整的 tinc 配置文件长这样

```
sa@mac /e/tinc> tree
.
├── ed25519_key.priv
├── hosts
│   ├── mac
│   ├── tx
│   └── vultr
├── rsa_key.priv
├── tinc-up
└── tinc.conf
```

- tinc-up 是一个 shell 脚本, 用来添加网卡的 ip, 每个平台不同, Windows 下是 bat 脚本, 安卓则不读这个文件
- 以前的 tincd 还需要写 tinc-down 用来关闭网卡
- tinc.conf 这里写 tincd 在启动时的配置具体参数 `man tinc.conf`
- hosts 写启动时信任的公钥

网络拓扑图:

![](/img/tinc/tinc_network.png)

tinc 运行时有三种模式, 分别对应 hub switch router, 我这里运行模式全是 router, 所以需要手动配一下 ip, 上面拓扑图中的 ip 要同时写在系统的路由表(tinc-up)和 tinc 自己的路由表( host/*) 中.

拓扑图中的连线没有意义, 因为 mesh 网络会自动发现可以连接的设备, 我配置的 tincd 一开始都会去连接 vultr 和 tx 两台机器, 获取元数据后会开始和别的 tincd 互联.

值得一提的是 vultr 的 subnet 是 /0 原因是这样子当 tincd 运行时会把所有的包当做内网包, 从实现使用流程中的第4条.

我并没有把内网 ip 和 vpn 内 ip 整合, 虽然 tinc 内可以跑 dhcp 协议, 但并不是所有客户端都能运行 dhcp 客户端


# 使用体验

tinc 有一套[消息序号](https://github.com/gsliepen/tinc/blob/master/doc/PROTOCOL), 所以只能用一个线程处理一个连接.

在 J1800 上使用 HTTP 下载文件, 最大速度是 65MiB/s 没有达到千兆, 此时 J1800 已经满载.

两台机器不互相通讯时连接建立的时间相当长, 但互相通讯时(哪怕只有一个 ping) 连接建立需要5-10秒, 直接体验就是 vpn 内拷贝文件前几秒速度只有3MiB/s, 然后飙升至65MiB/s

# 如何在办公室远程工作?

```
ssh sa@10.1.0.2  "ssh mac@10.1.0.7"
sudo mount -t nfs -o resvport,rw 10.1.0.4:/data /private/nfs
```
