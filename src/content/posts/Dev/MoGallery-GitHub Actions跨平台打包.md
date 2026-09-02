---
title: MoGallery-GitHub Actions跨平台打包
published: 2026-08-11
description: "MoGallery Desktop 使用 GitHub Actions 构建 Windows、macOS 和 Linux 安装包的配置记录"
image: "api"
tags: ["mo-gallery", "GitHub Actions", "Wails", "开发笔记"]
category: Dev
draft: false
---

## 当前发布产物

MoGallery Desktop 使用 Wails 2.12.0。Release workflow 在各平台的原生 GitHub runner 上构建，当前每个版本共生成 23 个桌面端文件。

| 平台 | 架构 | 格式 |
|---|---|---|
| Windows | AMD64、ARM64、x86 | Portable `.exe`、NSIS Setup `.exe` |
| macOS | AMD64、ARM64、Universal | `.zip`、`.dmg`、`.pkg` |
| Linux | AMD64、ARM64 | `.tar.gz`、`.deb`、`.rpm`、`.AppImage` |

Windows x86 对应 Wails 目标 `windows/386`。macOS Universal 同时包含 Intel 和 Apple Silicon 代码。

## Workflow 触发与版本限制

Release workflow 在以下情况触发：

- 推送到 `master` 分支；
- 在 GitHub Actions 页面手动执行 `workflow_dispatch`。

版本号从 `RELEASE.md` 第一条 `## vX.Y.Z` 标题读取，并要求以下文件的版本保持一致：

- 根目录 `package.json`；
- `desktop/frontend/package.json`；
- `desktop/wails.json` 中的 `info.productVersion`；
- `flutter/pubspec.yaml`。

如果对应 Git tag 已存在，所有平台构建和 Release 发布都会跳过。例如远端已有 `v0.8.0` 时，必须升级到 `0.8.1` 等新版本才能重新触发发布。

## Windows 打包

每个 Windows 架构生成两种文件：

```text
mo-gallery-desktop-版本-windows-架构-portable.exe
mo-gallery-desktop-版本-windows-架构-setup.exe
```

Portable 文件无需安装；Setup 文件使用 NSIS 生成，提供正常的安装与卸载流程。

## macOS 打包

每个 macOS 架构生成三种文件：

- `.zip`：包含 `.app` 应用包；
- `.dmg`：带有 Applications 快捷方式的磁盘映像；
- `.pkg`：安装到 `/Applications` 的安装包。

### 签名与公证 Secrets

正式对外发布需要在 GitHub 仓库的 `Settings -> Secrets and variables -> Actions` 中配置：

| Secret | 用途 |
|---|---|
| `MACOS_CERTIFICATE_P12_BASE64` | Apple Developer `.p12` 证书的 Base64 内容 |
| `MACOS_CERTIFICATE_PASSWORD` | 导出 `.p12` 时设置的密码 |
| `MACOS_SIGNING_IDENTITY` | `Developer ID Application` 签名身份 |
| `MACOS_INSTALLER_IDENTITY` | `Developer ID Installer` 签名身份 |
| `APPLE_ID` | Apple Developer 账户邮箱 |
| `APPLE_TEAM_ID` | Apple Developer Team ID |
| `APPLE_APP_SPECIFIC_PASSWORD` | Apple ID 的 App 专用密码 |

处理流程如下：

```text
构建 .app
  -> codesign 签名
  -> notarytool 提交 Apple 公证
  -> stapler 附加公证凭证
  -> 生成 ZIP、DMG 和 PKG
```

未配置 Secrets 时，workflow 仍会生成 ZIP、DMG 和 PKG，但它们未签名、未公证，只适合内部测试。其他 Mac 用户打开时可能遇到 Gatekeeper 的“无法验证开发者”提示。

## Linux 打包

Linux 每个架构生成：

- `.tar.gz`：原始可执行文件归档；
- `.deb`：Debian、Ubuntu 系发行版安装包；
- `.rpm`：Fedora、RHEL 系发行版安装包；
- `.AppImage`：单文件分发格式。

Wails Linux 程序仍依赖 GTK3 和 WebKitGTK 4.1。不同发行版安装包名称可能不同，出现运行库缺失时应优先通过系统包管理器安装对应依赖。

## 发布前检查

1. 同步所有项目版本号和 `RELEASE.md`。
2. 确认目标 Git tag 尚不存在。
3. 正式发布 macOS 时确认 Apple Secrets 已配置。
4. 推送到 `master` 或手动运行 Release workflow。
5. 检查各矩阵 job 成功，并确认 GitHub Release 包含全部预期文件。

