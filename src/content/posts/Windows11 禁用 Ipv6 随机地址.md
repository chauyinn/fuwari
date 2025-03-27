---
title: Windows11 禁用 IPv6 随机地址
published: 2025-03-23
description: '解决 Windows11 系统中 IPv6 随机标识符禁用后重启失效的问题，通过计划任务实现持久化设置'
image: ''
tags: ["Windows", "网络", "IPv6"]
category: '系统优化'
draft: false 
lang: 'zh-cn'
---

## 前言

Windows11 默认启用了 IPv6 随机标识符（RandomizeIdentifiers），这是一种保护隐私的机制，可以防止用户在网络上被追踪。但在某些情况下，我们需要固定 IPv6 地址后缀以方便外部访问（如远程桌面访问家庭服务器）。

Windows 提供 `Set-NetIPv6Protocol -RandomizeIdentifiers Disabled` 命令来禁用随机标识符，但在 Windows11 最新版本中，执行该命令后，重启系统后随机标识符仍然会被重新启用，这似乎是一个系统bug。相关问题讨论可见：
- [Microsoft 问答社区：IPv6 设置在重启后不保留](https://learn.microsoft.com/en-us/answers/questions/1130080/ipv6-settings-not-remaing-after-reboot)
- [Reddit 讨论：Windows11 将 IPv6 地址重置为随机模式](https://www.reddit.com/r/ipv6/comments/1el83oy/ipv6_eui64_address_non_random_windows_11_reverts/)

## 问题表现

网卡 MAC 值为 `28:cf:e9:12:5d:fe`，在启用随机标识符时，同一台设备在不同时间会有不同的IPv6地址后缀，如：
- 2001:db8::/64 前缀下的地址: **3725:f4b2:a198:e3d4**
- 2001:db8::/64 前缀下的地址: **8f12:c637:92ae:5b19**（重启后）

禁用随机标识符后，IPv6地址应该保持固定的MAC地址派生后缀，如：
- 2001:db8::/64 前缀下的地址: **2acf:e9ff:fe12:5dfe**（基于网卡MAC地址生成）

:::tip[IPv6 EUI-64地址生成规则说明]

当禁用随机化时，Windows使用EUI-64规则从MAC地址生成IPv6接口标识符：
1. 取MAC地址（例如`28:cf:e9:12:5d:fe`）
2. 在MAC地址中间插入`ff:fe`：`28:cf:e9:ff:fe:12:5d:fe`
3. 翻转MAC地址第7位（通用/本地位）：`28`的二进制是`00101000`，翻转得到`00101010`即`2a`
4. 最终接口标识符：`2acf:e9ff:fe12:5dfe`

这样生成的IPv6地址是可预测的，便于网络管理，但也可能带来隐私风险。
:::

## 解决方法

通过 Windows 计划任务来执行禁用随机标识符的命令，以确保重启后随机标识符仍然禁用。

### 1. 创建计划任务

创建一个系统启动时自动运行的计划任务，来执行禁用随机标识符的命令。有两种方法可以创建此任务：

1. 按 `Win + R` 打开运行对话框，输入 `taskschd.msc` 并点击确定
2. 在任务计划程序中，右键点击"任务计划程序库"，选择"创建任务"
3. 常规选项卡中：
    - 名称：输入 `Disable IPv6 RandomizeIdentifiers`
    - 选择不管用户是否登录都运行
    - 选择"使用最高权限运行"
4. 触发器选项卡中：
    - 点击"新建"，选择"启动时"，点击确定
5. 操作选项卡中：
    - 点击"新建"
    - 程序/脚本：输入 `powershell.exe`
    - 添加参数：输入 `-Command "Set-NetIPv6Protocol -RandomizeIdentifiers Disabled"`
    - 点击确定
6. 点击"确定"保存任务（可能需要输入管理员密码）

### 2. 验证配置

创建任务后，可以手动运行一次或重启系统来验证配置：

1. 重启系统后，打开 PowerShell
2. 执行命令：`Get-NetIPv6Protocol | Select RandomizeIdentifiers`
3. 确认输出显示 `Disabled`

如果显示为 `Disabled`，则表示配置成功，IPv6 地址将不再随机化。
