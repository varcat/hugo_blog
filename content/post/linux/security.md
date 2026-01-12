---
title: "Linux 服务器安全加固指南：从被扫描到睡个安稳觉"
description: "记录一次阿里云服务器被暴力破解后的安全加固过程，包括 SSH Key、修改端口、Fail2Ban 等实战配置。"
date: 2025-11-12T19:34:00+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:
    - linux
---

## 前言

我手头这台阿里云 ECS 实例平时主要用于托管博客和运行 Docker 服务，此前并未在安全配置上投入太多精力。

直到最近偶然查看系统日志时，才发现 `/var/log/secure` 中充斥着密密麻麻的 `Failed password for root` 记录。经溯源分析，这些恶意登录尝试来自全球各地。面对这数万条暴力破解记录，我不禁对服务器的处境感到担忧——暴露在公网环境下的服务器，实际上正时刻经受着各类自动化脚本的无差别扫描与攻击。

尽管我设置了高强度的复杂密码，但这种被持续“盯上”的潜在风险始终令人不安。一旦防御被突破，服务器上的数据安全将无从谈起。

因此，我特意抽出时间对服务器进行了全面的安全加固。本文将详细记录这一整改过程，希望能为同样运行着公网服务器的朋友们提供一份实用的安全参考。

## 第一步：知己知彼，日志排查

在动手之前，先看看情况到底有多严峻。

### 查查有没有人已经混进来了

这是最关键的，先确认“处境”。使用 `last` 命令查看最近成功的登录记录：

```bash
last
```

重点看有没有陌生的 IP 地址。如果是自己不认识的 IP，那说明已经被攻破了，这时候重装系统可能是最稳妥的选择。好在我查了一圈，除了我自己的 IP，没有别的成功记录。

### 看看是谁在敲门

看看那些失败的尝试都来自哪里。可以使用以下命令统计一下攻击者的 Top 10 IP：

```bash
# CentOS/RHEL 系通常在 /var/log/secure
# Ubuntu/Debian 系通常在 /var/log/auth.log
grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head -n 10
```

跑完这个命令，看着那些成百上千的数字，你会有动力的。

## 第二步：基础加固，把门焊死

既然是 SSH 暴力破解，那我们就在 SSH 上做文章。

### 禁用 Root 直接登录，创建普通用户

直接用 root 登录是大忌。我们先创建一个普通用户，并赋予 sudo 权限。

```bash
# 创建用户 user (换成你自己的名字)
useradd -m user
passwd user

# 赋予 sudo 权限
# CentOS/RHEL:
usermod -aG wheel user
# Ubuntu/Debian:
usermod -aG sudo user
```

### 配置 SSH Key 登录，禁用密码

密码再复杂也有被撞破的可能，密钥就安全多了。

在你的**本地电脑**（不是服务器）上生成密钥对（如果你还没有的话）：

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

然后把公钥传到服务器上：

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@<服务器IP>
```

测试一下能不能用密钥登录成功。**确认能登录后**，再继续下一步。

### 修改 SSH 配置文件

编辑 `/etc/ssh/sshd_config`，做以下几个关键修改：

```bash
sudo vim /etc/ssh/sshd_config
```

找到并修改（或添加）以下配置：

```bash
# 1. 修改默认端口，避开 99% 的脚本扫描
# 选一个 1024-65535 之间的端口，比如 22222
Port 22222

# 2. 禁止 root 登录
PermitRootLogin no

# 3. 禁止密码登录（必须确保密钥登录已配置好！）
PasswordAuthentication no

# 4. 仅允许特定用户登录（可选，更严格）
AllowUsers user
```

改完后，别急着重启 sshd，先去**阿里云控制台的安全组**里，把 TCP 22222 端口放行！否则一重启你就把自己关门外了。

放行端口后，重启 SSH 服务：

```bash
sudo systemctl restart sshd
```

这时候，绝大多数的扫描脚本因为连不上 22 端口，就已经放弃你的服务器了。

## 第三步：主动防御，Fail2Ban

修改端口只能防住傻瓜脚本，对于定向扫描还是不够。这时候就需要 Fail2Ban 登场了。它的原理很简单：检测日志，发现某个 IP 在短时间内多次登录失败，就调用防火墙把它封禁一段时间。

### 安装 Fail2Ban

```bash
# CentOS/RHEL (需要先安装 EPEL 源)
sudo yum install epel-release -y
sudo yum install fail2ban -y

# Ubuntu/Debian
sudo apt-get install fail2ban -y
```

### 配置 Jail

Fail2Ban 的默认配置文件在 `/etc/fail2ban/jail.conf`，但我们不要直接改它，而是创建一个 `jail.local`：

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo vim /etc/fail2ban/jail.local
```

重点修改 `[sshd]` 部分：

```ini
[sshd]
enabled = true
# 如果你改了端口，这里一定要改
port    = 22222
# 过滤规则
filter  = sshd
# 日志路径
logpath = /var/log/secure
# 最大尝试次数
maxretry = 3
# 封禁时间（秒），这里设为一天
bantime = 86400
# 检测时间窗口（秒）
findtime = 600
```

### 启动并查看状态

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

过一段时间，你可以用这个命令看看战果：

```bash
sudo fail2ban-client status sshd
```

你会看到 "Currently banned" 后面跟着一串 IP，看着它们被关进小黑屋，心里莫名暗爽。

## 第四步：最后的防线，防火墙

虽然阿里云有安全组（Security Group），但我建议系统内部的防火墙（Firewalld 或 UFW）也要开起来，双重保障。

以 CentOS 的 Firewalld 为例：

```bash
# 启动防火墙
sudo systemctl start firewalld
sudo systemctl enable firewalld

# 放行新的 SSH 端口
sudo firewall-cmd --permanent --add-port=22222/tcp

# 移除默认的 ssh 服务（因为它对应 22 端口）
sudo firewall-cmd --permanent --remove-service=ssh

# 重载配置
sudo firewall-cmd --reload
```

## 总结

经过这一番折腾：
1.  **SSH 端口改了**，扫描脚本找不到门。
2.  **Root 禁了**，必须用普通用户提权。
3.  **密码登不上了**，必须有密钥。
4.  **Fail2Ban 守着**，谁敢暴力破解直接封 IP。

再次审视系统日志，异常登录尝试已大幅减少，日志环境恢复了应有的清朗。

网络安全领域不存在绝对的“铜墙铁壁”，防御的核心策略在于不断提升攻击者的成本。对于个人服务器而言，切实落实上述安全加固措施，足以有效抵御绝大多数自动化脚本的恶意扫描与攻击。

至此，服务器的基础安全防线已构建完成。
