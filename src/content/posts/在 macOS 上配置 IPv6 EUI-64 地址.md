---
title: 在 macOS 上配置 IPv6 EUI-64 地址
published: 2025-04-14
description: '通过 Python 脚本在 macOS 上实现 IPv6 EUI-64 地址配置，固定 IPv6 地址便于远程访问'
image: ''
tags: ["macOS", "IPv6", "网络", "EUI-64", "自动化"]
category: '系统配置'
draft: false 
lang: 'zh-cn'
---

## 前言

macOS 的 IPv6 地址生成机制基于 [RFC 7217](https://www.rfc-editor.org/rfc/rfc7217)，采用隐私保护方式而非直接使用 MAC 地址创建固定的标识符。与 Android 类似，macOS 主要通过 [SLAAC](https://en.wikipedia.org/wiki/IPv6#Stateless_address_autoconfiguration_(SLAAC))（无状态地址自动配置）获取 IPv6 地址。虽然 macOS 技术上支持 [DHCPv6](https://en.wikipedia.org/wiki/DHCPv6)，但在大多数网络环境中仍优先使用 SLAAC，且值得注意的是 Android 系统完全不支持 DHCPv6。当使用 Mac 作为远程服务器且仅有 IPv6 公网访问时，配置 EUI-64 格式的地址可以使 IPv6 地址后缀保持固定，从而实现稳定可靠的远程访问。

虽然可以在 macOS 上使用 [DDNS-GO](https://github.com/jeessy2/ddns-go) 将动态 IPv6 地址绑定到域名，但这种方法存在局限性：
- 地址后缀不固定（RFC 7217 特性），需要频繁调整防火墙规则
- 依赖于第三方 DDNS 服务的可靠性
- 域名解析可能存在延迟

本文介绍如何使用 Python 脚本在 macOS 上配置 IPv6 EUI-64 地址，实现地址后缀固定，便于构建稳定的远程访问方案。

## 脚本实现

> [!NOTE]
> 代码主要由 AI 辅助生成。

以下是一个 Python 脚本，用于自动获取系统的 IPv6 前缀，结合 MAC 地址生成 EUI-64 地址，并将其应用到网络接口：

```python
import subprocess
import os
import re
import argparse
import time

def get_ipv6_prefix_from_system():
    """从 macOS 系统获取 IPv6 前缀，只获取自动配置的地址前缀"""
    try:
        # 获取所有网络接口
        result = subprocess.run(['ifconfig'], capture_output=True, text=True, check=True)
        interfaces_output = result.stdout
        
        # 解析出接口名称
        interface_pattern = re.compile(r'^(\w+):.*$', re.MULTILINE)
        interfaces = interface_pattern.findall(interfaces_output)
        
        # 存储找到的前缀
        prefixes = []
        # 用于跟踪已经处理的前缀和接口组合
        processed_prefixes = set()

        for interface in interfaces:
            # 跳过回环接口和隧道接口
            if interface == 'lo0' or interface.startswith('utun'):
                continue
                
            # 获取该接口的详细信息
            if_result = subprocess.run(['ifconfig', interface], capture_output=True, text=True, check=True)
            if_output = if_result.stdout
            
            # 查找带有autoconf secured或autoconf temporary标记的IPv6地址
            ipv6_pattern = re.compile(r'inet6\s+([a-f0-9:]+)(%\w+)?\s+prefixlen\s+(\d+).*?(autoconf secured|autoconf temporary)', re.IGNORECASE)
            ipv6_addresses = ipv6_pattern.findall(if_output)
            
            for addr_info in ipv6_addresses:
                addr = addr_info[0]
                prefix_length = int(addr_info[2])
                autoconf_type = addr_info[3]
                
                # 跳过链路本地地址（通常不需要，因为前面的正则已经过滤了）
                if addr.startswith('fe80:'):
                    continue
                
                # 计算网络前缀
                if prefix_length == 64:
                    # 直接使用前64位作为前缀
                    prefix_parts = addr.split(':')[:4]
                    prefix = ':'.join(prefix_parts)
                    
                    # 检查接口和前缀组合是否已处理
                    prefix_key = f"{interface}_{prefix}"
                    if prefix_key in processed_prefixes:
                        continue
                    
                    processed_prefixes.add(prefix_key)
                    
                    prefixes.append({
                        'interface': interface,
                        'prefix': prefix,
                        'prefix_length': 64,
                        'full_addr': addr,
                        'type': autoconf_type
                    })
                    
        return prefixes
    except Exception as e:
        return f"获取 IPv6 前缀时出错: {e}"

def wait_for_ipv6_prefix(interface=None, max_attempts=12, wait_time=10):
    """等待 IPv6 前缀可用，最多等待指定次数和时间"""
    print(f"等待 IPv6 前缀可用{'（接口: ' + interface + '）' if interface else ''}...")
    
    for attempt in range(1, max_attempts + 1):
        prefixes = get_ipv6_prefix_from_system()
        
        # 检查是否获取到前缀列表（而不是错误消息）
        if isinstance(prefixes, list) and prefixes:
            # 如果指定了接口，检查该接口是否有前缀
            if interface:
                interface_prefixes = [p for p in prefixes if p['interface'] == interface]
                if interface_prefixes:
                    print(f"在第 {attempt} 次尝试后成功获取到IPv6前缀")
                    return prefixes
            else:
                # 未指定接口，有任何前缀即可
                print(f"在第 {attempt} 次尝试后成功获取到IPv6前缀")
                return prefixes
        
        # 未找到前缀，等待后重试
        print(f"尝试 {attempt}/{max_attempts}: 未找到IPv6前缀，等待 {wait_time} 秒后重试...")
        time.sleep(wait_time)
    
    # 超过最大尝试次数
    print(f"超时：在 {max_attempts} 次尝试后仍未能获取到IPv6前缀")
    return []

def get_mac_address(interface):
    """获取指定网络接口的 MAC 地址"""
    try:
        result = subprocess.run(['ifconfig', interface], capture_output=True, text=True, check=True)
        output = result.stdout
        
        # 查找MAC地址
        mac_pattern = re.compile(r'ether\s+([0-9a-f:]{17})', re.IGNORECASE)
        match = mac_pattern.search(output)
        
        if match:
            return match.group(1)
        return None
    except:
        return None

def mac_to_eui64(mac, prefix):
    """将 MAC 地址转换为 EUI-64 地址"""
    # 移除MAC地址中的冒号
    mac = mac.replace(':', '')

    # 在MAC地址中间插入FFFE
    eui64 = mac[0:6] + 'fffe' + mac[6:]

    # 转换为字节列表
    eui64_bytes = bytearray.fromhex(eui64)

    # 反转第7位（从0开始计数）
    eui64_bytes[0] ^= 0b00000010

    # 将字节列表转换回十六进制字符串
    modified_eui64 = ''.join(f'{b:02x}' for b in eui64_bytes)

    # 按照IPv6地址格式插入冒号
    eui64_parts = [modified_eui64[i:i+4]
                   for i in range(0, len(modified_eui64), 4)]

    # 创建完整的IPv6地址
    prefix_parts = prefix.split(':')
    while '' in prefix_parts:
        prefix_parts.remove('')

    # 确保我们只使用前缀的前4个部分（64位）
    prefix_parts = prefix_parts[:4]

    # 组合前缀和EUI-64部分
    ipv6_parts = prefix_parts + eui64_parts

    # 构建IPv6地址
    ipv6_address = ':'.join(ipv6_parts)

    return ipv6_address

def add_eui64_address(interface, eui64_ipv6):
    """添加 EUI-64 格式的 IPv6 地址到指定接口"""
    if os.geteuid() != 0:
        return "需要管理员权限才能添加IP地址，请使用sudo运行此脚本"

    try:
        # 直接添加EUI-64地址，不移除现有地址
        subprocess.run(['ifconfig', interface, 'inet6', eui64_ipv6, 'prefixlen', '64', 'alias'],
                      check=True, capture_output=True)

        return f"已成功添加EUI-64地址 {eui64_ipv6} 到接口 {interface}"
    except subprocess.CalledProcessError as e:
        return f"添加IPv6地址时出错: {e}"
    except Exception as e:
        return f"发生错误: {e}"

def main():
    """主函数"""
    # 解析命令行参数
    parser = argparse.ArgumentParser(description='从系统获取IPv6前缀并添加EUI-64地址')
    parser.add_argument('--interface', help='指定要配置的网络接口')
    parser.add_argument('--apply', action='store_true', help='自动应用生成的EUI-64地址')
    parser.add_argument('--retry', type=int, default=12, help='最大重试次数（默认12次）')
    parser.add_argument('--wait', type=int, default=10, help='重试间隔秒数（默认10秒）')

    args = parser.parse_args()

    # 获取IPv6前缀
    print("正在从系统获取IPv6前缀...")
    
    # 使用带重试逻辑的函数获取IPv6前缀
    if args.apply:
        prefixes = wait_for_ipv6_prefix(args.interface, args.retry, args.wait)
    else:
        prefixes = get_ipv6_prefix_from_system()

    if isinstance(prefixes, str):
        print(prefixes)  # 打印错误信息
        return

    if not prefixes:
        print("未找到IPv6前缀，请确保您的网络连接支持IPv6")
        return

    print(f"找到 {len(prefixes)} 个IPv6前缀")

    results = []

    # 为每个前缀和接口生成EUI-64地址
    for prefix_info in prefixes:
        interface = prefix_info['interface']
        prefix = prefix_info['prefix']

        mac = get_mac_address(interface)
        if not mac:
            print(f"接口 {interface} 未找到MAC地址")
            continue

        eui64_address = mac_to_eui64(mac, prefix)

        results.append({
            'interface': interface,
            'prefix': prefix,
            'mac': mac,
            'current_ipv6': prefix_info['full_addr'],
            'eui64_ipv6': eui64_address
        })

    # 显示结果
    for i, result in enumerate(results, 1):
        print(f"\n{i}. 接口: {result['interface']}")
        print(f"   MAC地址: {result['mac']}")
        print(f"   IPv6前缀: {result['prefix']}/64")
        print(f"   当前IPv6: {result['current_ipv6']}")
        print(f"   EUI-64 IPv6: {result['eui64_ipv6']}")
        print(f"   \n   要添加此地址，可以使用命令:")
        print(
            f"   sudo ifconfig {result['interface']} inet6 {result['eui64_ipv6']} prefixlen 64 alias")

    # 如果指定了应用参数和接口
    if args.apply:
        if os.geteuid() != 0:
            print("\n需要管理员权限才能添加IP地址，请使用sudo运行此脚本")
            return

        # 确定要配置的接口
        selected_interface = args.interface
        selected_result = None

        if selected_interface:
            # 使用指定的接口
            selected_results = [
                r for r in results if r['interface'] == selected_interface]
            if selected_results:
                selected_result = selected_results[0]
            else:
                print(f"\n未找到指定的接口: {selected_interface}")
                return
        elif results:
            # 使用第一个结果
            selected_result = results[0]
        else:
            print("\n未找到可配置的接口")
            return

        print(f"\n正在添加EUI-64地址到接口 {selected_result['interface']}...")
        result = add_eui64_address(
            selected_result['interface'], selected_result['eui64_ipv6'])
        print(result)

        # 输出验证信息
        print("\n当前接口状态:")
        subprocess.run(['ifconfig', selected_result['interface']], check=False)

if __name__ == "__main__":
    main()
```

## 使用方法

### 手动运行

这个脚本的核心功能是获取 IPv6 前缀并生成固定的 EUI-64 地址，提供了多种使用方式：

查看可用的 IPv6 前缀和可生成的 EUI-64 地址：

```bash
python3 add_eui64.py
```

自动应用 EUI-64 地址（需要管理员权限）：

```bash
sudo python3 add_eui64.py --apply
```

为特定接口应用 EUI-64 地址：

```bash
sudo python3 add_eui64.py --apply --interface en0
```

自定义重试参数（例如最多尝试 24 次，每次等待 5 秒）：

```bash
sudo python3 add_eui64.py --apply --interface en0 --retry 24 --wait 5
```

### 与 DDNS-GO 结合使用

如果你已经在使用 [DDNS-GO](https://github.com/jeessy2/ddns-go)，可以利用以下命令提取 IPv6 前缀并拼接固定的 EUI-64 后缀：

```bash
ifconfig | grep "autoconf secured\|autoconf temporary" | awk '{print $2}' | awk -F: '{print $1":"$2":"$3":"$4}' | head -n 1 | xargs -I{} echo "{}:ffff:ffff:ffff:ffff"
```

这个命令会获取当前网络接口的 IPv6 前缀，并拼接指定的 EUI-64 后缀（示例中为`ffff:ffff:ffff:ffff`，请替换为您设备生成的实际 EUI-64 标识符）。

## 设置自动化服务

为了确保系统重启或网络变化后 EUI-64 地址始终存在，建议配置定时运行的系统服务：

### 1. 安装脚本

首先，将脚本安装到系统目录并设置权限：

```bash
sudo cp add_eui64.py /usr/local/bin/
sudo chmod 755 /usr/local/bin/add_eui64.py
```

### 2. 创建 LaunchDaemon 配置

创建一个 plist 文件，定义服务的运行参数和计划：

```bash
sudo nano /Library/LaunchDaemons/com.eui64.plist
```

填入以下内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.eui64</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/python3</string>
        <string>/usr/local/bin/add_eui64.py</string>
        <string>--apply</string>
        <string>--interface</string>
        <string>en0</string>
        <string>--retry</string>
        <string>12</string>
        <string>--wait</string>
        <string>10</string>
    </array>
    <key>StartInterval</key>
    <integer>3600</integer>
    <key>RunAtLoad</key>
    <true/>
    <key>StandardErrorPath</key>
    <string>/dev/null</string>
    <key>StandardOutPath</key>
    <string>/dev/null</string>
</dict>
</plist>
```

> [!NOTE]
> 将上面的 `en0` 替换为你系统中实际的网络接口名称。
> 
> 关于日志文件：不保存日志（使用 /dev/null）或选择将日志保存到临时目录（/tmp），避免长期运行导致日志文件过大占用存储空间。

### 3. 设置权限并加载服务

```bash
sudo chown root:wheel /Library/LaunchDaemons/com.eui64.plist
sudo chmod 644 /Library/LaunchDaemons/com.eui64.plist
sudo launchctl load /Library/LaunchDaemons/com.eui64.plist
```

服务将在系统启动时自动运行，并且每小时运行一次，确保 EUI-64 地址始终可用。

## 验证配置

### 检查服务状态

确认服务是否正常加载：

```bash
sudo launchctl list | grep com.eui64
```

如果服务已加载，您将看到类似以下的输出：

```
-   0   com.eui64
```

### 验证 IPv6 地址

查看网络接口上是否已添加 EUI-64 地址：

```bash
ifconfig en0 | grep inet6
```

您应该能看到两种类型的非链路本地 IPv6 地址：
1. 自动配置的随机地址（带有 `autoconf` 标记）
2. 添加的 EUI-64 格式地址

## 故障排查

### 地址未成功添加

如果未看到 EUI-64 地址，可能的原因包括：
- 脚本权限不足：确保以 root 权限运行脚本
- IPv6 网络不可用：检查你的网络是否支持 IPv6
- 接口名称错误：验证网络接口名称是否正确

### 服务启动失败

如果服务无法正常启动：
- 检查日志文件（如配置了日志输出）查看错误信息
- 确认 Python 3 已正确安装
- 检查脚本路径和权限