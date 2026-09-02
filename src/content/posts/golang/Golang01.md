---
title: Golang01
published: 2026-04-10
description: 核心算法策略
image: api
tags:
  - java
  - 算法
category: java
draft: false
---
![[Pasted image 20260617171511.png]]
这个提示的意思是：你的 Go 编辑器/插件想调用 gopls，但系统里找不到这个命令。

  gopls 是 Go 官方语言服务器，主要给 VS Code、GoLand、Neovim 等编辑器提供代码补全、跳转定义、重命名、格式化、诊断等功能。提示里的
  命令：
```
go install -v golang.org/x/tools/gopls@latest
```

就是安装最新版 gopls。

然后，添加至环境变量：
```
C:\Users\Administrator\go\bin
```

## 变量定义

```go
func main() {

    // 先声明再赋值

    var name string

    name = "shai"

    fmt.Println(name)

  

    // 声明并赋值

    var name1 string = "shaiii"

    fmt.Println(name1)

  

    // 省略类型，自动判断

    var name2 = "SHAI"

    fmt.Println(name2)

}
```