---
title: Web 阅读器迁移——讨论上下文与未决问题
summary: 把 open-writer 的 Web 阅读器落到 dsh 的可行性评估现场：已测量的规模、已核实的 dsh 架构事实、已达成的结论、待裁定的决策，以及阻塞设计的核心未决问题（阅读器的归属模型与 UI 落位）。
keywords: reader | open-writer | ow | 阅读器 | 迁移 | slots | shell.overlay | workspace | 未决问题
scope: 本 fork 内“把 ow 阅读器搬到 dsh”这项工作的上下文交接，尚未成为计划
related_files: docs/subsystems/slots.md | docs/subsystems/workspace.md | docs/subsystems/storage.md | packages/client/ui-layout/src/client/index.ts | packages/client/ui-primitives/src/markdown | packages/client/ui-trajectory
dependencies: dev_docs/AI_Coding_Context.md | dev_docs/memos/index.md | dev_docs/plans/index.md
verified_at: 2026-09-06
---

# Web 阅读器迁移——讨论上下文与未决问题

> **本文定位**：[`memos/`](index.md) 的第三类内容——**上下文交接**。记录一段设计讨论的现场：做到哪、有哪些已核实的事实、哪些假设还没验证、下一步该裁定什么。
>
> **本文不是事实的归属地。** 引用的 dsh 事实一律以 `docs/` 与源码为准；引用的 ow 事实以 `/Users/zibuyu/code/zibuyu/open-writer` 为准。
>
> **毕业判据**：下文 §6 的 D1–D4 四条决策全部冻结后，本文提炼为 [`plans/active/`](../plans/active/index.md) 下的正式计划，本备忘随即删除。在此之前它会被反复讨论、反复改写。
>
> **当前进展（2026-09-06）**：假设 A1 / A2 / A4 / A5 已查实（见 §8），D1 与 D3 已裁定，架构设计已产出 → [`plans/active/2026-09-06-reading-capability-architecture.md`](../plans/active/2026-09-06-reading-capability-architecture.md)。本备忘保留的价值仅剩**规模测算**（§2、§4）与 **ow 侧 ADR 清单**（§7）；设计相关内容一律以计划文档为准。

---

## 1. 背景与目标

个人产品方向：把 `open-writer`（下称 **ow**，`/Users/zibuyu/code/zibuyu/open-writer`）的能力重建在 dsh 上，最终作为独立产品发布。目标特性四项，本文只覆盖第 2 项：

| # | 特性 | 本文覆盖 |
| --- | --- | --- |
| 1 | Prompt 输入劫持（中文输入 → 翻译 → 英文送模型） | ✗（落点已探明：`agent/pre-step` + `MessageSourceMap` 扩展） |
| 2 | **Web 阅读器 + 在线 AI 辅助阅读** | ✓ 本文 |
| 3 | 分层记忆系统 | ✗ |
| 4 | 辅助写作 | ✗（v1 不做） |

**一句话目标**：本地 markdown / txt 文件的 Web 阅读体验，带划词 AI 助读，作为 dsh 的**基础能力**存在。

**已冻结的外围前提**（前几轮讨论的结论，非本文议题）：

- ow 的 TUI（31,343 行）不迁移。
- 交付形态是**独立仓库消费已发布的 `@deepseek-ai/dsh-*` npm 包**，不是 fork 内开发；本 checkout 保留为参考与实验场。因此**不受本仓 `test:coverage` 每文件 100% 门禁约束**。
- dsh 包当前 `0.1.3-alpha.1`，上游明确会有破坏性变更 → 消费时锁精确版本，手动升级。
- 白牌化在技术上可行（MIT + 品牌插槽设计），但排在功能之后。唯一建议提前做的是禁用 `session-log-deepseek` 与 `plugin-package-inventory-deepseek` 两行——那是隐私边界，不是品牌问题。

---

## 2. ow 侧实测规模

`verified_at: 2026-09-06`，非测试代码。

**后端 surfaces —— 4,978 行**

| 文件 | 行 |
| --- | --- |
| `src/web/surfaces/local-reader/surface.ts` | 871 |
| `src/web/surfaces/assist/lesson-surface.ts` | 546 |
| `src/web/surfaces/assist/chat-surface.ts` | 493 |
| `src/web/surfaces/assist/actions.ts` | 461 |
| `src/web/surfaces/assist/surface.ts` | 330 |
| `src/web/surfaces/assist/chunking.ts` | 291 |
| `src/web/surfaces/assist/chat-context.ts` | 275 |
| `local-reader/` 其余（`marks` `file-index` `path-safety` `picker` `history` `json-store`） | 655 |
| `assist/` 其余（`anchoring` `runner` `chat-summary-cache` `types`） | 484 |
| `annotations/` + `reading-position/` | 330 |

**前端 —— 约 15,000–17,000 行**

| 文件 | 行 |
| --- | --- |
| `LocalReader.tsx` | **2,794** |
| `BookReader.tsx` | 1,276 |
| `AssistWindow.tsx` | 926 |
| `SelectionAssist.tsx` | 846 |
| `LessonPane.tsx` | 833 |
| `reader-chrome.tsx` | 832 |
| `DocChatPanel.tsx` | 422 |
| `selection-helpers` `local-reader-errors` `keymap` `assist-windows` `assist-dock` | 1,739 |
| `AnnotationLayer` `ChapterPane` `reader-position` `assist-stream` `reader-icons` | 1,287 |
| `markdown/`（渲染管线） | 2,202 |
| `styles/` 中阅读器相关 | ~4,000 |

**合计约 20,000 行非测试代码。** 这是 ow 里最不通用的一块——没有任何底座能替你写掉它。

**产品决策资产**：`dev_docs/spec/web-console/decisions/` 下 **89 篇 ADR**。这是本次迁移**真正的资产**——代码绑在 opencode 的传输与装配上，是负债；ADR 里的产品判断（划词菜单何时弹、dock 如何锚定、选区生命周期在哪结束）是踩坑踩出来的，不可再生。

---

## 3. dsh 侧已核实的架构事实

以下每条都经源码或 `docs/` 核实，**是本次设计的硬约束**。

### 3.1 dsh 白送的（不用写）

| ow 的东西 | dsh 已有 | 备注 |
| --- | --- | --- |
| `markdown/` 2,202 行 + react-markdown/remark 全家桶 | [`packages/client/ui-primitives/src/markdown/`](../../packages/client/ui-primitives/src/markdown) | micromark/mdast + shiki 高亮 + katex 数学 + **CJK 友好加粗** + 增量渲染 + 视口懒高亮。**能力强于 ow 的管线** |
| `src/web/server/` router / sse / static / multipart | `ctx.webServer`（[`packages/host/webserver`](../../packages/host/webserver)） | 命名路由注册表 + 索引变换 tap + 静态兜底 |
| `local-reader/picker.ts` 原生选目录 | `ui-directory-picker-native` + `ui-directory-picker-browse` | ow ADR-027 / 049 白送 |
| `local-reader/json-store.ts` `history.ts` | `ctx.storageDomain`（[storage.md](../../docs/subsystems/storage.md)，json / sqlite 双后端） | |
| `assist/runner.ts` `assist-stream.ts` | LLM 流式 + session 事件流 | |
| React 外壳 / 主题 / i18n / 三栏布局 / 可拖拽面板 | `ui-layout` `ui-theme` `client-locale` | |
| 文件读写 + 路径安全 | `ctx.fs` + policy 层 | ow `path-safety.ts` 159 行大部分可省 |

**dsh 的 markdown 管线缺**：mermaid、`remark-directive`、heading slug / 自动锚点（TOC 依赖它）、图片 lightbox、宽表格弹窗。这几项需自行补，且**不能改上游包**，只能包一层。

### 3.2 插槽树：根层只有四个，没有“路由”概念

[`packages/client/ui-layout/src/client/index.ts:124-131`](../../packages/client/ui-layout/src/client/index.ts)：

```text
root
├─ sidebar        { kind: 'single', scope: 'root' }
├─ conversation   { kind: 'single', scope: 'session-maybe' }
├─ details        { kind: 'single', scope: 'session' }
└─ shell.overlay  { kind: 'list',   scope: 'root' }
```

完整层级见 [`docs/subsystems/slots.md`](../../docs/subsystems/slots.md#current-hierarchy)。三条推论：

1. **`conversation.view`（会话 Tab 环，`ui-trajectory` 所在处）是 `scope: session`。** 阅读器若放这里，就**天然隶属于某个会话**——与 §5 的设想直接冲突。
2. **`sidebar` 与 `conversation` 是 `single`**，即替换点；占用它们会顶掉现有占位者。
3. **`shell.overlay` 是 root 作用域的 `list`**，且其 JSDoc（同文件 `:83-85`）明写：

   > *This is the additive seat for a frame-wide surface of your own: a fresh `id` is added beside the shipped entries instead of replacing them.*

   **这是 dsh 官方指定的“自有全框架级界面”加法席位。** 渲染点在 [`AppFrame.tsx:210-211`](../../packages/client/ui-layout/src/client/AppFrame.tsx)，包在 `css.overlayLayer` 里。**注意**：该层是 click-through 的，占位者需自行 opt-in pointer events；它渲染在三栏**之上**。

### 3.3 客户端没有任何路由

全量 grep `pushState` / `replaceState` / `location.hash` / `location.search` / `popstate`，`packages/client` 与 `apps/web` 的产品代码中**零命中**（仅两处测试 fixture）。

**推论**：dsh Web 客户端连“当前是哪个会话”都不在 URL 里。ow 的 ADR-038（阅读器 URL 状态）**在 dsh 没有地基**——要么自建，要么放弃可链接/可书签的阅读位置。

### 3.4 workspace 的形状 = “实体 → 有序会话账目”

[`docs/subsystems/workspace.md`](../../docs/subsystems/workspace.md)：`ctx.workspaceRegistry` 把一个目录记录为 `WorkspaceId`（branded uuid，**不是路径**）+ 规范化路径 + 标题 + **该 workspace 名下的有序会话账目**。记录存于 `storageDomain`；`~/.dsh/storages/workspace.json` 里可见 `sessionIds: []` 字段。

**关键约束**：**成员资格按 `SessionHeader.cwd` 校验**——即“会话属于哪个 workspace”这一事实的权威锚点是会话头里的 **cwd（目录）**，粒度是目录，不是文件。

这一条直接决定了 §5 的可行方案集合。

### 3.5 其他已探明的落点

| 需求 | 落点 | 出处 |
| --- | --- | --- |
| 拦截/改写用户输入 | `agent/pre-step` 瀑布 | `packages/core/agent/src/runtime-types.ts:276` |
| 自定义消息来源（携带选区/原文/文件路径） | `MessageSourceMap` 声明合并 | `packages/llm/llm/src/message.ts:95-149` |
| 跨会话引用 / 文件补全候选 | `session-reference` + `file-reference` | [session-reference.md](../../docs/subsystems/session-reference.md) |
| 一个完整 Tab 的参考实现（纯消费者） | `ui-trajectory`（11,350 源码 + 4,707 测试） | [`packages/client/ui-trajectory`](../../packages/client/ui-trajectory) |

---

## 4. 落到 dsh 的工作量估算

| 模块 | 非测试估算 | 说明 |
| --- | --- | --- |
| Host 插件：文件索引 / 目录列举 / 读写 / 最近目录 / 书签 / 位置 | 1,200–1,800 | 走 `ctx.fs` + `ctx.storageDomain` + Typert `@Remote`，比 ow 省 |
| 阅读器主体（渲染、TOC、章节、设置、快捷键、滚动位置） | 4,000–6,000 | CSS Module + locale 字典会加量 |
| 划词 → 助读（选区模型、菜单、窗口/dock、锚定） | 3,000–4,000 | ow 最精华也最重的部分 |
| 助读后端 | 400–600 | 若走 D2「session turn」路线，比 ow 的 1,300 行省一大截 |
| 标注层 | 600 | |
| markdown 补丁（mermaid / TOC 锚点 / lightbox） | 800 | |
| locale 字典（中英） | 500 | |
| **非测试合计** | **11,000–16,000** | |
| 测试（务实标准 ≈0.6×） | ~8,000 | 不受本仓 100% 覆盖门禁约束 |
| **总计** | **≈ 20,000–24,000 行** | |

**参照物**：`ui-trajectory` 一个 Tab = 16,057 行（源码 + 测试），而它只是表格 + 时间轴。阅读器复杂度高于它，估算量级 ×1.5–2 合理。

**节省不在阅读器本身，在阅读器周围**——省掉的是 ow 那 55k 行通用底座。

---

## 5. 核心未决问题：阅读器的归属模型

> **这是当前最重要、也最未定的一条，其余决策都依赖它。**

### 5.1 设想（用户表述，2026-09-06）

> Web 阅读应该作为一个**基础功能**，供本地 markdown 和 txt 阅读使用。因此阅读**不应该隶属于 session**；相反，**可能会有多个 session 是关于同一个文档的**。

翻译成架构语言：**文档是一等实体，会话挂在文档下**，关系是 `document 1 — n session`。这**推翻**了前一轮“阅读器做成 `conversation.view` 的一个 Tab”的建议——因为该插槽是 `scope: session`（§3.2）。

### 5.2 两个正交的子问题

必须分开裁定，混在一起讨论会绕：

- **P1｜实体锚点**：“某文档的所有会话”这一关系存在哪、由谁保证一致。
- **P2｜UI 落位**：阅读界面在 `AppFrame` 的哪个位置渲染。

### 5.3 P1 实体锚点——三个候选

| | 甲：无实体 | 乙：书库目录即 workspace | 丙：新增 `documentRegistry` |
| --- | --- | --- | --- |
| 文档如何寻址 | 规范化绝对路径 | 路径（目录由 workspace 管） | branded `DocumentId`（uuid over 规范化路径） |
| “某文档的会话”从哪来 | 反查会话元数据 | workspace 的会话账目 + 会话内自述文档 | registry 自持有序会话账目 |
| 复用 | `storageDomain` KV | **完全复用 `ctx.workspaceRegistry` 形状与 UI** | 照抄 workspace 实现 |
| 一致性锚点 | 无 | **`SessionHeader.cwd`**（现成、权威） | **无对应物** ← 核心难点 |
| 工作量 | 最小 | 小 | 大 |
| 贴合设想程度 | 低 | 中 | 高 |

**乙的要害**：workspace 成员资格按 `SessionHeader.cwd` 校验，粒度是**目录**（§3.4）。所以“一个文档 = 一个 workspace”不成立；只能“**一个书库目录 = 一个 workspace**，文档是其中的文件”。那么“这个会话是关于哪个文档的”仍需额外记录。

**丙的要害**：`documentRegistry` 可以照抄 workspace 的 `documentId → sessionIds[]` 账目结构，但 **dsh 里不存在 `SessionHeader.document` 这样的权威校验锚点**。workspace 之所以敢维护一份账目，是因为它能拿 `SessionHeader.cwd` 做交叉校验；document 没有这个后盾，账目一旦与实际会话漂移就没有真相来源。

> **⚠️ 未验证假设 A1**：一个 session 能否携带“我关于哪个文档”这一事实，并被反向查询？三条候选路径，均**未核实**：
> 1. 作为**首条 session 事件**写入日志（符合“模型可见 ⟺ 已记录”，但反查需扫日志或建索引）；
> 2. `SessionHeader` 是否有可扩展的元数据字段（**待读** [`docs/subsystems/persistence.md`](../../docs/subsystems/persistence.md) 的 `SessionHeader` 定义）；
> 3. `ctx.sessionQuery` 是否支持按事件内容检索（**待读** [`docs/subsystems/session-query.md`](../../docs/subsystems/session-query.md)）。
>
> **A1 的答案直接决定甲/乙/丙哪个可行。这是下一步最该做的调研。**

### 5.4 P2 UI 落位——三个候选

前提：既然阅读不隶属会话，落位必须是 **root 作用域**，`conversation.view` 出局。

| | I：`shell.overlay` | II：fork `ui-layout` 增顶级插槽 | III：替换 `sidebar` / `conversation` |
| --- | --- | --- | --- |
| 上游改动 | **零** | 需 patch/fork `ui-layout` + `ui-renderer` | 零，但顶掉现有占位者 |
| 官方背书 | **JSDoc 明写是“自有全框架级界面的加法席位”** | 无 | 明确标注为“替换点”，不建议 |
| 作用域 | `root` ✓ | 自定义 ✓ | `sidebar` root ✓ / `conversation` session-maybe ✗ |
| 代价 | 浮层语义：click-through 需 opt-in；渲染在三栏之上；与侧栏共存需自理 | 持续承担上游合并成本 | 破坏既有产品形态 |

**倾向 I**，理由是 §3.2 引用的那句 JSDoc——它不是变通，是 dsh 为这种需求预留的正门。但**浮层语义是否能承载一个常驻主界面**（而非弹窗）**尚未验证**。

> **⚠️ 未验证假设 A2**：`shell.overlay` 的占位者能否表现为一个**常驻、占满、可与侧栏并列**的主界面，而不是浮在内容之上的层？需读 `AppFrame.module.css` 的 `overlayLayer` 几何，并做一次最小实验。

---

## 6. 决策清单

状态：**已定** / **建议**（我给了倾向，待你确认） / **未定**（缺前置调研）

| # | 决策 | 状态 | 内容 |
| --- | --- | --- | --- |
| **D1** | 实体锚点（P1） | **未定** | 甲 / 乙 / 丙，阻塞于假设 A1 |
| **D2** | 助读走哪条路 | **建议** | **不做 ow 式旁路 `assist/runner`，而是让划词助读成为一次真实 session turn**，带自定义 `MessageSourceMap`（携带选区/原文/文件路径）。后果：助读全程进 session log → 特性 3「记忆系统」可直接读日志拿到全部阅读行为；助读还能顺手用上全套工具 |
| **D3** | UI 落位（P2） | **建议** | I（`shell.overlay`），阻塞于假设 A2 |
| **D4** | v1 只读还是可编辑 | **建议** | **只读**。ow 的可编辑面带来 CodeMirror + 版本历史 + 脏页确认 + 分段编辑一整条线（ADR-009/029/044），砍掉省 3,000+ 行；需要改文件时让用户对 Agent 说，`edit`/`write` 工具与权限层本就在 |
| D5 | 搬哪个阅读器 | **建议** | **只搬 `LocalReader`**（任意本地文件）。`BookReader` 连带 `content-store` 1,560 + `library` 4,199 + `ingestion` 937，v1 全砍 |
| D6 | 标注存哪 | **建议** | 阅读位置 / 书签 / 最近目录 → `storageDomain`；**标注 → session 事件**。依据：dsh 硬规矩「模型可见 ⟺ 已记录」，标注迟早要进助读 prompt |
| D7 | 版本策略 | **已定** | 锁精确版本，手动升级，不用 `^` |
| D8 | i18n | **建议** | 照 dsh 的 locale-owned 规矩走类型化字典。自有仓库无此门禁，但产品本身是中英双语 |
| D9 | v1 ADR 白名单 | **建议** | 见 §7 |

---

## 7. ADR 白名单草案（D9）

从 ow 的 89 篇中筛。**这是体量的主要来源，建议从严。**

**进 v1**：005 阅读面 · 008 布局与 TOC popover · 026 本地文件面 · 035 最近目录 · 046 侧栏搜索 · 051 位置与书签 · 056 键位 · 053 / 063 / 064 / 065 划词四条 · 015 助读出站 · 016 多窗口助读 · 074 / 075 / 080 助读 dock · 047 / 048 专注模式 · 018 本地标注 · 085 错误映射 · 081 选区生命周期

**不进 v1**：011 / 012 双语分屏 · 031 版本历史 · 044 分段编辑 · 055 / 061 局域网访问 · 071 / 072 / 073 lesson 笔记本 · 024 / 078 book 导入 · 050 / 084 content-store · 017 助读回写 · 040–043 doc-chat（被 D2 取代）

**待定**：038 URL 状态——**dsh 无路由地基**（§3.3），要么自建要么放弃

---

## 8. 遗留问题与待验证假设

| # | 问题 | 状态 | 结论 |
| --- | --- | --- | --- |
| **A1** | session 能否携带并反查“关于哪个文档” | **已查实** | **元数据层不能**——`SessionHeader` 与 `CreateSessionOptions.meta` 均为封闭接口，`SessionResultFilter` 是封闭联合（仅 `id`/`cwd`/`created-at`/`parent`/`availability`）。**但日志层可以**——`ctx.sessionProjections` 提供纯同步 fold，`filterEvents` 支持按事件类型过滤。→ D1 采用「投影 + 可重建索引」，见计划文档 §4 |
| **A2** | `shell.overlay` 能否承载常驻主界面而非浮层 | **已查实** | **可以**。`.overlayLayer { position:absolute; inset:0; z-index:20; pointer-events:none }` + `> * { pointer-events:auto }`。占位者可铺满 `.frame` 并接管指针事件。代价：覆盖三栏含侧栏，是接管而非并列 → D3 采纳，见计划文档 §8 |
| A3 | 无路由 → 阅读位置能否被链接/书签 | 已裁定 | v1 放弃 ADR-038。位置本就 durable 在 `reading-store`，「重开回到上次位置」不依赖 URL |
| **A4** | `ctx.fs` 是否允许读工作区之外的任意本地路径 | **已查实** | **可以**。`fs-observation-policy` 只裁决 `write-intent`/`edit-intent` 两个瀑布，读不过策略门。`listDir`/`readText`/`streamText`/`readBytes` 全部现成 |
| **A5** | `MarkdownText` 能否被外部扩展（本地图片 / TOC 锚点 / mermaid） | **已查实** | **不能**。渲染器策略：图片必须绝对 HTTP(S)、fragment 锚点不过白名单、DOM 被 fixture 逐字节钉死；且 `parse.ts`/`incremental.ts` 未导出。→ 阅读器需自有渲染臂（`micromark`/`mdast-util-*` 是普通 npm 依赖），`CodeBlock` 可复用。见计划文档 §2.5、§10 |
| A6 | 独立仓库如何消费 client 侧插件构建链 | **已查实**（作为计划文档的 V3） | `dsh.client.platform: 'web'` + 导出 `./client`，Host 自动扫描供给；`lib/client.js` 缺失响亮失败；共享基线含 `ui-primitives`。详见计划文档 §6 |
| A7 | 「财务状况」会话 Tab 来源不明 | 未查 | 无（好奇）；问用户或查 `--dump-config` |

**新增未决**（随设计产生，见计划文档 §12）：V1 `reading/attach` 是否 `ignorable`、V2 阅读会话的生命周期、V4 图片资源路由鉴权、V5 Typert 是否唯一正路、V6 大文档分窗。

---

## 9. 下一步

~~1. 查 A1 → 裁定 D1。~~ **已完成**，D1 采用「会话投影 + 可重建索引」。
~~2. 查 A2 → 裁定 D3。~~ **已完成**，D3 采用 `shell.overlay`。

3. 确认 **D2 / D4 / D5 / D6 / D9** 五条建议。
4. 全部冻结后，本文提炼为 `plans/active/YYYY-MM-DD-reader-migration.md`，含：
   - `00-decisions`：D1–D9 的最终答案，冻结，后续所有文档引用它
   - `01-architecture`：包划分、插槽契约、Typert 方法签名、`MessageSourceMap` 扩展、`storageDomain` schema
   - `02-adr-inventory`：89 篇 ADR 逐条标注进/不进/改写
   - `M1`–`M8` 里程碑，每篇四段：**验收条件**（浏览器里做 X 看到 Y）、**不做什么**（比“做什么”更重要）、**契约**（写死的类型签名）、**参考实现**（dsh 内对应范本路径）

> **关于“交给 Claude Code 全程完成”**：一份计划 + 一次性执行，在 2 万行规模上会失败——前 3,000 行顺利，之后架构漂移，回头重来的成本高于重写。可行的模式是**每个里程碑一个 PR，人工在浏览器里验收后再合**。8 个里程碑，每个约 5–10 个会话。

---

## 延伸阅读

- [`AI_Coding_Context.md`](../AI_Coding_Context.md) — dev_docs 总入口
- [`apps_cli_and_web.md`](../apps_cli_and_web.md) — UI 改动定位表
- [`plugin_development_guide.md`](../plugin_development_guide.md) — 扩展点选择决策树
- [`docs/subsystems/slots.md`](../../docs/subsystems/slots.md) — 插槽层级与扩展规则（**权威**）
- [`docs/subsystems/workspace.md`](../../docs/subsystems/workspace.md) — workspace 实体形状（**权威**）
