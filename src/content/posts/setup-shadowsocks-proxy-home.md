---
title: 搭建 shadowsocks 代理回家
published: 2025-03-23
description: '通过 shadowsocks 搭建安全的家庭网络代理，实现远程访问家庭局域网资源'
image: ''
tags: ["docker", "代理", "网络", "shadowsocks"]
category: '网络工具'
draft: false 
lang: 'zh-cn'
---

## 前言

不在家庭网络时，比如外出，可能需要访问家中不方便暴露在公网的局域网服务。通过搭建 shadowsocks 代理，你可以"回家"，就像身处局域网一样访问这些资源。相比于 WireGuard 组网，shadowsocks 可以通过代理工具编写复杂的规则，实现更灵活的代理策略。

## 准备工作

- 一台拥有公网 IP 的设备，IPv4 或 IPv6 均可
- 设备上已安装 Docker
- 确保设备和设备网关的 8388 端口（或你自定义的端口）已开放

## 服务端安装 shadowsocks

首先，创建配置文件目录：

```bash
mkdir -p ~/ss-server && cd ~/ss-server
```

编写配置文件 `config.json`：

```json
{
  "server": "::",
  "server_port": 8388,
  "password": "your_strong_password_here",
  "method": "chacha20-ietf-poly1305",
  "mode": "tcp_and_udp"
}
```

然后启动 shadowsocks 服务器：

```bash
docker run -d --name ss-server --restart always --network host \
  -v ~/ss-server/config.json:/etc/shadowsocks-rust/config.json \
  ghcr.io/shadowsocks/ssserver-rust:latest
```

> **安全提示**：请务必将配置中的 `password` 替换为强密码，避免使用默认值！

## 客户端安装 shadowsocks

根据你的设备选择适合的客户端：

- **安卓设备**：
  - [shadowsocks Android](https://github.com/shadowsocks/shadowsocks-android/releases)
  - [v2rayNG](https://github.com/2dust/v2rayNG/releases)

- **Windows 设备**：
  - [shadowsocks Windows](https://github.com/shadowsocks/shadowsocks-windows/releases)
  - [Clash Verge Rev](https://github.com/clash-verge-rev/clash-verge-rev/releases)

## 配置客户端

以安卓 shadowsocks 为例：

1. 打开 shadowsocks 应用
2. 点击右上角 "+" 添加配置
3. 手动配置以下信息：
   - 配置名称：自定义（如"家庭网络"）
   - 服务器地址：服务端公网 IP
   - 服务器端口：8388（或你自定义的端口）
   - 密码：你在服务端设置的密码
   - 加密方法：chacha20-ietf-poly1305
   - 路由：全局模式（或根据需要选择）
4. 保存并启用配置

## 测试连接

1. 客户端已连接到 shadowsocks
2. 尝试访问家庭网络中的资源，如：
   - 局域网服务
   - NAS 管理界面

## 常见问题排查

### 无法连接

- 检查服务器防火墙是否已开放 8388 端口
- 确认配置文件中的密码和加密方法与客户端一致
- 查看服务器日志：`docker logs ss-server`

### 连接速度慢

- 尝试更换加密方法，如 `aes-256-gcm`
- 检查服务器带宽限制
- 考虑更换网络质量更好的服务器
