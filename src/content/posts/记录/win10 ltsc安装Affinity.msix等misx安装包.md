---
title: win10 ltsc安装Affinity.msix等misx安装包
published: 2026-04-17
description: win10 ltsc安装Affinity.msix等misx安装包
image: api
tags:
  - windows
  - 踩坑
category: 记录
draft: false
---
系统Windows10 ltsc
![[Pasted image 20260426142232.png]]我想安装affinity或可画，但是提供的是msix安装包，于是执行

 `add-appxpackage C:\Users\Administrator\Downloads\Canva.msix`

结果报错
```
add-appxpackage : 部署失败，原因是 HRESULT: 0x800B010A, 无法建立到信任根颁发机构的证书链。 错误 0x800B010A: 应用包或捆绑包中的签名的根证书和所有中间证书必须是受信任的证书。 注意: 有关其他信息，请在事件日志中查找 [ActivityId] 8acc6094-c82a-000e-93ca-cc8a2ac8dc01，或使用命令行 Get-AppPackageLo g -ActivityID 8acc6094-c82a-000e-93ca-cc8a2ac8dc01 所在位置 行:1 字符: 1 + add-appxpackage C:\Users\Administrator\Downloads\Canva.msix + ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ + CategoryInfo : NotSpecified: (C:\Users\Admini...oads\Canva.msix:String) [Add-AppxPackage], Exception + FullyQualifiedErrorId : DeploymentError,Microsoft.Windows.Appx.PackageManager.Commands.AddAppxPackageCommand
```

Affinity：
```
PS C:\Users\Administrator> add-appxpackage D:\Affinityx64.msix
add-appxpackage : 部署失败，原因是 HRESULT: 0x80073CF3, 包无法进行更新、相关性或冲突验证。
Windows 无法安装程序包 Canva.Affinity_3.1.0.4231_x64__8a0j1tnjnt4a4，因为此程序包依赖于一个找不到的框架。请随要安装的此
程序包一起提供由“CN=Microsoft Corporation, O=Microsoft Corporation, L=Redmond, S=Washington, C=US”发布的框架“Microso
ft.VCLibs.140.00.UWPDesktop”(具有中性或 x64 处理器体系结构，最低版本为 14.0.30035.0)。当前已安装的名称为“Microsoft.VC
Libs.140.00.UWPDesktop”的框架为: {Microsoft.VCLibs.140.00.UWPDesktop_14.0.27810.0_x64__8wekyb3d8bbwe}
注意: 有关其他信息，请在事件日志中查找 [ActivityId] 8acc6094-c82a-000e-1ca1-cc8a2ac8dc01，或使用命令行 Get-AppPackageLo
g -ActivityID 8acc6094-c82a-000e-1ca1-cc8a2ac8dc01
所在位置 行:1 字符: 1
+ add-appxpackage D:\Affinityx64.msix
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : WriteError: (D:\Affinityx64.msix:String) [Add-AppxPackage], IOException
    + FullyQualifiedErrorId : DeploymentError,Microsoft.Windows.Appx.PackageManager.Commands.AddAppxPackageCommand
```

是ltsc不支持吗？

---

不小心找到解决了![:rofl:](https://cdn.ldstatic.com/images/emoji/twemoji/rofl.png?v=15 ":rofl:")

https://t2.re/archives/1140/

执行
```
 Add-AppxPackage C:\Users\Administrator\Downloads\Microsoft.VCLibs.x64.14.00.Desktop.appx
```

![[Pasted image 20260426142433.png]]