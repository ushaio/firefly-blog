---
title: Tiptap-markdown
published: 2026-09-05
description: Tiptap-markdown
image: "api"
tags: [MO Gallery, 归档]
category: MO Gallery
draft: false
---
<img src="https://messages-prod.27c852f3500f38c1e7786e2c9ff9e48f.r2.cloudflarestorage.com/01a06a13-d53d-70f1-851d-759a34765239/1788593529987-image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=b33de61d4f22a31b59b25364ab5037c5%2F20260905%2Fauto%2Fs3%2Faws4_request&X-Amz-Date=20260905T073211Z&X-Amz-Expires=3600&X-Amz-Signature=298405e9caecc9bfef5ed7f1988c286565da454384a1cc2dc9271607fb58556f&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject" style="zoom: 50%;" />

项目是基于tiptap开源开发的，想将markdown语法和键盘交互对齐typora的操作，但是总是出现一些逻辑上的bug且很难彻底修复，或者几个问题循环出现，而且我这边是想做一个能实现微信公众号等其他场景发布的编辑器，因此不是那种纯markdown语法编辑器，为此对内容有许多样式风格化的调整，有什么建议吗，或者其他替代方案？



看了截图，你这个产品定位其实很清楚：**富文本内容编辑器 + Markdown 快捷输入 + 多渠道（公众号）样式化导出**。先说结论：问题大概率不在 Tiptap 这个选型上，而在于“把两种不同的文档模型硬拧在一起”，换编辑器不一定能解决，换思路更重要。

## 一、为什么 bug 会“循环出现”

Typora 和 Tiptap 的底层模型完全不同：

|                     | Typora                | Tiptap / ProseMirror                         |
| ------------------- | --------------------- | -------------------------------------------- |
| 真相源              | Markdown 纯文本       | 有 Schema 的文档树（JSON）                   |
| 列表                | 文本行 + 缩进         | `bulletList > listItem > paragraph` 严格嵌套 |
| 加粗                | `**` 字符本身在文档里 | Mark 是附加在文本上的属性，没有字符          |
| 字体/字号/对齐/颜色 | 不存在                | 是节点/Mark 的 attrs                         |

你要的“对齐 Typora 的键盘交互”，本质是想让一棵结构化的树表现得像一段纯文本。典型的坑：

- **Enter / Backspace / Tab 在列表里的语义**：Typora 里“空列表项回车”“行首退格”是对文本行的操作；PM 里要处理 `splitListItem`、`liftListItem`、`joinBackward` 以及“空 listItem 里还有嵌套列表”（截图里那个空的 `2.` 就是这种状态，Typora 里几乎不可能产生）。
- **多个扩展抢同一个按键**：StarterKit 的 ListItem、你自定义的 keymap、Placeholder、HardBreak、undoInputRule……每个都注册了 Enter/Backspace，谁先返回 `true` 谁赢。每修一个场景就改一处 keymap，结果另一个场景又坏——这就是“循环出现”的真正原因。
- **样式 attrs 和 Markdown 往返丢失**：如果你还在用 Markdown 做存储或中转（如 `tiptap-markdown`），字体、字号、居中这些一进 Markdown 就没了，然后你不得不打各种补丁。

## 二、建议：留在 Tiptap，但改架构

### 1. 明确真相源：文档 JSON，不是 Markdown

- 存储和协作只用 Tiptap JSON（或 HTML）。
- Markdown 只做两件事：**输入快捷方式**（InputRule）和 **导入/导出**（有损，用户可接受）。
- 不要试图让文档在 Markdown 和富文本之间“无损往返”，这是不可能的，一旦放弃这个目标，一半的补丁就没必要了。

### 2. 把键盘行为写成一张“决策表”，而不是散落的 keymap

建一个单独的 `KeyboardBehavior` 扩展，设高 `priority`，所有 Enter / Backspace / Tab / Shift-Tab 都先进这里，根据上下文判断：

text



```
上下文 = { 父节点类型, 深度, 是否为空, 光标在块首/块尾, 上一个兄弟节点类型 }
```

例如 Enter 的表：

| 上下文                   | 行为                         |
| ------------------------ | ---------------------------- |
| 非空 listItem            | splitListItem                |
| 空 listItem，depth > 1   | liftListItem（升一级）       |
| 空 listItem，depth == 1  | 退出列表变段落               |
| 空 listItem 但含嵌套列表 | 先把嵌套列表提上来，再删空项 |
| blockquote 内空段落      | 退出引用                     |
| codeBlock                | 插入换行（三次空行退出）     |
| 标题末尾                 | 新建段落而不是标题           |

每一行都对应一条测试用例。命中就返回 true，不命中就 `return false` 交给默认行为。这样修 bug 就是改表格 + 加用例，而不是猜哪个扩展抢了按键。

### 3. 直接利用官方已经踩过坑的东西

- `@tiptap/extension-list-keymap`（Tiptap 3 里在 `@tiptap/extension-list` 包内）：专门修列表 Backspace/Delete 的诡异行为，很多人自己重写反而更糟。
- Tiptap 默认 keymap 里的 `undoInputRule`：实现了 Typora 那种“自动转换后按 Backspace 撤回转换”。
- Tiptap 3 官方出了 `@tiptap/markdown`（比社区的 `tiptap-markdown` 对 JSON 模型更友好），可以看看是否满足你的导入导出需求。

### 4. 给每个键盘场景写自动化测试

用 vitest + `prosemirror-test-builder`（或直接构造 JSON 文档，设置 selection，触发 command，断言 JSON）。列表相关的回归 bug 没有测试基本不可能收敛。装个 `prosemirror-dev-toolkit` 看每次 transaction 也很有帮助。

### 5. “内容”和“主题”分层，这是公众号场景的核心

微信公众号粘贴的限制：**只认内联 style，没有外部 CSS，没有 JS，没有 iframe，图片必须传微信素材库**。截图里那个 Spotify 卡片进了公众号一定会没掉。

推荐的做法（doocs/md、mdnice 都是这么做的，只是它们从 Markdown 出发，你从 JSON 出发）：

text



```
文档 JSON（内容 + 少量块级覆盖样式 attrs）
        ↓
主题（默认字体、字号、标题样式、引用样式、配色）
        ↓
渲染器：generateHTML(json, extensions) → 按主题注入 CSS → juice 内联化
        ↓
公众号 HTML / 知乎 HTML / 网页 HTML / Markdown（有损）
```

- 编辑器里工具栏调的字体、字号、对齐，尽量做成**主题级别**或**块级 attrs**，而不是随手往文本上塞 `<span style>`。否则导出到不同渠道时样式不可控，用户切换主题也没法生效。
- 像 Spotify 这类富媒体块，在渲染阶段按渠道降级：公众号 → 封面图 + 文字链接；网页 → 保留 embed。

## 三、如果真的想换，值得看的替代方案

| 方案                         | 适合吗 | 说明                                                         |
| ---------------------------- | ------ | ------------------------------------------------------------ |
| **BlockNote**（基于 Tiptap） | ⭐⭐⭐⭐   | Notion 式块模型，列表/键盘/拖拽/斜杠菜单开箱即用，支持自定义块和 Markdown 快捷输入。你现在的产品形态（右侧 AI、块级拖拽）和它很接近。缺点：React 绑定较深，深度魔改 UI 成本高。 |
| **Lexical**（Meta）          | ⭐⭐⭐    | 列表和 Markdown 快捷键做得不错，性能好。但生态比 Tiptap 小，节点系统要重新学，迁移成本高，问题不见得更少。 |
| **Milkdown**                 | ⭐⭐     | 底层还是 ProseMirror，但 Markdown 是真相源，最像 Typora。可正因为如此，你的样式化需求会和它天然冲突（要自造语法）。 |
| **Vditor / Cherry Markdown** | ⭐⭐     | 国产，Vditor 的 wysiwyg 模式明确模仿 Typora，Markdown 体验最接近。但它们是 Markdown 编辑器，样式能力弱，和你“不是纯 Markdown 编辑器”的定位矛盾。 |
| **CKEditor 5**               | ⭐⭐     | 列表引擎和 autoformat 非常成熟稳定，商业级质量，但闭源许可/收费，定制 UI 风格麻烦。 |
| Slate / Editor.js            | ⭐      | Slate 一切自己写，列表是出了名的痛；Editor.js 块模型太粗，不适合公众号排版。 |

**我的判断**：如果不是被逼到重写的地步，留在 Tiptap；如果确实想重来且技术栈是 React，BlockNote 是唯一“换了之后大概率能少踩坑”的选择，因为它把你正在手写的那部分（块模型 + 列表键盘行为）替你写完了。

## 四、一个心态上的建议

不要追求“100% Typora”。Typora 用户真正依赖的其实只有几条：`# `/`- `/`1. `/`> `/`````/`---` 的自动转换、`**x**` 的即时加粗、空列表项回车退出、Backspace 撤销自动转换、Tab 缩进。把这十几个高频行为做稳定、写好测试，剩下的长尾差异用户是感知不到的；反而是你现在这种“为了追平边角行为不断打补丁”的方式，最容易把高频路径也搞坏。