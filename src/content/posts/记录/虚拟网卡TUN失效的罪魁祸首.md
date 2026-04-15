---
title: 虚拟网卡TUN失效的罪魁祸首
published: 2026-04-12
description: 虚拟网卡TUN失效的罪魁祸首
image: api
tags:
  - 代理
  - 虚拟网卡
category: 记录
draft: false
---
自从重装系统到win10 ltsc发现太丝滑了，干啥都很快，FlClash 的虚拟网卡也正常如初了，心中激动万分，暗暗道再也不乱搞了，一定好好对你，我的爱机！

但安装OPPO互联后，震惊发现，虚拟网卡瞬间失效了，心中突然一凉，不会又给我整坏了吧，但好在也是发现具体啥原因了。于是尝试卸载OPPO互联，但是还是timeout，最后重启电脑后与恢复正常了，心里终于松了口气。

剩下的就是寻找啥原因了，直接Google了下”OPPO 互联导致虚拟网卡失效“，没想到已经有前人踩过坑了，太感动了，不然不知道走多少弯路，甚至还要找其他替代。


https://github.com/clash-verge-rev/clash-verge-rev/discussions/6031

最后，只要把文件重命名一下或者删了即可
![[Pasted image 20260412170746.png]]