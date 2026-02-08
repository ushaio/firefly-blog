---
title: Flclash开启虚拟网卡，在wsl中不生效
published: 2026-02-08
description: Flclash开启虚拟网卡，在wsl中不生效而无法访问Google的其他解决方案
image: ./images/firefly1.webp
tags: [代理]
category: 记录
draft: false
---

---

使用Flclash开启虚拟网卡，想wsl也使用Windows的代理的，但是发现不生效，生效的前提是开启系统代理，但这样就有违该设置了。



wlsconfig配置：

```
[wsl2]
networkingMode=Mirrored
autoProxy=false
firewall=false
processors=8
memory=16GB
swap=0
dnsTunneling=false
```

Flclash配置

![image](./../../images/Wsl对Flclash的tun模式无效问题/2dd1861763808abbca81f30df7802f546072177b_2_302x500.png)

其他测试：

![image](./../../images/Wsl对Flclash的tun模式无效问题/83a2d6813ca4f6244a5b8ce2d1c6c32f3654a9e1_2_690x306.png)



## 最后解决方案：

**临时方案：**

同时开启系统代理



**一些参考性的回答：**

![image-20260208094303378](./../../images/Wsl对Flclash的tun模式无效问题/image-20260208094303378.png)

![image-20260208094310254](./../../images/Wsl对Flclash的tun模式无效问题/image-20260208094310254.png)



**建设性回答：**

flclash不能监听0.0.0.0,只接受127.0.0.1 走局域网代理的话走不通，后面换成了clash party [请问如何将监听地址改为0.0.0.0？ · Issue #1037 · chen08209/FlClash](https://github.com/chen08209/FlClash/issues/1037)



最后也确实，换了clash party后单独开启虚拟网卡也解决了。
