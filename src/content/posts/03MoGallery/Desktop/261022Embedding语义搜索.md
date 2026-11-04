---
title: Embedding
published: 2026-10-22
description: Embedding
tags:
  - Emulsion
  - SQLite
  - 架构设计
  - embedding
category: MO Gallery
comment: true
---
mlx-community/embeddinggemma-2-bf16

mlx-community/embeddinggemma-2-8bit

onnx-community/embeddinggemma-2-ONNX

>本内容由AI生成。

## 结论概要

语义检索目前只在**桌面端本地资源库**（`emulsion-desktop-v3`）里实现；`mo-gallery-web`、`mo-gallery-offical`、`emulsion-app` 都没有 embedding 相关代码。整体结构是：

**把编码后端与向量语义分开**：算法在可替换的编码后端里，而维度、归一化、向量写入数据库时使用的键和检索排序由宿主独占。查询走**关键词 FTS5 与向量 KNN 的 RRF 混合排序**，不是纯向量检索。

---

## 1. 编码层（后端二选一，接口相同）

宿主定义 `EmbeddingInferenceEngine` 接口（`Text` / `Image` / `Health` / `ModelSpecs`），有两种实现：

| 后端 | 实现位置 | 说明 |
|---|---|---|
| 本地（默认） | `storage_plugins/embedding.go` + `runtime/inference/lib/embedding-*.mjs` | 内置 `embedding@1` 插件子进程，Node + `onnxruntime-node` 跑 ONNX |
| 远程 | [embedding_remote.go](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\embedding_remote.go) | 模型站 HTTP 端点，方言支持 openai / cohere / jina，限流与 5xx 重试 3 次，启动时探维度（维度要进 `model_version`） |

本地模型是 **EmbeddingGemma 2**（`onnx-community/embeddinggemma-2-ONNX`，revision 固定为 `daa72c51…`），规格集中在一处：[embedding-models.mjs](D:\Projects\mo-gallery\emulsion-desktop-v3\runtime\inference\lib\embedding-models.mjs:1)

- 三档权重 q4（默认）/ quantized / fp32，**明确不提供 fp16**（会静默产出 NaN 向量不报错）
- 原生 768 维，MRL 截断档 768/512/256/128，**默认 256 维**写入数据库
- **多模态分两段**：图像按 16×16 patch（默认网格 50、最小 26）进入视觉塔 → 512 维 soft token → 替换文本中的 `<|image|>` 占位符 → 文本塔输出 `sentence_embedding` → 截断 + **重新 L2 归一化**
- 文本强制注入任务前缀（`document` / `searchQuery`），图像路径不加前缀

宿主侧的接缝实现见 [embedding_inference.go](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\embedding_inference.go:16)：懒启动、5 分钟空闲退出、并发闸门、把插件错误码转换成 `EmbeddingInferenceError`、进程失效即丢弃。

## 2. 建索引侧

入口 `StartEmbeddingIndex`（由资源库底栏“开始索引”手动触发，**不跟随扫描自动跑**），见 [embedding_index.go](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\embedding_index.go:303)。

- 输入：图像路径复用 `faceImageForAnalysis`（长边 ≤1024、转 RGB8）；文本路径完全不读文件，用 `display_title + file_name + notes`（标签刻意排除，因为改标签没有可观测信号）
- **两级跳过检查**：① 图像路径比 `file_size + file_mtime`；② 输入 `content_hash`（sha256，含流水线版本和模态），且上次未 failed 才算跳过——failed 行永远重试
- 存储：`asset_embeddings`（向量 BLOB，主键 `asset_id + model_id + model_version`）+ `asset_embedding_status`（`content_hash`/`status`/`dim`/`grid`/`error_text`/mtime），建表见 [store.go](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\store.go:515)
- **空间标识靠派生串**：`model_version = revision + variant + d<dim> + g<grid> + img|txt`（[embeddingModelVersionFrom](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\embedding_index.go:668)）。所以更换档位、维度、网格或插件版本等于换空间，旧向量仍可读取、绝不混用，也不需要额外失效逻辑
- 连续 3 次“致命”推理失败会中止整轮；进度通过 `embedding_index_status` 事件推送（200ms 节流）

## 3. 检索侧

入口是 `AssetQuery.Semantic`（`{text}` 或 `{assetId}`，可选 `dim/modelId/modelVersion` 固定空间），`ListAssets` 检测到非空即走混合分支，不需要新绑定。主干在 [searchAssetsSemantic](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\embedding_search.go:146)：

1. **选空间**：从库里真实存在的 `(model_id, model_version)` 里挑，先过滤到当前后端能派生出的 `model_id`，再按模态选（文字找图优先图像空间，只有文本空间时退到 `txt` 空间），多档并列时取插件默认档，否则取行数最多、再按版本升序——必须确定性
2. **编查询向量**：文本走 `searchQuery` 前缀同一次 `embeddingText`；以图搜图会**现编一帧**（同一条预处理管线），不读索引缓存，避免“这张还没被编到”失效
3. **先固定筛选条件**：文件夹/评分/标签/集合/availability 先在 SQL 里定下候选集，语义只在候选集内重排
4. **语义排名**：读该空间全部向量，在候选集内计算余弦（归一化后就是点积，全表扫 + `sort.SliceStable`，坏行或零向量跳过并计数），取前 200（[embeddingSearchSemanticRank](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\embedding_search.go:459)），没有使用 ANN 索引
5. **关键词排名**：FTS5 `bm25(asset_search)` 前 200，不被语义通道替换
6. **RRF 融合**：`score = Σ 1/(60 + rank)`，并列按 `asset_id` 升序（[embeddingSearchFuse](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\embedding_search.go:579)）
7. **分页回填**：游标值为 RRF 分数，`sort.Search` 二分定位下一页起点；取行复用列表页那条 `listAssets` 路径再按融合顺序重排

**降级策略**（[listAssetsSemantic](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\embedding_search.go:105)）：插件不可用 / 模型未下 / 库里没可比空间 / 空间不是当前后端编的 → 文本查询**静默回落纯 FTS5**；以图搜图不回落（没有等价关键词版本，宁可报原因）。严格入口 `SearchAssetsSemantic` 则把可判定的错误原样抛出。以图搜图时查询图自身从候选集剔除，否则第一名永远是刚右键的那张。

另外还有一条轻量方案（不走模型）：`PHashSimilarQuery` 按 64 位感知哈希汉明距离找近重复，见 [near_duplicate.go](D:\Projects\mo-gallery\emulsion-desktop-v3\local_library\near_duplicate.go)，与 embedding 以图搜图互补。

## 4. 前端与设置

- [semantic-sort.ts](D:\Projects\mo-gallery\emulsion-desktop-v3\frontend\src\features\library\local\embedding\semantic-sort.ts)：判定“语义通道是否就绪”（插件/模型/已存向量数），开关打开但通道掉线时不会硬发 `semantic`
- `SemanticSearchToggle`（搜索框旁开关）、`EmbeddingIndexControl`（索引进度/开始/取消）
- 设置页 `EmbeddingBackendSection`：本地/远程后端切换、精度档位、远程端点“测试连接”
- `embedding_overview.go`：向量总览/按资产查看向量（向量浏览器）

## 5. 设计依据

选型与全部硬约束（MRL 截断后必须重新归一化、禁 fp16、query 与库内必须同维同空间）写在 [embeddinggemma-2-design.md](D:\Projects\mo-gallery\.trellis\spec\emulsion-desktop\local-ai\embeddinggemma-2-design.md:29)，代码注释与它逐条对齐。

---

**说明**：以上结论来自阅读源码与设计文档，我没有运行 `go test ./local_library` 或前端构建（该包有 `embedding_search_test.go`、`embedding_index_test.go`、`embedding_e2e_test.go` 覆盖上述路径，需要可真跑一遍验证）。如果你问的是某个具体仓库（比如官网的模板/创意广场检索），那边目前是关键词实现，没有向量检索。

