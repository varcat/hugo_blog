---
title: "frp 内网穿透"
description:
date: 2026-01-12T22:00:00+08:00
image: 
math:
license:
hidden: false
comments: false
draft: true
slug: frp
tags:
    - network
---

## 前言

作为一名程序员，我家里折腾了一台性能强劲的 NAS，但受限于内网环境，出门就成了“断线鱼”。虽然手里有一台云服务器，但那点可怜的 CPU 和内存资源，跑个大型服务实在吃力。

最好的方案莫过于：云服务器只做“路牌”，把流量转发到家里的“性能猛兽”上。

在对比了多种工具后，我最终选择了 frp。它轻量、稳定且配置极简，完美解决了我的内网穿透需求。今天就把这套“云端中转 + 本地高配”的搭建方案分享给大家。

## 为什么选择 frp？

| 方案 | 优点 | 缺点 | 适用人群 |
| --- | --- | --- | --- |
| DDNS (域名解析) | 原生速度快，无中转延迟 | 必须有公网 IP，受限于运营商 | 有公网 IP 的技术大牛 |
| 商业工具 (如花生壳) | 傻瓜式操作，无需服务器 | 免费版带宽极低、限制流量、不安全 | 零基础小白 |
| VPN (如 Tailscale) | 安全性极高，无需中转机 | 需安装客户端，难实现公网直接访问 | 纯私人远程办公 |
| frp (自主搭建) | 支持全协议、高度自定义、稳定高效 | 需准备一台公网 VPS | 追求极致体验的玩家 |

## frp 工作原理简析

* 角色分配： 
  * frps (Server)： 部署在公网服务器，负责监听和转发。
  * frpc (Client)： 部署在内网机器，负责发起连接。
* 流量走向： 用户请求 -> 公网服务器 (frps) -> 隧道 -> 内网设备 (frpc) -> 本地服务。

## 准备工作

* 硬件： 一台带公网 IP 的 VPS（云服务器）。
* 软件： frp [GitHub Release](https://github.com/fatedier/frp/releases) 下载对应架构的包。
* 环境： 确认服务器防火墙已放行对应端口（如 7000, 80/443 等）。

## 实战配置步骤

### 服务端配置 (frps.toml)
* 设置绑定端口。

* 安全加固： 设置 auth.token（防止别人蹭你的服务器）。

```toml
bindPort = 7000 # 默认端口
auth.token = "your-secure-token" # 自定义token，建议设置为复杂字符串
transport.tcpMuxKeepaliveInterval = 60 # tcp mux 的心跳检查间隔时间，单位秒

# 端口限制（可选，提升安全性）
allowPorts = [
  { start = 10000, end = 100010 },   # 允许 10000-100010 端口
  { single = 3306 },                 # 允许 3306 端口
]
```

### 客户端配置 (frpc.toml)

```toml
# 基本配置
serverAddr = "" # 公网服务器 IP 或域名
serverPort = 7000 # 公网服务器端口，需与 bindPort 一致
auth.token = "your-secure-token" # 与服务端配置一致

# 基础传输优化
transport.tcpMux = true
transport.tcpMuxKeepaliveInterval = 60 # tcp mux 的心跳检查间隔时间，单位秒

# tcp 传输
[[proxies]]
name = "ssh"
type = "tcp"
localPort = 22
remotePort = 8001 # ssh -p 8001 user@serverAddr
```

## 常见问题排查 (FAQ)

### 连接超时 (Timeout)
**现象**：客户端启动时提示 `login to server failed: dial tcp x.x.x.x:7000: i/o timeout`。
**原因**：
*   服务器防火墙未放行 `bindPort` (默认 7000)。
*   云服务商的安全组未开放对应端口。
*   服务器 IP 地址填写错误。
**解决**：
*   检查服务器防火墙设置 (如 `ufw`, `iptables`)。
*   登录云服务商控制台，检查安全组规则，确保 TCP 7000 端口已放行。
*   确认客户端配置中的 `serverAddr` 是否正确。

### 认证失败 (Authorization Failed)
**现象**：提示 `login to server failed: authorization failed`。
**原因**：
*   客户端和服务端的 `auth.token` 不一致。
*   时间不同步 (frp 对客户端和服务端的时间差有要求，默认允许偏差 15 分钟)。
**解决**：
*   确保 `frps.toml` 和 `frpc.toml` 中的 `auth.token` 完全一致。
*   检查服务器和客户端的系统时间，必要时进行时间同步 (NTP)。

### 连接被拒绝 (Connection Refused)
**现象**：提示 `dial tcp x.x.x.x:7000: connect: connection refused`。
**原因**：
*   服务端 (frps) 未启动或已停止运行。
*   服务端监听端口不是 7000。
**解决**：
*   在服务器上执行 `ps aux | grep frps` 检查进程是否存在。
*   检查服务端配置文件 `frps.toml` 中的 `bindPort` 设置，确保与客户端一致。

### 端口已被占用 (Port Already In Use)
**现象**：启动时提示 `port already in use`。
**原因**：
*   指定的端口 (如 7000 或远程映射端口) 已经被其他程序占用。
*   frp 进程重复启动。
**解决**：
*   使用 `netstat -tunlp | grep <端口号>` 查找占用端口的进程。
*   修改配置文件中的端口号，或结束占用端口的进程。

## 结语

通过以上配置，我们成功打通了公网与内网的阻隔，无论是远程管理 NAS、访问内部 Web 服务，还是调试本地代码，frp 都能提供稳定高效的支持。

相比于其他方案，frp 虽然有一定的配置门槛，但其带来的灵活性和高性能体验是无可比拟的。希望这篇文章能帮你快速上手，搭建属于自己的内网穿透服务，尽情释放本地设备的强大潜能！
