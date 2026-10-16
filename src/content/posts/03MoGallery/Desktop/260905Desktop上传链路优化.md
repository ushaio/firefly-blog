---
title: Desktop上传链路优化
published: 2026-09-05
description: Desktop上传链路优化
image: "api"
tags: [MO Gallery, Desktop]
category: MO Gallery
draft: false
---
上传链路耗时分析：

| 阶段             | 耗时  | 说明                           |
| ---------------- | ----- | ------------------------------ |
| 压缩             | 76.9s | 7 次全图编码，详见下文         |
| 原图上传 1.9MB   | 0.71s | 8 块，2.65 MB/s，正常          |
| 缩略图 AVIF 编码 | 3.98s | 512px 小图编码 3.76s，异常偏大 |
| 缩略图上传 64KB  | 0.42s | 正常                           |
| 服务器 register  | 3.72s | 服务器端处理慢                 |
| 冷启动（一次性） | 0.21s | 无问题                         |

## 压缩 77 秒的三个原因（按浪费程度排序）

1. **逻辑缺陷：级别重试永远不可能成功，每轮 2/3 时间纯浪费**。`CompressToWebP` 每轮按 BestCompression → Default → BestSpeed 顺序全试。但压缩级别越高输出越小——**如果 `BestCompression`（level 6）都超目标，后两个更大的输出必然也超**。日志印证：round 0 三个级别输出 12.23/12.43/12.59MB，全部失败，54 秒里后两次编码（35 秒）是注定无用的。round 1 同样浪费了 10 秒。仅此一项就浪费约 45 秒。
2. **编码器与内容不匹配：纯 Go 无损 WebP 编照片**。`nativewebp` 是 VP8L 无损编码器，11MP 照片输出 12.2MB（≈1.1 字节/像素），离 2MB 目标差 6 倍，只能靠不断缩图硬凑。最终为了达标把 4096px 缩到了 **1493×994**——分辨率损失远比有损编码大。若用有损编码（WebP 有损或 AVIF），4096px 原尺寸就能压进 2MB，速度快一个数量级以上。
3. **缩图步长下限 0.55 过于保守**。round 0 失败后按 `sqrt(2/12.2)×0.92≈0.37` 计算本应一次缩到位（约 1700px），但被 `ratio = max(0.55, ...)` 钳制到 0.55，只缩到 2252px，白白多跑了一整轮（15 秒）。没有 0.55 下限的话 2 轮就够了。

## 其他两个异常

- **缩略图 AVIF 3.76s**：512px @ Speed 8 不应该这么慢，gen2brain/avif（libaom）在这台机器上的编码路径值得单独查，但优先级低于压缩。
- **register 稳定 3.7–4.1s**：两次上传都是这个量级，说明是服务器端处理（Postgres 写入或事务）固定耗时，与桌面端无关，可单独排查。

## 结论

一次上传 86 秒里：约 60 秒是上面 1+3 的纯浪费（注定失败的编码），其余 17 秒是无损编码器本身太慢，真正必要的传输只有约 1.1 秒。如果改为“按输出大小预估目标尺寸一次缩放到位 + 有损编码 + 去掉注定失败的级别重试”，压缩大概率能从 77 秒降到 5 秒以内，整次上传进入 10 秒档。要动手改的话告诉我。





----

# 优化方案

按“收益/改动量”排序，前两条改完基本就能把 77 秒打到秒级，后面是锦上添花。

## ~~0. 先问一个问题：这张图需不需要压缩？~~

原图 1.9MB，压缩目标 2MB——**原图本身已经达标**。如果“压缩版”的语义只是“≤2MB、≤4096px 的可分发版本”，那这次上传的正确做法是直接复用原图，压缩耗时应该是 0。

建议加一个前置短路：

Go



```
if origSize <= targetBytes && maxSide(origW, origH) <= maxDim && isWebFriendly(format) {
    return original, nil // 跳过压缩
}
```

这一条改动 5 行，覆盖大量手机/相机直出的 JPEG。如果产品语义确实要求统一转成 WebP/AVIF，再往下看。

## ~~1. 换编码器：照片用有损，不要用 `nativewebp`~~

`nativewebp` 只有 VP8L 无损，对照片天然 ~1 byte/px，怎么调都不会好。方案按改动量：

| 方案                                      | 11MP 编码耗时（估） | 2MB 下能保留的分辨率 | 备注                             |
| ----------------------------------------- | ------------------- | -------------------- | -------------------------------- |
| 标准库 `image/jpeg` q80                   | ~0.3s               | 4096px 全尺寸        | 零依赖，最稳                     |
| cgo libwebp 有损（`kolesa-team/go-webp`） | ~0.5s               | 4096px               | 需要 cgo 工具链                  |
| `gen2brain/webp`（wasm libwebp 有损）     | ~1–2s               | 4096px               | 纯 Go 可交叉编译，但有 wasm 开销 |
| 保留无损（截图/PNG 等）                   | 按需                | —                    | 仅对非照片内容                   |

按源格式分流：`image/jpeg`、HEIC 等照片走有损；PNG/截图（色彩数少）才走无损。

## ~~2. 重写尺寸/质量决策：一次估准，最多重试 2 次~~

核心思路：**有损编码下，输出大小 ≈ 像素数 × bpp**，bpp 在固定质量下很稳定，所以第一次编码结果就能直接算出目标尺寸。同时**先降质量再缩图**，分辨率比质量值钱得多。

Go



```
func CompressToTarget(src image.Image, maxDim int, target int) ([]byte, error) {
    img := fitWithin(src, maxDim) // 用 BiLinear/Box，别用 CatmullRom 缩 11MP

    qualities := []int{85, 75, 65}
    for round := 0; round < 3; round++ {
        for _, q := range qualities {
            buf := encodeLossy(img, q)
            if buf.Len() <= target {
                return buf.Bytes(), nil
            }
            // 质量降到 65 仍超标才缩图；跳出去按比例缩
            if q == qualities[len(qualities)-1] {
                ratio := math.Sqrt(float64(target)/float64(buf.Len())) * 0.92
                img = scale(img, ratio) // 不设 0.55 下限，最多钳一个 0.25 防极端
            }
        }
    }
    return nil, errCannotFit
}
```

同时修掉原逻辑的两个问题：

- **删掉 BestCompression→Default→BestSpeed 的降级重试**。无论有损无损，同一尺寸下更低压缩级别只会更大。如果保留无损路径，就只试最高级别一次，超标直接进缩图。
- **去掉 `max(0.55, ratio)` 钳制**。这个下限的初衷大概是怕一次缩太狠，但 0.92 安全系数已经够了；真怕就用 0.25，并且缩完第一次就应该命中。

预期：11MP 照片 → 缩到 4096px（0.2s）→ q85 编码（0.3s）→ 大概率 1.5–2.5MB，最坏再编一次 q75，**总耗时 < 1.5s**。

## 3. 检查隐藏成本：解码与缩放

看日志里每轮 15–18 秒，其中不一定全是编码。确认：

- **原图只解码一次**，`CompressToWebP` 各轮复用同一个 `image.Image`，而不是每轮 `image.Decode`。
- **缩放算法**：`golang.org/x/image/draw` 的 `CatmullRom` 缩 11MP 要 2–4 秒。改成 `ApproxBiLinear`，或先按整数倍 `Box` 缩到 2 倍目标、再 `BiLinear` 到最终尺寸，视觉差别很小，速度差 10 倍。
- **像素格式**：`image.YCbCr` 直接喂给编码器会触发逐像素转换，先转 `*image.RGBA`/`*image.NRGBA` 一次，后续所有编码复用。

## ~~4. 缩略图 3.76s：大概率是 wasm 冷启动，不是 libaom 慢~~

`gen2brain/avif` 是 libavif+aom 编译成 WebAssembly 后用 wazero 跑的，**第一次调用要编译 wasm 模块，通常 1–3 秒**，之后才是真正的编码。三个选法：

1. **最简单**：缩略图改用 JPEG（q80，512px 约 30–60KB）或 WebP，512px 的 JPEG 编码只要几毫秒。缩略图的体积收益本来就小，AVIF 在这里不值。
2. 坚持 AVIF：进程启动时后台 `warmup` 一次（编一张 1×1），把 wasm 编译成本移出上传链路；并确认没有每次上传都重新 `Encode` 初始化。
3. 顺带检查缩略图是不是从 11MP 用 CatmullRom 直接缩到 512——这一步本身就可能 1–2 秒。改成从压缩版（4096px）再缩，或用上面说的两段式缩放。

## 5. 并发化上传链路

现在是串行：压缩 → 传原图 → 编缩略图 → 传缩略图 → register。三条数据流互不依赖：

text



```
decode ─┬─ upload original (0.7s)
        ├─ compress → upload compressed
        └─ thumbnail → upload thumb (0.4s)
        └───────────── all done ──→ register
```

用 `errgroup` 并发，总耗时变成 `max(三条) + register`。压缩降到 1.5s 后，客户端侧总时间 ≈ 1.5–2s。

## 6. Register 3.7s：服务器端单独排查

两次都 3.7–4.1s、与图片大小无关，说明是固定成本，典型嫌疑：

- 事务里同步做了重活：解析 EXIF、生成派生数据、调外部服务、对对象存储做 HEAD/校验。
- 等待某个锁或串行队列。
- Postgres 侧：缺索引导致的全表扫描、`synchronous_commit` + 慢盘、连接池耗尽等待。

处理方式：给 register handler 加分段计时日志定位，然后把非必要工作改成 **写行 + 入队异步处理，立即返回 202**。目标 < 200ms。

## 预期效果

| 阶段         | 现在     | 优化后                  |
| ------------ | -------- | ----------------------- |
| 压缩         | 76.9s    | 0（原图已达标）或 ~1.5s |
| 缩略图编码   | 3.98s    | < 0.1s                  |
| 上传（并发） | 1.13s    | ~0.7s（与压缩并行）     |
| register     | 3.72s    | < 0.3s                  |
| **合计**     | **~86s** | **~2–3s**               |

建议实施顺序：先做 0 和 2（纯逻辑修改，半天内完成，立即去掉 60 秒浪费），再做 1 和 4（换编码器），5 和 6 可以并行由服务端同学推进。