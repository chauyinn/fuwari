---
title: Caddy 反向代理 RDP
published: 2025-05-10
description: '通过 Caddy 和 caddy-l4 插件实现 RDP 的反向代理'
image: ''
tags: ["Caddy", "反向代理", "RDP"]
category: '网络工具'
draft: false 
lang: 'zh-cn'
---

## 前言

[Caddy](https://github.com/caddyserver/caddy) 是一个开源的 Web 服务器，支持反向代理、负载均衡和 HTTPS 等功能。Caddy 的核心功能是处理 HTTP 和 HTTPS 请求，支持自动获取和管理 SSL/TLS 证书，非常适合用作 Web 服务器或反向代理。不过 Caddy 自身并不支持 RDP 协议的直接处理，但可以通过一些配置和插件来实现 RDP 的反向代理，这里用到的是 [caddy-l4](https://github.com/mholt/caddy-l4) 插件。

## 安装 Caddy

1. 通过 [Caddy 构建页面](https://caddyserver.com/download)，选择需要的插件进行构建，例如这次需要的是 `caddy-l4` 插件。

2. 参考 [Caddy 官方文档](https://caddyserver.com.cn/docs/install) 安装 Caddy。

## 配置 RDP 反向代理

在 `Caddyfile` 中添加以下内容：

```CaddyFile
{
    layer4 {
            :3389 {
                @rdp rdp
                route @rdp {
                    proxy 192.168.1.100:3389
                }
            }
    }
}
```

配置说明：
- `layer4`：指定使用 `layer4` 模块来处理 TCP/UDP 流量。
- `:3389`：监听本地服务器的 3389 端口。
- `@rdp`：定义一个名为 `@rdp` 的匹配器，用于匹配 RDP 流量。
- `route @rdp`：定义一个路由规则，当匹配到 `@rdp` 时执行后续的代理操作。
- `proxy`：将匹配到的流量反向代理到目标服务器的 RDP 端口。
