---
title: Clash虚拟网卡（TUN 模式）重启后需要重复操作ICS 的问题分析
published: 2026-02-08
description: Clash 虚拟网卡（TUN 模式）重启后需要重复操作ICS的问题分析
image: "api"
tags: [代理]
category: 记录
draft: false
---





> **电脑开启clash的虚拟网卡功能时必须得在 ‘控制面板\网络和 Internet\网络连接’把虚拟网卡打开：属性>共享>Internet连接共享勾选开启，在选择家庭网络后点击确定，再取消勾选Internet连接共享勾选开启点击确定后才能正常联网，重启后又得重新执行上述操作，这是什么原因？如何解决？**

> AI回复，仅供参考！

# Clash 虚拟网卡（TUN 模式）重启后需要重复操作ICS的问题分析

## 问题原因

这个问题的本质是 **Windows 网络接口的路由/度量值（Metric）优先级冲突**。

### 具体机制：

1. **Clash TUN 模式**会创建一个虚拟网卡（通常名为 `Clash` 或 `clash0`），并通过修改路由表将所有流量导向这个虚拟网卡
2. 但 Windows 重启后，虚拟网卡的**接口度量值（Interface Metric）**可能高于物理网卡，导致路由优先级不正确
3. 你做的"开启 ICS → 选择家庭网络 → 确定 → 再关闭 ICS → 确定"这套操作，**本质上是触发了 Windows 重新计算网络接口的路由表和度量值**，相当于"重置"了一下网络栈，让路由顺序恢复正常

简单说：**你并不是真的需要 ICS 共享功能，只是借助它的开关操作来刷新路由表。**

------

## 解决方案

### 方案一：手动固定接口度量值（推荐）

1. 打开 `控制面板 → 网络和 Internet → 网络连接`
2. 找到你的 **物理网卡**（以太网/Wi-Fi），右键 → 属性 → `Internet 协议版本 4 (TCP/IPv4)` → 属性 → 高级
3. **取消勾选**"自动跃点"，手动设置接口跃点数为 **较高值**（如 `700`）
4. 对 **Clash 虚拟网卡** 做同样操作，设置跃点数为 **较低值**（如 `1` 或 `5`）

> 跃点数越小，优先级越高。这样确保 Clash 虚拟网卡始终优先接管流量。

### 方案二：写一个开机自动执行的脚本

如果你不想改度量值，可以创建一个脚本来自动刷新路由，效果等同于你手动操作 ICS：

**创建 `fix-clash-tun.bat`：**



```bat
@echo off
:: 重启 Clash 虚拟网卡来刷新路由
netsh interface set interface "Clash" admin=disable
timeout /t 2 /nobreak >nul
netsh interface set interface "Clash" admin=enable
:: 或者直接刷新路由表
route /f
ipconfig /release
ipconfig /renew
```

> ⚠️ 将 `"Clash"` 替换为你实际的虚拟网卡名称（在网络连接中查看）。

**设置开机自动运行**：将此脚本加入「任务计划程序」，触发器选"用户登录时"，勾选"使用最高权限运行"。

### 方案三：在 Clash 配置中启用 `auto-route` 和 `auto-detect-interface`

在 Clash（Mihomo）的配置文件中确保 TUN 部分有以下设置：



```yaml
tun:
  enable: true
  stack: system      # 或 gvisor / mixed
  auto-route: true
  auto-detect-interface: true
  dns-hijack:
    - any:53
```

- **`auto-route: true`**：自动配置路由规则，不依赖 Windows 默认的度量值
- **`auto-detect-interface: true`**：自动检测出站网卡

如果你用的是 **Clash Verge Rev** 或 **mihomo-party** 等 GUI，在「设置 → TUN 模式」中通常有对应的开关。

### 方案四：使用 `system` 栈替代或切换 TUN 栈类型

如果当前用的是 `gvisor` 栈，可以尝试切换为 `system` 或 `mixed`，不同栈对 Windows 路由的处理方式不同，`system` 栈通常兼容性更好。

------

## 总结

| 方案            | 难度    | 效果                  |
| --------------- | ------- | --------------------- |
| 固定接口度量值  | ⭐ 简单  | 根治，重启后无需操作  |
| 开机脚本        | ⭐⭐ 一般 | 自动化当前的手动操作  |
| 配置 auto-route | ⭐ 简单  | 让 Clash 自己管理路由 |
| 切换 TUN 栈     | ⭐ 简单  | 可能直接解决          |

**最推荐**：先试 **方案三**（检查 Clash 配置的 auto-route），再试 **方案一**（固定度量值），这两个组合基本能根治问题。



# 最终解决方案：

```bat
@echo off
:: ============================================
:: Auto fix Mihomo TUN network route
:: Simulate: mihomo ICS enable (share to WLAN) then disable
:: ============================================

set NIC_NAME=mihomo
set HOME_NIC=WLAN
set LOG_FILE=D:\fix-network.log

echo [%date% %time%] Event detected, checking NIC... >> "%LOG_FILE%"

:: Check if mihomo NIC is up
powershell -ExecutionPolicy Bypass -Command "if ((Get-NetAdapter -Name '%NIC_NAME%' -ErrorAction SilentlyContinue).Status -eq 'Up') { exit 0 } else { exit 1 }"
if %errorlevel% neq 0 (
    echo [%date% %time%] NIC "%NIC_NAME%" not connected, skip. >> "%LOG_FILE%"
    goto END
)

echo [%date% %time%] NIC "%NIC_NAME%" is up, wait 8s... >> "%LOG_FILE%"
timeout /t 8 /nobreak >nul

echo [%date% %time%] Enable ICS: %NIC_NAME% share to %HOME_NIC%... >> "%LOG_FILE%"

powershell -ExecutionPolicy Bypass -Command ^
  "regsvr32 /s hnetcfg.dll; " ^
  "$m = New-Object -ComObject HNetCfg.HNetShare; " ^
  "$pubConn = $null; " ^
  "$prvConn = $null; " ^
  "foreach ($c in $m.EnumEveryConnection) { " ^
  "  $props = $m.NetConnectionProps($c); " ^
  "  if ($props.Name -eq 'mihomo') { $pubConn = $c }; " ^
  "  if ($props.Name -eq 'WLAN') { $prvConn = $c }; " ^
  "}; " ^
  "if ($pubConn -and $prvConn) { " ^
  "  $pubCfg = $m.INetSharingConfigurationForINetConnection($pubConn); " ^
  "  $prvCfg = $m.INetSharingConfigurationForINetConnection($prvConn); " ^
  "  $prvCfg.EnableSharing(1); " ^
  "  $pubCfg.EnableSharing(0); " ^
  "  Write-Host 'ICS enabled: mihomo -> WLAN'; " ^
  "  Start-Sleep -Seconds 3; " ^
  "  $pubCfg.DisableSharing(); " ^
  "  $prvCfg.DisableSharing(); " ^
  "  Write-Host 'ICS disabled'; " ^
  "} else { " ^
  "  Write-Host 'ERROR: NIC not found'; " ^
  "  if (-not $pubConn) { Write-Host 'mihomo not found' }; " ^
  "  if (-not $prvConn) { Write-Host 'WLAN not found' }; " ^
  "}"

timeout /t 3 /nobreak >nul
ipconfig /flushdns

echo [%date% %time%] Fix completed! >> "%LOG_FILE%"

:END
echo.
echo ========================================
echo   Done! Check log: D:\fix-network.log
echo ========================================
pause
exit /b 0

```

