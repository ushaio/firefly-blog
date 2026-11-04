---
title: FTS5
published: 2026-11-04
description: 从倒排索引、影子表到增量更新，理解 SQLite FTS5 如何支撑本地资源库的关键词搜索。
tags:
  - wiki
  - SQLite
  - 全文检索
category: 数据库
draft: true
---

## FTS5 是什么

**FTS5（Full-Text Search 5）是 SQLite 的全文搜索扩展**。它将文本拆分为可检索的单元，并建立专门的索引，让应用能够按关键词查找记录，还能进行短语匹配、前缀查询和相关性排序。

它直接运行在 SQLite 数据库中，不需要另起一个搜索服务，适合本地笔记、文章存档、文档管理和资源库等场景。是否能够使用 FTS5，取决于应用使用的 SQLite 是否编译或加载了这个扩展。

在本地资源库里，它负责搜索的是**描述资源的文字**：文件名、路径、标题、备注、标签、集合名称，以及相机和镜头型号。图片本身的视觉内容需要另外的图像分析或向量检索能力。

## 为什么需要它：从逐行匹配到倒排索引

假设资源库里存了几万张照片，希望输入一个相机型号，就能找到相关照片。最直接的做法是：

```sql
SELECT id, file_name
FROM assets
WHERE file_name LIKE '%Hasselblad%'
   OR notes LIKE '%Hasselblad%';
```

这种写法容易理解，但有两个局限：

1. **任意位置的子串搜索通常难以利用普通 B-tree 索引。** 对于这种以 `%` 开头的模式，SQLite 通常需要扫描候选记录；再跨标签、集合和 EXIF 搜索，还要处理关联数据。
2. **文本包含某个字符串，不等于已经具备全文检索能力。** 多个词如何组合、是否连续出现、如何比较相关程度，都需要额外实现。

`LIKE` 仍然适合小规模筛选或简单模式匹配。在满足条件时，`LIKE 'Hass%'` 这样的前缀查询也可能利用普通索引；不能把所有 `LIKE` 查询都说成全表扫描。

FTS5 的核心思路是建立**倒排索引**。普通记录告诉我们“一条资源有哪些文字”，倒排索引则记录“某个词出现在哪些资源中”。例如：

| 资源行号 | 标签 | 相机 |
| --- | --- | --- |
| 1 | mountain | Hasselblad |
| 2 | beach | Sony |
| 3 | mountain | Sony |

可以将索引的逻辑关系理解为：

```text
mountain   → 资源 1、资源 3
beach      → 资源 2
hasselblad → 资源 1
sony       → 资源 2、资源 3
```

查询 `mountain` 时，可以从索引中查到匹配行号；查询 `mountain AND sony` 时，则组合两个词的匹配结果，得到资源 3。

实际索引还会根据配置记录词所在的列、位置等信息，以支持短语查询和相关性计算。它避免了每次搜索都重新遍历并处理全部原文，但依然需要付出索引存储、更新和结果排序的成本。实际耗时应以数据规模、查询条件和设备上的测量为准。

## 一个最小示例

下面的 SQL 可以在启用了 FTS5 的 SQLite 中运行：

```sql
CREATE VIRTUAL TABLE docs USING fts5(title, body);

INSERT INTO docs(rowid, title, body) VALUES
  (1, 'FTS5 introduction', 'SQLite full text search'),
  (2, 'SQLite storage', 'Transactions and local files'),
  (3, 'Local search', 'Full text search for photos');

-- 在所有索引列中查找 sqlite，命中第 1、2 条。
SELECT rowid, title
FROM docs
WHERE docs MATCH 'sqlite';

-- 要求两个词都出现，命中第 1 条。
SELECT rowid, title
FROM docs
WHERE docs MATCH 'sqlite AND search';

-- 短语查询：要求词序和位置连续，命中第 1、3 条。
SELECT rowid, title
FROM docs
WHERE docs MATCH '"full text"';

-- 词的前缀查询，命中第 2 条。
SELECT rowid, title
FROM docs
WHERE docs MATCH 'trans*';

-- 限定只搜索 title 列，命中第 2 条。
SELECT rowid, title
FROM docs
WHERE docs MATCH 'title : sqlite';
```

`MATCH` 接收的是 FTS5 查询表达式。查询表达式中的词仍然要经过分词器处理，因此它与 `LIKE '%字符串%'` 的子串匹配并不等价。

实际应用中，查询值应通过 SQL 参数绑定传入。如果产品只提供普通搜索框，还需要决定如何处理引号、星号以及 `AND`、`OR` 等查询语法；参数绑定防止 SQL 注入，但不会自动把这些字符变成普通搜索文字。

### 相关性排序

FTS5 提供 `bm25()` 函数，用来比较记录与查询的相关程度：

```sql
SELECT rowid, title, bm25(docs) AS score
FROM docs
WHERE docs MATCH 'search'
ORDER BY score;
```

**在 SQLite FTS5 中，`bm25()` 返回值越小，匹配越好，因此按升序排列。** 它综合词频、词的稀有程度和文档长度等因素，也支持为不同列设置权重。

需要注意：使用 `MATCH` 只表示进行全文匹配，并不自动保证结果按相关性排列。应用需要明确指定排序方式。

## 为什么创建一张表，却出现多张表

关键在于这条语句：

```sql
CREATE VIRTUAL TABLE docs USING fts5(title, body);
```

`docs` 是一张**虚拟表**：它提供 SQL 查询接口，实际存储和检索逻辑由 FTS5 模块实现。为了把原文、倒排索引和统计信息持久化，模块会自动创建一组普通的底层表，称为**影子表（shadow tables）**。

默认配置下，除了 `docs`，还会看到：

```text
docs_config
docs_content
docs_data
docs_docsize
docs_idx
```

这些影子表共同支撑同一个全文搜索对象。它们的名称以虚拟表名为前缀，由 FTS5 自动创建和维护。

因此，本地资源库里看到的七个名字，应分成三类：

| 名称 | 类型 | 用途 |
| --- | --- | --- |
| `asset_search` | FTS5 虚拟表 | 应用执行全文查询的入口 |
| `asset_search_config` | FTS5 影子表 | 保存 FTS5 的持久化配置和格式版本等信息 |
| `asset_search_content` | FTS5 影子表 | 保存各索引列的原始文本 |
| `asset_search_data` | FTS5 影子表 | 保存主要的倒排索引数据块及部分统计信息 |
| `asset_search_docsize` | FTS5 影子表 | 保存每行各列的 token 数量，供长度统计和相关性计算使用 |
| `asset_search_idx` | FTS5 影子表 | 辅助定位索引段中的数据页，配合 `_data` 完成查找 |
| `asset_search_dirty` | 项目自行创建的普通表 | 记录哪些资源的搜索内容需要更新 |

### `_content`：原文副本

对于默认的、保存内容的 FTS5 表，插入的列值会保存在 `_content` 中。它通常包含 `id`、`c0`、`c1` 等列，其中 `id` 对应虚拟表的 `rowid`，`c0`、`c1` 依次对应索引表的各列。

因此，**原文保存在 `_content`，而不是 `_data`**。应用从 FTS5 表读取列值，以及部分高亮、摘要和维护操作，都可能需要这些原文。

### `_data` 与 `_idx`：索引存储和定位

倒排索引主要以数据块的形式保存在 `_data` 中；`_idx` 辅助定位索引段中的页。可以把它们理解为“索引数据”和“访问这些数据的辅助目录”。

把 `_idx` 单独称为完整的倒排索引并不准确，二者是协作关系。它们的空间占比也会随文本、更新历史和配置变化，没有固定百分比。

### `_docsize` 与 `_config`：统计和配置

`_docsize` 保存每行各列的 token 数量，使 FTS5 可以快速获得文档长度等信息。`_config` 保存模块的格式版本及部分持久化设置，例如用户配置的排名函数或合并策略；它并不等于完整的建表参数清单。

影子表的组合还会随配置变化。例如，外部内容模式通常不创建 `_content`，而 `columnsize=0` 会省去 `_docsize`。上面的五张影子表对应的是本文示例和资源库使用的普通配置。

## 放到本地资源库里：它具体做什么

以下以 Emulsion Desktop 的本地资源库实现为例。索引结构和更新流程位于 `local_library/store_search.go`。

### 把分散的文字汇总成一条搜索记录

资源的可搜索信息分散在不同业务表中：

| 搜索字段 | 来源 |
| --- | --- |
| `file_name`、`relative_path` | 资源文件名和相对路径 |
| `display_title`、`notes` | 用户填写的标题、备注 |
| `tags` | 资源关联的标签名称 |
| `collections` | 资源所属的集合名称 |
| `camera` | EXIF 中的相机品牌、型号和镜头型号 |

项目通过 `asset_search_source` 视图汇总这些字段，再写入 `asset_search`。这个视图只是取数逻辑，本身不额外保存一份索引。

全文索引中的 `rowid` 对应 `assets.local_id`。搜索命中后，应用可以根据这个行号关联回资源记录，继续获取业务字段和应用其他筛选条件。

整体关系是：

```text
assets / exif / tags / collections 等业务数据
                     ↓ 汇总
          asset_search_source 视图
                     ↓ 建立或更新索引
          asset_search + FTS5 影子表
                     ↓ MATCH 查询
              命中的资源行号
                     ↓ 关联 assets
                返回资源列表
```

业务数据是源头，搜索表是可以从业务数据重新生成的派生数据。FTS5 并不会自动理解这些表之间的关系，字段如何汇总、何时同步，都由项目实现。

### 为什么还需要 `asset_search_dirty`

扫描资源库时，同一张照片可能连续发生多次变化：创建资源记录、更新 EXIF、关联标签等。如果每次变化都马上重写全文索引，就会产生重复工作。

项目为此创建了一个去重队列：

```sql
CREATE TABLE asset_search_dirty (
  asset_id TEXT PRIMARY KEY
);
```

资源插入、可搜索字段修改、标签或集合关联变化等操作，会通过触发器把受影响的资源 ID 放入队列。主键保证同一资源只需要等待一次重建。

随后，`flushAssetSearch` 在搜索前或扫描收尾时处理队列：

1. 查出待更新资源对应的索引行。
2. 删除这些资源的旧索引记录。
3. 从 `asset_search_source` 重新读取汇总后的文字，批量写回索引。
4. 清空已处理的待更新队列并提交事务。

资源删除是另外一种情况：删除触发器会直接移除对应的索引行和待更新记录，避免保留已不存在的资源。

这套机制的目标是**合并重复更新，减少扫描期间的索引维护开销，并让搜索读取更新后的内容**。批量刷新本身仍然在数据库事务中执行，并不意味着索引写入被放到了事务之外。

`dirty` 在这里表示“索引需要刷新”，不是业务数据损坏。队列短暂有记录是正常现象，但不能为了让它看起来干净就直接清空，否则尚未同步的变更会被遗漏。

### 它与向量搜索如何配合

FTS5 擅长查找文本中明确出现的词，例如文件名片段、相机型号或标签；向量检索则用于寻找语义上或视觉上相近的资源。

在项目的混合检索路径中，`local_library/embedding_search.go` 分别取得关键词和向量通道的候选排名：关键词通道使用 `MATCH` 和 `bm25()`，再通过 RRF（倒数排名融合）合并两路结果。

普通关键词列表查询也使用 FTS5 进行匹配筛选，但其最终排序由列表查询决定，不能据此认定所有搜索结果都会自动按 BM25 排列。

## 分词器决定“能搜到什么”

**token 是分词器产生的检索单元，不一定等于自然语言中的一个词。** 同一段文字如何切分，直接决定什么查询能够命中。

默认分词器根据 Unicode 字符类别识别连续的字母、数字等字符，并以空格、标点等分隔。它不提供中文词典意义上的自然语言分词，因此不能仅凭“用了 FTS5”就断言中文关键词检索已经完善。

例如，一段没有空格和标点的中文可能被视为一个连续 token。用户只输入其中两个字时，并不必然命中。若希望支持中文词语搜索，需要选择或接入适合的分词方案；若希望支持任意子串，则需要另外评估相应索引策略。

FTS5 内置的 `trigram` 分词器提供了基于连续三个 Unicode 字符的子串检索能力，但也有边界：使用 `MATCH` 查询时，不足三个字符的子串不会命中。因此，它也不能直接解决所有中文短词查询。

另外，示例中的 `trans*` 是 **token 的前缀匹配**，不等于任意位置的子串搜索。项目最终的匹配行为，应以建表时的 `tokenize` 配置和实际查询处理为准。

## 内容存储模式的取舍

FTS5 不只有一种内容存储方式：

| 模式 | 原文在哪里 | 主要取舍 |
| --- | --- | --- |
| 普通模式 | FTS5 自己的 `_content` 表 | 可以直接读取索引列，但会额外保存一份原文 |
| 外部内容模式 | 指定的业务表或视图 | 减少原文重复存储，需要应用保证原文与索引一致 |
| 无内容模式 | FTS5 不保存可供读取的索引列原文 | 节省存储，但读取原文和部分维护操作受限，具体取决于配置 |

本文的资源库使用普通模式，未指定外部 `content` 来源。因此，业务表之外还会有 `_content` 中的文本副本。

从结构上可以看出，这个选择以额外存储换取了自包含的 FTS 内容存储和维护方式。外部内容模式也能使用稳定的资源 ID，并通过表或视图提供原文；它并非做不到，只是需要采用对应的同步设计。

## 检查、整理与重建

通常只需要操作 FTS5 虚拟表，由 SQLite 维护影子表。不要直接修改或删除 `_data`、`_idx` 等内部结构。

在前面的 `docs` 示例中，可以通过特殊命令进行维护：

```sql
-- 检查 FTS 索引的内部一致性；成功时不会返回结果行。
INSERT INTO docs(docs) VALUES('integrity-check');

-- 合并索引段，整理全文索引。
INSERT INTO docs(docs) VALUES('optimize');

-- 根据当前保存的原文重新建立全文索引。
INSERT INTO docs(docs) VALUES('rebuild');
```

这里要区分几件事：

- **普通模式支持 `rebuild`。** 它根据当前 FTS 表保存的原文重建索引；无内容表不能这样重建。
- **重建内部索引不等于重新导入业务数据。** 如果 `asset_search` 里的原文本身已经落后于 `assets`、标签或 EXIF，仅执行 `rebuild` 不会重新运行 `asset_search_source`，仍需要应用完成数据同步或全量回填。
- **`optimize` 不保证数据库文件立刻变小。** 它整理的是全文索引结构，数据库文件的空闲页回收属于另一个层面的维护。
- **看到影子表存在，不足以证明索引健康。** 表名只能说明结构存在，还需要结合一致性检查、实际查询和应用同步状态判断。

## 参考资料

- [SQLite FTS5 官方文档](https://www.sqlite.org/fts5.html)
- [FTS5 数据结构：索引、原文、文档长度与配置](https://www.sqlite.org/fts5.html#fts5_data_structures)
- [FTS5 分词器](https://www.sqlite.org/fts5.html#tokenizers)
- [外部内容与无内容表](https://www.sqlite.org/fts5.html#external_content_and_contentless_tables)
- [FTS5 特殊维护命令](https://www.sqlite.org/fts5.html#special_insert_commands)

项目实现参考：`local_library/store_search.go`、`local_library/store.go`、`local_library/embedding_search.go`、`local_library/scan_pipeline.go`。
