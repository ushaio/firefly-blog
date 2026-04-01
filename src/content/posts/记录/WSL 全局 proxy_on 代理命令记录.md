---
title: WSL 全局 proxy_on 代理命令记录
published: 2026-03-24
description: 记录 WSL 中全局 proxy_on、proxy_off、proxy_status 命令的添加背景、作用、添加方法和配置位置
image: "api"
tags: [代理, WSL, Docker]
category: 记录
draft: false
---

这篇笔记记录一下在 WSL 中添加 `proxy_on`、`proxy_off`、`proxy_status` 这组全局命令的背景、作用、添加方式，以及它们实际写在什么位置。后面如果又碰到 `docker compose`、`git clone`、`curl`、`apt` 在 WSL 里走不通代理，不至于一脸懵地到处翻历史记录。

## 背景

Windows 侧开了代理软件之后，浏览器和部分桌面应用能直接走代理，但 WSL 里的命令行环境并不会自动继承这些设置。

这就会带来一个很典型的场景：

- Windows 上代理已经可用；
- WSL 里执行 `docker compose`、`docker pull`、`git clone`、`curl`、`pip` 或 `apt` 时，依然连不上外网；
- 手动 `export http_proxy=...` 虽然能顶一会儿，但每次开新 shell 都得重来一遍，纯属重复劳动。

更麻烦的是，WSL 访问 Windows 宿主机代理时，通常不能直接写死一个固定 IP。因为 WSL 的默认网关地址会随着网络环境变化而变化，所以更稳妥的做法是每次动态获取默认网关，再把代理变量指过去。

## 作用

这组命令的作用很直接：

- `proxy_on`：自动获取当前 WSL 默认网关地址，并设置 `http_proxy`、`https_proxy`、`all_proxy` 及其大写环境变量；
- `proxy_off`：清理这些代理环境变量；
- `proxy_status`：查看当前 shell 中代理变量是否已经生效。

如果 Windows 宿主机上的代理监听端口是 `7890`，那么执行 `proxy_on` 后，通常会看到类似下面的输出：

```bash
Proxy enabled via 172.26.128.1:7890
```

这说明当前 shell 已经把代理指向了 Windows 宿主机在 WSL 网络中的地址。

## 命令内容

当前使用的函数定义如下：

```bash
proxy_on() {
  local host_ip
  host_ip=$(ip route | awk '/default/ {print $3}')
  export http_proxy="http://${host_ip}:7890"
  export https_proxy="http://${host_ip}:7890"
  export all_proxy="socks5://${host_ip}:7890"
  export HTTP_PROXY="$http_proxy"
  export HTTPS_PROXY="$https_proxy"
  export ALL_PROXY="$all_proxy"
  echo "Proxy enabled via ${host_ip}:7890"
}

proxy_off() {
  unset http_proxy https_proxy all_proxy
  unset HTTP_PROXY HTTPS_PROXY ALL_PROXY
  echo "Proxy disabled"
}

proxy_status() {
  echo "http_proxy=${http_proxy:-<empty>}"
  echo "https_proxy=${https_proxy:-<empty>}"
  echo "all_proxy=${all_proxy:-<empty>}"
}
```

这里最关键的一行是：

```bash
host_ip=$(ip route | awk '/default/ {print $3}')
```

它会读取当前 WSL 的默认路由网关，也就是 Windows 宿主机在 WSL 里的可访问地址。这样代理地址不是写死的，适应性会比硬编码强不少。

## 添加方法

如果要手动添加这组命令，可以把上面的函数追加到当前用户的 `~/.bashrc` 中。

示例步骤如下：

```bash
nano ~/.bashrc
```

把下面这段追加到文件末尾：

```bash
proxy_on() {
  local host_ip
  host_ip=$(ip route | awk '/default/ {print $3}')
  export http_proxy="http://${host_ip}:7890"
  export https_proxy="http://${host_ip}:7890"
  export all_proxy="socks5://${host_ip}:7890"
  export HTTP_PROXY="$http_proxy"
  export HTTPS_PROXY="$https_proxy"
  export ALL_PROXY="$all_proxy"
  echo "Proxy enabled via ${host_ip}:7890"
}

proxy_off() {
  unset http_proxy https_proxy all_proxy
  unset HTTP_PROXY HTTPS_PROXY ALL_PROXY
  echo "Proxy disabled"
}

proxy_status() {
  echo "http_proxy=${http_proxy:-<empty>}"
  echo "https_proxy=${https_proxy:-<empty>}"
  echo "all_proxy=${all_proxy:-<empty>}"
}
```

保存后执行：

```bash
source ~/.bashrc
```

然后就可以直接使用：

```bash
proxy_on
proxy_status
proxy_off
```

## 配置位置

这次实际检查到的配置位置是：

```bash
/root/.bashrc
```

并且函数定义出现在该文件大约第 `102` 行开始的位置。

这意味着：

- 这些命令是 **root 用户的全局 shell function**；
- 它们和某个具体项目无关；
- 只要进入的是 root 用户的交互式 `bash`，就能直接调用；
- 如果换成其他用户登录 WSL，这些函数未必可用，因为它们通常只写在当前用户自己的 `~/.bashrc` 里。

## 使用场景

这组命令比较适合下面这些场景：

- 在 WSL 中执行 `docker compose`、`docker pull`；
- 在 WSL 中使用 `git clone` 拉取国外仓库；
- 使用 `curl`、`wget`、`pip`、`npm`、`apt` 等命令访问外网；
- 临时开启代理，任务结束后再执行 `proxy_off` 清理环境变量。

如果你是在 `root@...` 下工作，这种写法最省事；但如果你的日常开发用户不是 root，最好把同样的函数放到对应用户的 `~/.bashrc` 里，不然切个用户就失效了。

## 验证方法

可以用下面这些方式验证：

先看变量是否已经写入当前 shell：

```bash
proxy_status
```

再用一个简单请求确认：

```bash
proxy_on
curl -I https://www.google.com
```

如果请求能通，说明代理变量已经生效。

## 结论

`proxy_on` 这套命令本质上就是给 WSL 命令行环境补上一层“临时代理开关”。它的核心价值不是功能多复杂，而是把重复的手动 `export` 动作收敛成了几个短命令，而且通过动态获取默认网关，避免了把代理地址硬编码死。

真要说，这种写法不算什么高深技巧，但胜在实用。命令行环境最怕的不是不会配，而是每次都得重新配，配着配着人就烦了。有了这几个函数，至少少骂两句机器。

