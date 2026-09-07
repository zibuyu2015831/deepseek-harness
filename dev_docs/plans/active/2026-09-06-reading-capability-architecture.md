---
title: 阅读能力架构设计
summary: 把「Web 阅读」作为 dsh 的一等能力落地的架构设计：文档访问接缝 ctx.documents 的三角角色、document ↔ session 双向关系的两套机制（会话投影 + 可重建索引）、锚点与阅读状态的归属、助读卡片的追问与停放模型、UI 在 shell.overlay 的落位、六个包的划分与里程碑。
keywords: reading | documents | 阅读器 | capability-seam | session-projection | shell.overlay | anchor | message-source | 助读卡片 | dock | 停放 | 追问 | 右栏 | 架构设计
scope: 本 fork 派生产品中「阅读」能力的架构设计与实施计划
related_files: docs/subsystems/session-projection.md | docs/subsystems/session-query.md | docs/subsystems/persistence.md | docs/subsystems/storage.md | docs/subsystems/slots.md | docs/subsystems/filesystem.md | packages/core/session/src/known-event-types.ts | packages/core/agent/src/inbox.ts | packages/session/session-persistence/src/storage-contract.ts | packages/session/session-title-llm/src/index.ts | packages/llm/llm/src/types.ts | packages/client/ui-layout/src/client/index.ts | packages/client/ui-primitives/src/markdown/render.tsx | packages/client/web/src/platform.ts | packages/api/session-controller/src/commands.ts
dependencies: dev_docs/memos/2026-09-06-reader-migration-context.md | dev_docs/AI_Coding_Context.md | dev_docs/rules/combined/AI_RULES.md
verified_at: 2026-09-07
---

# 阅读能力架构设计

> **本文定位**：[`plans/`](../index.md) 下的设计与实施计划。上下文、规模测算与决策历史在 [`memos/2026-09-06-reader-migration-context.md`](../../memos/2026-09-06-reader-migration-context.md)。
>
> **与 `docs/` 冲突时一律以 `docs/` 为准。** 本文引用的每条 dsh 事实都标注了出处，未经核实的推断显式标注为**推断**。

## 当前状态（2026-09-07）

**已定，可据以实施**：

| | 结论 | 出处 |
| --- | --- | --- |
| 能力拆分 | 阅读 = 文档访问接缝 + 阅读状态数据 + 助读消费者，三者分离 | §3 |
| `session → document` | 搭载 `user/message.source` 的 `reading-selection`，经会话投影折叠；**不新增事件类型** | §2.6、§4.2 |
| `document → sessions` | `reading-store` 里的可重建索引，发布 `./invariant` | §4.2 |
| UI 落位 | `shell.overlay` 全框架占位者（阅读模式接管） | §2.3、§8 |
| markdown | 自有 mdast→React 渲染臂；`ui-primitives` 的 `CodeBlock` 复用 | §2.5、§6 |
| 图片交付 | `@Remote` 走已鉴权的 `/api`，**不建 HTTP 路由**；必须做引用校验 | §7.4 |
| 客户端构建 | `dsh.client.platform: 'web'` + `./client`，Host 自动扫描供给 | §6 |
| 包划分 | v1 六个包，`tool-documents` 等三项延后 | §6 |
| v1 范围 | 只读、只支持 md/txt、只搬 `LocalReader`、放弃 URL 路由 | §11 |
| 助读卡片 | **一张卡就是一条会话**（`ctx.agents.create`，卡片专属最小组合）：追问 = `followup()`，卡片之间独立并发；内容在会话日志，停放状态在 `reading-store` | §7.2、§9.2 |
| 右栏 | 阅读界面内自建右栏：本文档的会话列表 + 完整转录。「转到完整会话」是换渲染，不是升级 | §8 |

**未定，阻塞实施**：

| | 问题 | 阻塞 |
| --- | --- | --- |
| **V13** | 是否摘掉 `ui-workspace` 以避免全局会话列表被卡片会话淹没 | **M7**。摘掉可行但会连带停掉侧栏外壳与两个目录选择器（硬注入 `uiWorkspace`），§7.1 的复用随之作废 ⇒ 须自建目录选择器。三条缓解见 §9.3 |
| V12 | 卡片 agent 的首字延迟与跨卡前缀缓存命中 | **不阻塞**，但影响卡片是否需要「开场走裸补全」的优化。实测见 §9.3 一 |

其余未决（V6 大文档分窗、V8 节点定义优先级、V10 卡片重建保真度）见 §13，均不阻塞 M1。

---

## 1. 目标

把「阅读本地文档」做成 dsh 意义上的**一等能力**——即 `ctx` 上的一个能力接缝，而不是一个 Web 界面功能。

判据有三条，缺一不可：

1. **不隶属会话**。阅读状态的生命周期独立于任何 session。
2. **一个文档对应多个会话**。关系可查询、可重建、有真相来源。
3. **模型也能用**。同一套文档词汇（大纲、锚点、片段）既服务阅读界面，也能服务模型工具——否则它只是个 UI 特性。

---

## 2. 设计约束（实测）

以下七条改变了设计，每条都经源码核实。§2.1–2.5 实测于 2026-09-06，§2.6–2.7 实测于 2026-09-07。

### 2.1 会话元数据是封闭的，唯一的分组轴是 `cwd`

`SessionHeader`（[persistence.md](../../../docs/subsystems/persistence.md#sessionheader--metadata-beside-the-log)）字段封闭：`version` `id` `createdAt` `cwd?` `parentSession?` `isSeeded` `origin?` `delegationDepth?` `agentPreset?`。`CreateSessionOptions.meta` 同样封闭，**没有任何可扩展的元数据槽**。

`SessionResultFilter`（[session-query.md](../../../docs/subsystems/session-query.md#provider-independent-filters-and-documents)）是封闭联合：

```ts
type SessionResultFilter =
  | { kind: 'id'; values: readonly SessionId[] }
  | { kind: 'cwd'; values: readonly (string | null)[] }
  | ({ kind: 'created-at' } & SessionResultRange)
  | { kind: 'parent'; values: readonly (SessionId | null)[] }
  | { kind: 'availability'; values: readonly SessionAvailability[] }
```

**推论**：dsh 只提供一个被索引、被校验的会话分组轴——`cwd`。这就是 `workspace` 用它的原因（[workspace.md](../../../docs/subsystems/workspace.md)：成员资格按 `SessionHeader.cwd` 校验）。**「这个会话关于哪个文档」在会话元数据层面无处安放。**

### 2.2 但会话**日志**是可扩展的，且有现成的折叠机制

`ctx.sessionProjections`（[session-projection.md](../../../docs/subsystems/session-projection.md)）：

> 域插件贡献**一个纯同步 fold**（`init(header, inheritedEventCount)` + `apply(state, event)`），框架订阅 `session/event` 并把每条已提交事件喂给每个单元。**框架驱动，域计算。** `wire` 块让该键对客户端可见；`state` 必须是纯 JSON（持久化缓存前提）。

**这是 `session → document` 的正解**：文档归属不是元数据，是**日志派生状态**——和会话标题（`session-title`）同一机制。

而 `filterEvents(sessionId, [{ kind: 'type', values: [...] }])` 支持按事件类型过滤（`SessionEventType` 随 `SessionEventMap` 声明合并而扩展），所以 `document → sessions` 可以**扫描重建**。

### 2.3 `shell.overlay` 是官方指定的自有全框架界面席位

[`packages/client/ui-layout/src/client/index.ts:124-131`](../../../packages/client/ui-layout/src/client/index.ts)：

```text
root
├─ sidebar        { kind: 'single', scope: 'root' }
├─ conversation   { kind: 'single', scope: 'session-maybe' }
├─ details        { kind: 'single', scope: 'session' }
└─ shell.overlay  { kind: 'list',   scope: 'root' }
```

`shell.overlay` 的 JSDoc（同文件 `:83-85`）原话：*"This is the additive seat for a frame-wide surface of your own: a fresh `id` is added beside the shipped entries instead of replacing them."*

几何（[`AppFrame.module.css:110-119`](../../../packages/client/ui-layout/src/client/AppFrame.module.css)）：

```css
.overlayLayer   { position: absolute; inset: 0; z-index: 20; pointer-events: none; }
.overlayLayer > * { pointer-events: auto; }
```

**推论**：占位者可以铺满整个 `.frame`（含侧栏之上），并自行接管指针事件。**能承载常驻主界面**——代价是它覆盖三栏，不是与它们并列的第四栏。

`conversation.view`（会话 Tab 环）是 `scope: session` ⇒ 阅读器放那里必然隶属会话 ⇒ **出局**。

### 2.4 `ctx.fs` 的读路径不受策略约束

[filesystem.md](../../../docs/subsystems/filesystem.md)：`fs-observation-policy` 只裁决 `fs/write-intent` 与 `fs/edit-intent` 两个单槽瀑布（read-before-write），**读不经过策略门**。`ctx.fs` 原语含 `resolve` `processPath` `fileUrl` `contains` `stat` `listDir` `readText` `streamText` `readBytes`。

**推论**：读任意本地路径、列目录、取图片字节、流式读大文档，全部现成。阅读器 Host 侧的 IO 无需自建。

### 2.5 `MarkdownText` 无法直接服务阅读器

[`packages/client/ui-primitives/src/markdown/render.tsx:1-17`](../../../packages/client/ui-primitives/src/markdown/render.tsx) 的模块头明写其**未受信输出策略**：

- 图片**必须是绝对 HTTP(S)** ⇒ 本地文档里的相对图片**不渲染**
- **fragment 锚点不通过白名单** ⇒ 文内跳转链接失效 ⇒ **TOC 点击无法靠链接实现**
- 原始 HTML 渲染为字面文本
- 「渲染出的 DOM 被 `tests/fixtures/markdown-dom` 逐字节钉死，不得漂移」

且 `parse.ts` / `incremental.ts` **未从包入口导出**（`src/index.ts:60-66` 只导出 `CodeBlock` `JsonBlock` `MarkdownText` `extractMarkdownPlainText`）。

**推论**：阅读器需要**自有的 mdast→React 渲染臂**。这不是重复造轮子——`micromark` / `mdast-util-*` 是普通 npm 依赖，直接用；`CodeBlock` 已导出可复用。**TOC 必须由界面驱动滚动**，而不是文内链接——这反而更好：大纲成为**接缝词汇**（`DocumentOutline`），而不是渲染器的私有产物。

### 2.6 仓外插件**不能**新增持久会话事件类型

三条事实合成一个硬约束：

1. `KNOWN_SESSION_EVENT_TYPES`（[`packages/core/session/src/known-event-types.ts`](../../../packages/core/session/src/known-event-types.ts)，生成产物）的文档注释原话：*"Every `SessionEventMap` member declared **in this repository**… **Downstream (out-of-repo) plugin events are outside this list by construction.**"*
2. [`storage-contract.ts:75`](../../../packages/session/session-persistence/src/storage-contract.ts) 的重载校验：
   ```ts
   if (!KNOWN_SESSION_EVENT_TYPES.has(event.type) && event.ignorable !== true) throw unsupported(...)
   ```
3. [`Session.append()`](../../../packages/core/session/src/index.ts) 的签名**没有设置 `ignorable` 的入口**——非 surface 事件的可选参数是空元组，构造的事件字面量里也从不写入该字段：
   ```ts
   append<T extends SessionEventType>(
     type: T, data: SessionEventMap[T],
     ...opts: T extends SurfaceEventType ? [opts: SurfaceIntent<T>] : []
   ): SessionEvent<T>
   ```

⇒ **通过 `Session.append` 写入的仓外事件类型，会话重载时必被拒绝**（`SessionFormatUnsupportedError`）。全仓无任何一处生产代码设置 `ignorable`；它只被读取与保留。这与 [外部插件 ignorable 保留决策](../../../.agents/notes/implemented/architecture/2026-08-30-retain-ignorable-external-session-events.md) 一致——该笔记明写 *"First-party writers do not set `ignorable` through `Session.append`"*，并把「按事件名注册已知类型」作为方案**明确否决**（理由：注册事件名不能分类「省略是否安全」，且会让读取依赖读者的当前组合）。

**唯一的合法逃逸口**：`ignorable` 被 **seed 校验**接受（[`index.ts:222`](../../../packages/core/session/src/index.ts) 的 `assertSessionEventEnvelope` 白名单含 `ignorable`），即会话创建时经 `CreateSessionOptions.seed` 注入。仅创建时可用。

**本设计不走 seed 逃逸口**，改为 §4.2 的方案——理由见那里。

### 2.7 会话无法对用户隐藏

`SessionHeader.origin` 是**只有一个值的封闭联合**：[`packages/core/session/src/index.ts:124`](../../../packages/core/session/src/index.ts) 校验 `record.origin !== undefined && record.origin !== 'subagent'` 即抛错。它的用途由 persistence.md 定义为「为子代理子会话准备的粗粒度产品分类，让产品导航隐藏重复的子行」。

而 `'subagent'` 这个值在四处触发子代理专属行为：[`acp/src/index.ts:249,309`](../../../packages/acp/acp/src/index.ts)、[`session-controller/src/history.ts:339`](../../../packages/api/session-controller/src/history.ts)、[`session-controller/src/agent.ts:85`](../../../packages/api/session-controller/src/agent.ts)、[`client/file-upload/src/index.ts:232`](../../../packages/client/file-upload/src/index.ts)。借用它来隐藏阅读会话会连带改变这四处的行为——**是错的**。

⇒ **每一个被创建的会话都会出现在会话列表里，没有第三方可用的隐藏机制。** 这条直接框定 §4.4 的选项空间：任何「每次交互新建一个会话」的设计都必须自己承担列表被淹没的后果。

---

## 3. 核心判断：阅读不是一个能力，是三个

ow 的 `local-reader/surface.ts`（871 行）把三件事混在一起。dsh 的分解纪律要求拆开：

| 关注点 | 性质 | 归属 |
| --- | --- | --- |
| **文档访问** — 给我这个文档的标识、标题、大纲、可寻址内容 | **能力接缝**（Definition / Provider / Consumer 三角） | `ctx.documents` |
| **阅读状态** — 位置、书签、标注、最近打开 | **持久数据**，不是接缝 | storage domain |
| **阅读助读** — 划词 → 提问 | 既有 agent/session 能力的**消费者** | 无新基础设施 |

**为什么这个拆分是整个设计的支点**：只有当「文档访问」是接缝，判据 3（模型也能用）才成立。接缝的价值不是 IO——那是 `ctx.fs` 的活——而是**结构词汇**：`ctx.fs` 给你路径上的字节，`ctx.documents` 给你一个*文档*（标识、大纲、块级可寻址、锚点），且提供方可插拔（今天 md/txt，明天 epub/pdf/网页）。

对照 dsh 既有接缝：`ctx.web` 用一个接缝承载 search + fetch 两个操作，理由是「一个提供方选择策略、一套错误词汇、一份面向产品的配置 API」。`ctx.documents` 同理。

---

## 4. 关系模型：`document ↔ session`

> 这是 [memo D1](../../memos/2026-09-06-reader-migration-context.md) 的答案。此前的三个候选（无实体 / 书库目录即 workspace / 新增 documentRegistry）**全部放弃**。

### 4.1 为什么 `documentRegistry` 是错的

`workspace` 敢维护一份 `sessionIds` 账目，是因为它背后有 `SessionHeader.cwd` 做交叉校验——账目漂了，cwd 是真相。

`documentRegistry` 没有对应物（§2.1）。按 [`packages/AGENTS.md`](../../../packages/AGENTS.md) 的规则，「独立观察可能发散」的关系**必须发布 `./invariant`**——而这个 invariant 将**无物可校验**。一个写不出 invariant 的账目，就是一个会静默腐烂的账目。

### 4.2 正解：两个方向，两套机制

**方向一 `session → document`：会话投影，日志派生，真相来源。**

§2.6 排除了新增事件类型的做法。**附着事实改为搭载在已知事件的载荷里**——`user/message` 的 `source`：

```ts
declare module '@deepseek-ai/dsh-llm' {
  interface MessageSourceMap {
    /**
     * 一条注入上下文消息的出处：用户正在读的文档，以及被选中的那一段。
     * 用户的提问/动作文本走另一条 source.kind: 'user' 的消息（§7.2）。
     */
    'reading-selection': {
      kind: 'reading-selection'
      document: DocumentId
      title: string              // 附着时的展示标题，供列表渲染无需打开文档
      anchor: DocumentAnchor
      excerpt: string
      action?: string            // 触发它的动作按钮，出处元数据；自由提问时缺省
    }
  }
}
```

**为什么这是安全的**（已核实）：`user/message` 是**已知事件类型**，重载校验只看 `event.type`；而其载荷里的 `source` 只被校验为「一个带非空字符串 `kind` 的对象」（[`index.ts:358-363`](../../../packages/core/session/src/index.ts) 的 `assertMessageEventShape`）——**任意 `kind` 都被接受**。只有 `assistant/message` 被强制要求 `kind === 'model'`。BFF 线协议的 `SessionWireEvent.data` 是 `JsonValue`，同样开放。

⇒ 带 `reading-selection` 来源的会话，**在原生 dsh 里也能正常重载**。零新增事件类型，零 `ignorable` 依赖。

投影单元照常：

```ts
// ProjectionDefinition<'reading'>
init:  () => ({ document: null, title: null })
apply: (s, e) => s.document === null
  && e.type === 'user/message'
  && e.data.source.kind === 'reading-selection'
    ? { document: e.data.source.document, title: e.data.source.title }
    : s                    // 同一引用 ⇒ 零下游开销（框架契约）
wire:  { /* 客户端直接拿到成品值，不折叠事件 */ }
```

一个会话「关于哪个文档」于是成为**日志的函数**，与会话标题同构。框架驱动、纯同步、可持久化缓存——全部白送。

**代价（明写）**：附着由**第一条阅读来源消息**建立，不是由会话创建建立。所以「打开了文档但还没问过任何问题」的会话不出现在该文档的列表里。这其实更诚实——会话与文档的关系由它**真的讨论过该文档**构成，而不是由一次打开动作构成。

**方向二 `document → sessions`：可重建索引，缓存，非真相。**

存于 `reading-store` 的 storage domain，`layout: 'per-record'`（该布局的 JSDoc 自述用途即「投影缓存——记录大、稀疏或可单独丢弃」，见 [`storage-domain/src/spec.ts:38-48`](../../../packages/storage/storage-domain/src/spec.ts)）。

重建路径完全由已有 API 组成：

```
ctx.sessionQuery.filterSessions([...])                                    // 全量语料
  → 对每个 id：filterEvents(id, [{ kind: 'type', values: ['user/message'] }])
  → 取首条 source.kind === 'reading-selection' 的消息，读出 document
  → 按 document 分组
```

**这个索引可以且必须发布 `./invariant`**：*索引中每个 `sessionId` 都存在对应会话，且其日志含一条 `source.kind === 'reading-selection'` 且指向该 `document` 的 `user/message`*。校验有物可依（方向一），发散可检测，重建有路径。这正是 `packages/AGENTS.md` 要求 invariant 的场景，也是与 §4.1 的关键差别。

> 精度取舍：`SessionEventResultFilter` 只能按事件**类型**过滤，`source.kind` 需在进程内二次筛。重建是低频操作（首次建索引 / invariant 报警后修复），可接受。

### 4.3 由此得到的产品语义

- 每个附着文档的会话都在文档的会话列表里——右栏那个列表（§8）就是它。
- 「多个 session 关于同一个文档」不是被实现出来的，是**从机制里掉出来的**。
- 阅读状态（位置/书签/标注）不进会话日志，**生命周期彻底独立于 session**——判据 1 成立。
- 会话在原生 dsh 里**可以正常打开**（§4.2 已核实）：它只是一批普通 `user/message`，`source.kind` 不被识别时退化为一个未知来源标签，对话本身完整可读、可续。

> ✅ **原 V1 已查实并作废**：曾计划的 `reading/attach` 独立事件在 §2.6 的约束下**根本不可行**——仓外事件类型重载必被拒，且 `Session.append` 没有 `ignorable` 入口。现方案不新增任何事件类型。

### 4.4 阅读会话的生命周期（**V2，未裁定**）

> **状态：已随 V9 落定为选项乙**（§9.2）——一张助读卡片就是一条会话，所以一个文档的会话数等于卡片数。本节保留三个选项的完整比较，因为它记录的边界事实与代价仍然是判断依据；乙那一列的弱点（列表淹没）改由 §9.3 的三条缓解承担，而不是靠改选甲或丙回避。

#### 框定选项空间的三条事实

1. **会话藏不掉**（§2.7）。没有第三方可用的隐藏机制，每个被创建的会话都进列表。
2. **fork 是一等能力，且可反查**。`ctx.agents.create({ seed, inheritedEventCount, meta: { isSeeded: true, parentSession } })` 创建分叉；`SessionResultFilter` 含 `{ kind: 'parent'; values }`（§2.1），所以**父子关系不需要我们自己维护索引**。
3. **一个会话只属于一个文档**（§4.2 的投影取首条阅读来源消息）。

#### 三个选项

| | 甲：每文档一条常驻 | **乙：每张卡一条（已采纳）** | 丙：一条常驻 + 显式 fork 深度会话 |
| --- | --- | --- | --- |
| 会话数 | = 文档数 | = 卡片数（读一本书可达 20+） | 1 + N，N 由用户显式创建 |
| 列表可读性 | 好 | **会被淹没**（事实 1），靠 §9.3 三条缓解 | 好，且有父子层级 |
| 卡内连续性 | 连续 | **连续**（追问 `followup()` 进同一条） | 连续 |
| 跨卡连续性 | 有：第 20 张能引用第 3 张 | **无**（这正是卡片独立的定义） | 常驻内有；fork 继承到分叉点 |
| 并发 | **否**（串行 driver，多卡排队） | **是**（独立 agent） | 否 |
| KV cache | 一条长前缀，命中率高 | 逐卡追加式前缀；跨卡若组合相同，供应商侧前缀缓存或可命中（V12） | 同甲 |
| 压缩影响 | 会话会长，早期划词迟早被压缩掉 | 单卡短，通常不触发 | 常驻会长；深度会话短 |
| 父子关系 | — | — | **`parentSession` 白送，可 `filterSessions` 直查** |

#### 为什么最终是乙

乙原本被否的唯一理由是列表淹没（事实 1）。V9 把卡片做成会话之后，这条理由有了三条独立于会话机制的缓解（§9.3 二）：阅读接管期侧栏不可见、右栏列表按文档过滤、独立产品可摘 `ui-workspace`。而选甲或丙要付的代价是把多张卡片挤进一条会话——那会让卡片彼此可见、彼此串行，正好抵消卡片这个交互形态的全部价值。

丙的 fork 机制因此不进 v1。若日后需要「把某张卡的讨论另存一份」，`ctx.agents.create({ seed, inheritedEventCount, meta: { isSeeded: true, parentSession } })` 仍然可用，且 `SessionResultFilter` 的 `{ kind: 'parent' }` 让父子关系不需要我们维护索引。

#### 延伸问题（不阻塞 M1–M8）

**卡片会话的压缩策略。** 一张卡追问几十轮后会变长。dsh 有 compaction 接缝，但阅读场景下「压缩掉早期追问」的可接受度未知。等真实长卡片出现后再定，不预设。

---

## 5. `ctx.documents` 的类型面

**这是本设计里唯一需要一次想对的东西**——它同时是阅读界面、存储层和未来模型工具的共同语言。

### 5.1 标识

```ts
/**
 * 一个文档的稳定标识：提供方限定的规范化 URI。
 * 本地文件为 `file://` + `fs.realpath` 规范化路径——与 workspace 的路径唯一性
 * 判准同源（尾斜杠、`..`、符号链接全部解析）。
 * 确定性构造：无需注册表即可从路径得到标识。
 */
type DocumentId = Branded<'DocumentId'>
```

**取舍（明写）**：`workspace` 选 uuid 而非路径，理由是「路径规范化会重写路径，而引用锚点必须稳定」。文档这里**反向选择**——用路径派生标识，换掉整个注册表。代价是**重命名/移动会断开关系**。v1 接受，记入已知限制；修复路径见 §11。

### 5.2 内容与结构

```ts
/** 一个文档的静态身份与结构，不含正文。 */
interface DocumentRef {
  readonly id: DocumentId
  /** 展示标题：frontmatter title → 首个 H1 → 文件名。由提供方 resolve，不是消费者兜底。 */
  readonly title: string
  /** 提供方名（`local` 等），用于诊断与能力分派。 */
  readonly provider: string
  /** 媒体类型（`text/markdown` | `text/plain` | …），提供方判定。 */
  readonly mediaType: string
  /** 后端版本令牌，语义等同 `FsVersion`：用于检测外部改动。 */
  readonly version: string
  /** 字节数与字符数，供大文档策略与进度显示。 */
  readonly size: { readonly bytes: number; readonly chars: number }
}

/** 层级大纲。TOC 由界面驱动滚动，不依赖文内链接（§2.5）。 */
interface DocumentOutline {
  readonly entries: readonly DocumentOutlineEntry[]
}
interface DocumentOutlineEntry {
  readonly depth: number          // 1–6
  readonly text: string
  readonly anchor: DocumentAnchor // 与标注共用同一锚点词汇
  readonly children: readonly DocumentOutlineEntry[]
}
```

### 5.3 锚点——本设计的原创部分

位置、书签、标注、划词助读**共用一套锚点**。文档会在外部被编辑，锚点必须能重定位。

```ts
/**
 * 一个可重定位的文档位置。三层冗余，按可靠性递降：
 * 结构路径定位块，引文验证身份，字符偏移做兜底与排序。
 */
interface DocumentAnchor {
  /** 块级路径：mdast 根到目标块的子序号序列。文档结构不变时精确。 */
  readonly blockPath: readonly number[]
  /**
   * 引文选择子（W3C Web Annotation 的 TextQuoteSelector 形状）。
   * `exact` 是被锚定文本，`prefix`/`suffix` 是各 32 字符上下文。
   * 结构变了但文本还在时，靠它重新定位。
   */
  readonly quote: { readonly prefix: string; readonly exact: string; readonly suffix: string }
  /** 锚定时的文档字符偏移。仅用于排序与重定位起点，从不单独作为真相。 */
  readonly charOffset: number
  /** 锚定时的文档版本；与当前版本不同即触发重定位。 */
  readonly version: string
}

/** 重定位结果。失败是一等状态，不是异常——文档被大改是正常情况。 */
type AnchorResolution =
  | { readonly kind: 'exact'; readonly charOffset: number }
  | { readonly kind: 'relocated'; readonly charOffset: number; readonly confidence: number }
  | { readonly kind: 'lost' }
```

重定位顺序：`blockPath` 命中且 `quote.exact` 匹配 → `exact`；否则以 `charOffset` 为中心做有界模糊引文搜索 → `relocated`（带置信度）；再否则 `lost`（标注保留、标记为孤儿，**不删除**）。

**为什么锚点属于接缝而不是界面**：模型工具要能说「在第 3 段这句话上加条批注」，标注层要能说「这条批注现在在哪」，两者必须是同一个词汇。ow 把 `anchoring.ts` 和 `marks.ts` 分放两处正是这个问题。

### 5.4 服务面

```ts
interface Documents {
  /** 提供方注册；返回 disposer（注册即效果）。 */
  register(name: string, provider: DocumentProvider): () => void

  /** 路径/URI → 标识与身份。提供方决定标题与媒体类型的 resolve，无隐式兜底。 */
  resolve(request: DocumentResolveRequest, signal?: AbortSignal): Promise<DocumentRef>

  /** 读正文。`window` 缺省为全文；大文档由消费者分窗。 */
  read(id: DocumentId, window?: DocumentWindow, signal?: AbortSignal): Promise<DocumentContent>

  /** 大纲。提供方可缓存于 version 之下。 */
  outline(id: DocumentId, signal?: AbortSignal): Promise<DocumentOutline>

  /** 锚点重定位。 */
  locate(id: DocumentId, anchor: DocumentAnchor, signal?: AbortSignal): Promise<AnchorResolution>

  /** 列出一个容器（本地目录）内的候选文档。 */
  list(container: string, signal?: AbortSignal): Promise<readonly DocumentListEntry[]>

  /** 内嵌资源（图片）字节。界面经 Host 路由取，不让浏览器碰本地路径。 */
  asset(id: DocumentId, ref: string, signal?: AbortSignal): Promise<DocumentAsset>
}
```

按 [`AGENTS.md`](../../../AGENTS.md) 的 explicit-over-implicit 规则：**默认值是提供方实现里显式的 `resolve(request): Spec` 步骤，不是 `read()` 里的 `?? default`**。

---

## 6. 包划分

命名遵循 [adding-a-package.md](../../../docs/cookbook/adding-a-package.md) 的「给存在的角色命名」。（发布时换成自有 scope。）

### v1 —— 六个包

| 包 | 角色 | 内容 |
| --- | --- | --- |
| `dsh-documents` | **Service Definition** | `ctx.documents` + §5 全部类型 + 提供方注册表。**不做 IO。** |
| `dsh-documents-local` | **Service Provider** | 本地文件系统实现，全部经 `ctx.fs`。md/txt 解析、标题 resolve、大纲抽取、锚点重定位、目录列举、图片字节。 |
| `dsh-reading-store` | 持久数据形态 | storage domain：位置 / 书签 / 标注 / 最近打开 / **卡片的表现状态**（位置、停放、dock 顺序）+ `document → sessions` 索引。**发布 `./invariant`**（§4.2）。卡片的轮次不在这里，在会话日志。 |
| `dsh-reading-session` | **Consumer**（会话侧） | 卡片会话的创建与驱动（`ctx.agents.create` + 卡片专属 `setup` 组合、`followup`）、面向客户端的 `@Remote` 方法（含 §7.4 的资源读取）。拥有 `reading-selection` 的 `MessageSourceMap` 扩展（含 `card` 字段）、`reading` 投影单元、系统提示词段。**不新增会话事件类型**（§2.6）。 |
| `dsh-client-ui-reading` | **Consumer**（界面） | `shell.overlay` 占位者：书库 / 文件树 / 阅读视图 / 大纲 / chrome / 键位 / 自有 markdown 渲染臂。 |
| `dsh-client-ui-reading-assist` | **Consumer**（界面） | 选区模型、动作菜单、助读卡片（几何、级联、视口钳制、追问框）、停放 dock，以及**右栏**：本文档的会话列表、完整转录、切换 / 清空 / 导出（§8、§9）。 |

### 延后的拆分

按「角色独立演化时才拆」的规则，下列**先不建**：

- `dsh-tool-documents` —— 模型工具（`document_outline` / `document_read` / `document_locate`）。**判据 3 靠它兑现**，但可以在 M6 之后加，且加它**不需要改接缝**——这正是接缝设计的回报。
- `dsh-client-ui-reading-annotations` —— 标注层，v1 先住在 `client-ui-reading` 里。
- `dsh-documents-epub` / `-pdf` / `-web` —— 后续提供方。

**依赖方向**：`documents-local` → `documents` ← `reading-session` → `reading-store`；客户端两包只经 wire 与 slots，**互不导入运行时值**（[slots.md 扩展规则](../../../docs/subsystems/slots.md#extension-rules)）。

### 客户端包的构建约束（V3 查证结果）

两个 `client-ui-*` 包在 `package.json` 声明 `dsh.client.platform: 'web'` 并导出 `./client`；Host 侧**扫描已启用的 Loader 条目并在 `/plugins` 下供给，无需逐插件接线**（[client-modules README](../../../packages/client/modules/README.md)）。三条必须遵守：

1. **启动前 `lib/client.js` 必须已构建**，缺失时**响亮失败**并打印构建指令与包路径列表——不是静默降级成空白界面。
2. **共享模块基线是冻结的**（[`packages/client/web/src/platform.ts`](../../../packages/client/web/src/platform.ts)）：
   ```ts
   export const PLATFORM_MODULES = [
     'react', 'react/jsx-runtime', 'react-dom', 'react-dom/client', '@deepseek-ai/cordis',
     '@deepseek-ai/dsh-client-store',
     '@deepseek-ai/dsh-client-ui-slots',
     '@deepseek-ai/dsh-client-ui-primitives',
   ] as const
   ```
   **`ui-primitives` 在基线内** ⇒ §2.5 说的 `CodeBlock` 复用零成本，`Modal` / `Menu` / `Tooltip` 等同理。
3. **`micromark` / `mdast-util-*` / `mermaid` 不在基线内** ⇒ 打进本包自己的 bundle（tsdown 默认行为）。`dsh.client.external` 只用于跨插件共享**非基线**模块，本设计不需要。

---

## 7. 三条数据流

### 7.1 打开一个文档

```
界面：选目录（复用 ui-directory-picker-native / -browse）
  → @Remote list(container)         → ctx.documents.list → ctx.fs.listDir
  → @Remote open(path)              → ctx.documents.resolve → DocumentRef
                                    → reading-store 读回上次位置/书签/标注
  → @Remote content(id, window)     → ctx.documents.read + outline
界面：自有 mdast→React 渲染臂渲染；图片经 @Remote asset() 取 base64 转 object URL（§7.4）；
      大纲驱动滚动；滚动位置节流写回 reading-store
```

**全程不涉及 session。** 判据 1 在数据流层面成立。

### 7.2 划词助读

```
界面：选区 → DocumentAnchor + excerpt + action
  → @Remote assist({ card, document, anchor, excerpt, action, question? })
reading-session：
  ① 取或建「这张卡片的会话」（ctx.agents.create，带卡片专属的最小组合；§9.2）
  ② append 两条 user/message，顺序照 dsh 自己的 [...claimed, context] 惯例：
     a. 用户的提问或动作文本  → source.kind: 'user'
     b. 选中的原文 + 定位     → source.kind: 'reading-selection'   ← 投影来源
  ③ 正常 turn
  ④ 流式结果经既有 session/event 推送回界面
追问：同一张卡再问 → handle.agent.followup()，进同一条会话
```

**一张卡就是一条会话**（§9.2）。所以追问天然连续、刷新天然可重建、Agent 天然看得见——都不需要额外机制。「转到完整会话」不是升级，是**把这条会话在右栏里换一种渲染**（§8）。

**卡片 agent 用最小组合。** `ctx.agents.create({ setup })` 让每个 agent 拥有自己的作用域工具与提示词段（[agent README](../../../packages/core/agent/README.md)）。卡片 agent 只带阅读相关工具，不带 shell / fs 写 / subagent。**依据**：录制快照实测，`text-turn` profile 的系统提示词 4,712 字节 + 工具 schema 36,003 字节 ≈ 40.7 KB，`agent-instructions` 达 82.6 KB；若卡片 agent 继承完整组合，每张卡都背这份前缀。

**为什么拆成两条消息**（V7 查证的结果）：`ui-chat` 的 `messageDefinition` 在 [`message.ts:55`](../../../packages/client/ui-chat/src/client/conversation-nodes/message.ts) 判 `source.kind !== 'user'` 就归为 **`context` 节点**（"Non-user context injected into model history"）。若把提问也塞进 `reading-selection` 一条里，它在任何 dsh 客户端里都会被渲染成"注入的上下文"而不是用户发言——**语义错了**。拆开之后：提问在哪都是正常的用户气泡，原文在哪都是注入上下文，与 dsh 对这两件事的既有建模完全一致。

**降级观感已核实**：未知 `source.kind` 落到 [`event-projection.ts:73-76`](../../../packages/client/ui-chat/src/client/conversation-nodes/event-projection.ts) 的文档化默认——

```ts
default:
  // MessageSourceMap is merge-extensible; keep an unknown producer
  // visible by its durable kind.
  return { role: 'inject', label: kind }
```

⇒ 原生 dsh 里显示为一条标着 `reading-selection` 的注入上下文，**可见、可读、不报错**。我们自己的产品用 `ctx.uiConversation.events.register()` 注册专属节点定义，把这一对渲染成一张阅读卡片。

**「模型可见 ⟺ 已记录」检查**：进入模型请求的是提问、excerpt（都在消息内容里）与文档标题（在系统提示词段里），全部落在同一条 `user/message` 里——内容在 `content`，标题在 `source.title`。✓ 卡片走真实 turn，这条规则原样适用、原样满足。

**KV cache 检查**：文档正文**绝不进系统提示词**。系统段只含稳定的「用户正在阅读《标题》」——一张卡绑一个文档，该段在会话生命周期内不变 ⇒ 前缀稳定；追问以追加消息进入 ⇒ 同一张卡天然缓存友好。（依据：实测该 Web 会话缓存命中 89%，注入策略必须保住它。）

**跨卡前缀**：不同卡片是不同会话，但若卡片 agent 的组合相同，它们的系统提示词与工具 schema **逐字节相同** ⇒ 供应商侧的前缀缓存仍可命中这一段。**未实测**，记为 V12 的一部分。

### 7.3 标注

```
界面：选区 → DocumentAnchor + note
  → @Remote annotate(...)  → reading-store（不进会话日志）
渲染时：ctx.documents.locate(id, anchor) 逐条重定位
        exact/relocated → 画；lost → 收进「孤儿标注」抽屉，不删
```

> **未决 D6′**：标注若要进入助读 prompt，按「模型可见 ⟺ 已记录」它就**必须**是会话事件而非仅 KV。当前设计选择：标注是**阅读状态**（存 store），进入 prompt 时作为**消息内容**随 `reading-selection` 一起落日志。这样两条规则都不破。**待确认此解读成立。**

### 7.4 文档内嵌图片

**不注册新的 HTTP 路由。** 图片字节经 Typert `@Remote` 走 `/api`，与 dsh 自己传送会话图片的做法一致（[`session-controller/src/commands.ts:390-393`](../../../packages/api/session-controller/src/commands.ts)）：

```ts
const stored = await this.ctx.attachments.readImage(ref)
return { attachment: stored.ref, data: Buffer.from(stored.data).toString('base64') }
```

三条理由，每条都是硬的：

1. **鉴权白送**。`ctx.webServer` 自述 *"carries no TLS, authentication, or origin policy of its own"*；策略归路由所有者。`/api` 的所有者 `client-connection` 已实现 **Host/Origin 浏览器信任围栏 + 持久浏览器认证**（`requestRejection(req)` → 401/403，`BrowserAuth` 经 `ctx.credentials` 发 cookie，`trustedHosts` 在加载期 `assertTrustedAuthority` 校验——误配置响亮失败）。自建路由等于把这套重写一遍。
2. **Electron 可用**。webserver README：*"It serves browsers only; Electron loads dist over `file://` and carries fetch over an IPC bridge."* 自建 HTTP 路由在 Electron 下**直接失效**；`/api` 已由 IPC 桥承载。
3. **授权模型可抄**。dsh 的图片方法先做 `referencedImage(source.events, attachmentId)`，未被该会话引用则 `ATTACHMENT_NOT_REFERENCED`。**这条必须照抄**：`asset(documentId, ref)` 必须验证 `ref` 确实出现在该文档解析出的 mdast image 节点里。否则 `asset(doc, '../../../.ssh/id_rsa')` 就是一个任意文件读原语——**这是本设计里唯一的真实安全风险点**。

代价：base64 约 +33% 体积，无 HTTP range/流式。缓解：图片尺寸上限走 `Config`，客户端把结果转 object URL 并缓存。

---

## 8. UI 落位

**结论：`shell.overlay` 全框架占位者。** 依据 §2.3——这是 dsh 为「自有全框架界面」预留的正门，零上游改动，root 作用域。

**代价（明写）**：它覆盖三栏（含侧栏），是**阅读模式接管**，不是与对话并列的第四栏。因此：

- 侧栏在接管期不可见 ⇒ 阅读器需自带返回入口。**附带好处**：卡片会话不会在阅读期出现在全局列表里（§9.3 缓解一）。
- **overlay 内部的布局完全由我们定**（`.overlayLayer` 铺满 `.frame` 并自行接管指针事件，§2.3），所以右栏是自建的，不占用 dsh 的任何槽位。

### 右栏：同一批会话的另一种渲染

阅读界面内自建右栏，承载三件事：

| | 内容 | 来源 |
| --- | --- | --- |
| 会话列表 | 本文档的全部卡片会话 | `reading-store` 的 `document → sessions` 索引（`SessionResultFilter` 没有 document 轴，§2.1） |
| 转录 | 当前选中会话的完整视图 | 该会话的日志 |
| 操作 | 切换 / 清空 / 导出 | 见下；「清空」已裁定，导出待定（V14） |

**卡片与右栏是同一批会话**（§9.2），只是渲染不同：卡片贴着正文、精简；右栏完整。「转到完整会话」= 在右栏里选中这张卡的会话，**不关闭阅读器**，也不做任何数据搬运。

**转录视图自建。** `ui-chat` 只导出类型与 `apply` / `inject`，没有可复用组件；`conversation` 槽是 `kind: 'single'`，注册即**替换整个对话界面**（[`ui-layout/src/client/index.ts:55-65`](../../../packages/client/ui-layout/src/client/index.ts)），无法嵌套进 overlay。与 §2.5 自建 markdown 渲染臂同性质。

**清空（已裁定）：该卡片改挂一条新会话，旧会话从 `document → sessions` 索引移除。**

读者得到的是一张空白的卡片，旧对话从右栏列表与卡片里彻底消失、不可再达。

**它不删除磁盘记录，因此这个控件不能叫「删除」。** dsh **没有删除会话的 API**，三条已查实：

- `dispose()` 的拆解顺序是「stop-and-drain the loop, unwind the scope, detach the agent, **detach the session**」（[agent README](../../../packages/core/agent/README.md)）——detach 是从内存注册表摘掉，磁盘 JSONL 不动。
- `session-persistence-jsonl` 里唯一的 `rm()` 是迁移暂存文件的清理（[`generation.ts:514`](../../../packages/session/session-persistence-jsonl/src/generation.ts)），不是会话删除。
- `SessionAvailability` 只有 `'live' | 'persisted'`（[session-query.md](../../../docs/subsystems/session-query.md)），没有 archived 或 deleted 态；`ui-workspace` 提供的 archive 是 workspace 层动作。

且 [根 AGENTS.md](../../../AGENTS.md) 对已发布 Session JSONL 的规则是 *never move, overwrite, or delete committed generations*。越过 `session-persistence` 自行删文件既绕过所有者又撞该规则，**不做**。

⇒ 「清空」是准确的措辞：它承诺视图被清空，不承诺磁盘。旧 JSONL 成为一条不可达记录，记入已知限制（§11）。

**保存 / 导出（V14 余项）** —— 会话本就持久（`session-persistence` 落 JSONL），「保存」若指持久化则是冗余控件，应当只意味着导出为 markdown。导出自建，从会话日志渲染。**待裁定：是否保留「保存」这个独立控件，还是只留「导出」。**

**放弃的备选**：fork `ui-layout` 增第四栏——产品形态更好，但持续承担上游合并成本。**推迟到接管形态被实际使用否决之后再考虑**，不预支。

**路由**：dsh 客户端**没有任何路由**（全量 grep `pushState`/`popstate`/`location.*` 在产品代码零命中）。阅读器开合与当前文档是客户端 store 状态。ow 的 ADR-038（URL 状态）v1 放弃——阅读位置本就durable 在 `reading-store`，「重开回到上次位置」不依赖 URL。

---

## 9. 助读卡片：一张卡就是一条会话

> §7.2 给出数据流。本节定义卡片的归属、并发、停放与右栏的关系。
>
> ow 侧事实一律标注 `文件：行`，以 `/Users/zibuyu/code/zibuyu/open-writer` 为准；dsh 侧事实标注仓内路径。

### 9.1 ow 现状：它不是会话，是 N 次一次性请求

**卡片的诞生与冻结。** 划词先弹 ephemeral trigger（动作条）；点动作才 spawn 一张 persistent card。card 在 spawn 时冻结 `selection` / `bookId` / `chapterId` / `initialAction` / `sourceLabel`，此后自足——翻页、切章都不影响它继续浮着（`assist-windows.ts:99-122`）。`sourceLabel` 在本地阅读器里是**文件 basename 而非绝对路径**，理由写在注释里：capsule 会出现在读者截的每一张图的边上。

**动作分两组七项**：`translate` / `explain` / `grammar-en` / `lecture`（理解）与 `fix` / `polish` / `rephrase`（修改），外加不是按钮的自由提问 `ask`。每项声明 `contextMode: 'none' | 'neighbor' | 'wide'`，客户端按档取选区邻域，服务端按档二次截断（2400 / 24000 字符）。

**三条实测事实：**

| | 事实 | 出处 |
| --- | --- | --- |
| 1 | **卡内无连续性。** 请求体只有 `{actionId, selectedText, context, userPrompt, uiLang, bookId, chapterId}`，**不带任何历史轮次**；服务端每次 `buildAssistMessages(id, input)` 从零拼 messages。同一张卡的第 3 轮看不见第 1 轮，`state.turns` 只是客户端显示历史 | `AssistWindow.tsx:250-265`、`actions.ts:454` |
| 2 | **不持久。** assist 相关文件无任何 `localStorage` / `sessionStorage`。刷新 = 20 张卡连同全部答案消失 | 全量 grep 零命中 |
| 3 | **模型看不到文档，也用不上工具。** `AssistRunner` 只有 `streamChat` / `chat` 两个方法，没有 agent 循环 | `assist/types.ts:33-39` |

三条都是本设计要修掉的。卡片**彼此独立**这一点保留。

### 9.2 归属模型：一张卡就是一条会话

**结论（V9 已裁定）：每张助读卡片对应一条独立会话，由 `ctx.agents.create()` 创建。**

```
卡片 X（80.md 第 2 段）  ←→  会话 X
├─ user  「翻译」                      source.kind: 'user'
├─ user  锚点 + 引文                   source.kind: 'reading-selection'  ← 投影来源
├─ assistant  Li Jia sat surrounded…
├─ user  「为什么用 perched」          ← followup()，同一条会话
└─ assistant  …                        ← 看得见上面全部

卡片 Y（80.md 第 50 段）←→ 会话 Y   （独立 agent，与 X 并发）
```

一张卡片的**内容**（全部轮次）住在会话日志里；一张卡片的**表现**（位置、停没停、dock 顺序）住在 `reading-store`。两者归属分明，各有唯一的家。

**这样成立的六件事：**

| | ow | 本设计 |
| --- | --- | --- |
| 卡片之间 | 独立 | **独立**（各自一条会话、一个 agent） |
| 卡内追问 | 第 3 轮看不见第 1 轮 | **连续**（同一条会话日志） |
| 刷新后 | 20 张卡连答案一起丢 | **可重建**（会话是持久的） |
| 并发 | 多卡同时流式 | **多卡同时流式**（独立 agent，§9.4） |
| 工具 | 用不上 | **可用**（真实 turn，工具集由卡片 agent 的组合决定） |
| Agent / 记忆系统可见 | 否 | **是**（全在会话日志里） |

**「转到完整会话」是换渲染，不是搬数据。** 卡片的会话与右栏里的会话**是同一条**：卡片是贴着正文的精简视图，右栏是完整转录。该动作只改变哪个视图在渲染它，不写入任何事件。

**`reading-selection` 携带 `card`**，与 `document` / `anchor` / `excerpt` 并列，用于把会话与发起它的卡片对上。§4.2 的投影照常工作：一条会话属于哪个文档，仍是首条 `reading-selection` 消息的函数。

### 9.3 裁定 V9 的代价（明写）

**一、每张卡是一条真实会话，不是一次调用。** 一个 agent、一条 JSONL、一次 loop 实例化，比 `ctx.llm.stream()` 重。缓解是卡片 agent 走**最小组合**（§7.2），但首字延迟仍会高于一次裸补全。**未实测**，记为 V12。

**二、会话数等于卡片数。** 这就是 §4.4 的选项乙，其已知弱点是会话列表被淹没（§2.7：`origin` 是封闭联合，借 `'subagent'` 会改掉四处子代理行为，不能用来隐藏）。三条缓解，按可靠性递降：

- **阅读接管期侧栏不可见**（§2.3、§8）。`shell.overlay` 覆盖三栏，所以读者在阅读时根本看不到全局列表；污染只在退出阅读器后可见。
- **右栏的列表是我们自己的**，来自 `reading-store` 的 `document → sessions` 索引，天然按文档过滤（`SessionResultFilter` 没有 document 轴，§2.1）。
- **独立产品可以不挂 `ui-workspace`**——它只是 [`packages/bundle/web-app/cordis.patch.yml:246`](../../../packages/bundle/web-app/cordis.patch.yml) 的一个普通条目。**但这不免费**：`uiWorkspace` 是 `ui-sidebar`、`ui-directory-picker-native`、`ui-directory-picker-browse`、`ui-agent-preset` 四个插件的**硬注入**依赖，摘掉它会连带停掉侧栏外壳与**两个目录选择器**，而 §7.1 正打算复用后者 ⇒ 走这条路就必须自建目录选择器。其余引用（`ui-chat`、`ui-conversation`）是 `import type {}`，纯编译期，不受影响。**取哪条路记为 V13。**

**三、每条会话可能触发标题生成。** `session-title-*-llm` 按 cadence 发辅助模型调用，20 张卡就是 20 次额外调用。自有 profile 里可以关掉或改 cadence，但**必须显式决定**，不能默认带过来。

### 9.4 并发：卡片之间并发，卡片内部串行

**事实：一个 dsh agent 是串行 driver。** `followup()` 把提示**入队**到 `Inbox` 的 `next-turn` 列表，turn 一次消费一条（[`inbox.ts` 的 `claim`](../../../packages/core/agent/src/inbox.ts)），`whenIdle()` 等到整体静默。

**这条约束落在一张卡内部，那正是它该在的地方**——同一张卡的追问本就是顺序的。**卡片之间是不同的 agent，天然并发**：读者可以同时开四张卡等四个答案，与 ow 一致。

ow 自己的注释诚实记下：7 张以上会在 Chrome 每源 6 连接处排队，两次测量实际损害都无效，所以**这个损害从未被测出**。我们的传输不同（Typert `@Remote` 走 `/api`，§7.4），该数字不可沿用；真需要限流时，那是**一条针对在途请求数的独立限制加一个可见的等待状态**，不是把卡片上限调小。

### 9.5 停放：纯表现状态

磁吸 dock 与会话无关，是卡片的位置状态，**存 `reading-store`**。三条 ow 用缺陷换来的教训原样复刻：

- **dock 的 `left` 是常量，不是算出来的**（ow B-207）。原先跟着文本列算，开右轨时一张**已经停好的** capsule 横跳 332px。「停放意味着读者把东西放下了，期望在原地找到它」，而「它不动」**没法靠重算维持**：那要求枚举每一条会移动列的规则，ow 侧 `grep` 出 29 条，无人维护该集合。代价是宽屏下 dock 不再紧贴文本——**可预测的位置比邻接更值钱**，因为拖放手势和回望都依赖它。
- **capsule 宽度只有一个值**（ow B-201）。曾有 36px 竖排作「优雅降级」，实测宽屏阅读模式下 1512 以下**每个**视口都落进去，而一张读不出动作名的 capsule「不是停放的卡片，是一道计数杠」。宽度固定，让**布局**去让位。
- **停放期间流不中断**。ow 靠 portal 保证（unmount 会 abort 流）；本设计里流由会话持有、不由组件持有，所以这是白得的——但仍要有一档测试扣住响应来证明它，因为它是产品承诺而不只是实现细节。

dock 在专注模式下整体消失（含每一个入口），与 ow 一致。

### 9.6 归属自查

| 事实 | 归属 | 理由 |
| --- | --- | --- |
| 卡片的全部轮次、锚点、开场动作 | 该卡的会话日志 | 模型可见 |
| 卡片的位置、停没停、dock 顺序 | `reading-store` | 模型不可见，纯表现 |
| 文档有哪些卡片会话 | `reading-store` 的 `document → sessions` 索引 | 缓存；真相在会话日志（§4.2） |
| 卡片 agent 的工具集与提示词段 | 该 agent 的 `setup` 组合 | 按 agent 作用域，不是全局 |
| 卡片上限、级联步长、停靠阈值、弹出距离 | `Config` | 部署可变的可调项 |
---

## 10. 与仓库规则的逐条自查

| 规则（[AGENTS.md](../../../AGENTS.md)） | 本设计 |
| --- | --- |
| 能力接缝是完整三角，不是单一角色 | `documents`（Definition）+ `documents-local`（Provider）+ `reading-session`/`client-ui-reading`（Consumer）v1 同时存在 ✓ |
| 模型可见 ⟺ 已记录 | §7.2 逐项对照：提问在 `source.kind: 'user'` 的消息里，excerpt 与文档标题在 `source.kind: 'reading-selection'` 的消息里，两条都是 `user/message`，全部落盘。卡片走真实 turn，规则原样适用、原样满足 ✓ |
| 注册即效果 | `documents.register()` 返回 disposer；slots / 投影 / 事件全走 `ctx.effect()` / `ctx.on()` ✓ |
| invariant 只在独立观察可能发散时发布 | 仅 `reading-store` 发布（`document → sessions` 索引 vs 会话语料）；接缝本身不发布 ✓ |
| 显式优于隐式，默认是 `resolve(request): Spec` | 标题/媒体类型/窗口默认全在提供方 `resolve`，`read()` 内无 `?? default` ✓ |
| 插件内无硬编码可调项 | 锚点上下文长度、模糊搜索半径、位置写回节流、最近目录上限 → 全部 `Config` 字段 ✓ |
| 误配置响亮失败 | 未知 provider、未注册 document、越权容器 → 抛错不静默跳过 ✓ |
| 跨边界 id 加品牌 | `DocumentId = Branded<'DocumentId'>` ✓ |
| 闭合联合以 `assertNever` 结尾 | `AnchorResolution` 闭合 ⇒ `assertNever`；`MessageSourceMap` / `action` 合并可扩展 ⇒ 文档化默认 ✓ |
| 界面文案 locale 化 | 两个客户端包各自 typed dictionary ✓ |
| 插件而非改循环 | 不触碰 `agent-loop`；只用 `sessionProjections` / `SessionEventMap` / `MessageSourceMap` / slots 四个既有扩展点 ✓ |
| 类型文件只放类型 | `documents/src/types.ts` 纯类型 ✓ |

---

## 11. 已知限制与取舍

**这些是设计的边界，不是待办清单。**

- **路径派生标识 ⇒ 重命名断链**（§5.1）。修复路径（v2）：`reading-store` 保存附着时的内容指纹，检测到「旧 id 消失 + 新 id 出现且指纹相同」时提供合并操作。**不做自动合并**——静默改写关系比断链更糟。
- **`document → sessions` 是缓存，不是真相**。它可能落后、可能漂移；invariant 检测，重建修复。这是刻意的：真相在会话日志里（§4.2）。
- **阅读器接管全屏**（§8）。侧栏与对话在阅读期不可见。
- **自有 markdown 渲染臂意味着自担安全策略**。`MarkdownText` 的白名单（协议白名单、原始 HTML 字面化、KaTeX 无信任命令）必须**逐条复刻**，本地图片是唯一有意放开的口子，且只经 §7.4 的 `@Remote asset()`——浏览器永不直接拿到本地路径，且服务端必须做引用校验。
- **不能新增持久会话事件类型**（§2.6）。这不是本设计的选择，是仓外插件的结构性约束。任何「给会话加一个新的durable事实」的需求，都只能搭载在已知事件的载荷里，或接受该会话在原生 dsh 中无法重载。
- **锚点在文档被大改时会 `lost`**。孤儿标注保留、可见、不自动删除。
- **「清空」不删磁盘记录**（§8）。dsh 没有删除会话的 API，被清空的卡片会话其 JSONL 留在磁盘上，成为一条产品界面无法再抵达的记录。长期使用会累积；若日后需要真正的清理，那是一条独立的、由 `session-persistence` 所有者提供的删除路径，不是我们能越过它做的事。
- **每条卡片会话可能触发标题生成**（§9.3 三）。`session-title-*-llm` 按 cadence 发辅助模型调用，20 张卡就是 20 次额外调用。自有 profile 里可关可改，但必须显式决定，不能默认带过来。
- **会话数等于卡片数**（§9.3 二）。这是 §4.4 的选项乙。三条缓解（阅读期侧栏不可见、右栏列表按文档过滤、可摘 `ui-workspace`）都在 §9.3，第三条有连带代价，记为 V13。
- **卡片之间互不可见**。各自一条会话，卡片 B 引用不到卡片 A 的结论。这是卡片独立的直接后果，不是缺陷。
- **每张卡是一条真实会话，不是一次调用**（§9.3 一）。一个 agent、一条 JSONL、一次 loop 实例化；首字延迟高于裸补全，缓解是卡片 agent 走最小组合。未实测，V12。
- **v1 只读**。改文件让用户对 Agent 说，`edit`/`write` 工具与权限层本就在。
- **只支持 md/txt**。epub/pdf 是新提供方，不是新接缝。

---

## 12. 里程碑

每个里程碑一个 PR，浏览器内人工验收后再合。**验收条件必须能被一个人在界面上点出来。**

| # | 里程碑 | 验收条件 | 不做什么 |
| --- | --- | --- | --- |
| **M1** | 接缝与本地提供方 | Host 单测：`resolve` 一个 md 文件得到正确 title/mediaType/version；`list` 列出目录；`read` 返回全文 | 无界面、无大纲、无锚点 |
| **M2** | 阅读视图落地 `shell.overlay` | 浏览器里打开阅读器 → 选目录 → 点文件 → 看见渲染后的正文；关闭回到对话 | 无大纲、无位置、无助读 |
| **M3** | 大纲与渲染补全 | 点大纲条目跳到对应位置；本地图片显示；代码高亮；mermaid 渲染 | 无位置持久化 |
| **M4** | 阅读状态 | 关掉重开回到上次位置；书签可加可跳；最近目录可用 | 无标注、无助读 |
| **M5** | 锚点 | 外部改文档后重开，书签仍在正确位置或被标记为孤儿 | 无标注 UI |
| **M6** | 助读卡片与追问 | 划词 → 点动作 → 看见流式回答；**在同一张卡里追问，回答引用得到前一轮**；关掉浏览器重开，卡片连同全部轮次重建；带 `reading-selection`（含 `card`）来源的消息落日志、投影可见；**该会话在原生 dsh 里仍能打开** | 无 dock、无多卡片、无右栏 |
| **M7** | 多卡片、停放与右栏 | 多卡片并存与级联；**两张卡同时流式互不阻塞**；拖到左槽停靠成 capsule、点击复原、向右弹出、dock 内重排；overlay 下收成计数 tab；专注模式整体消失；**扣住响应证明停放期间流不中断**；右栏列出本文档全部会话、可切换、可导出；**V13 已裁定并落实** | 无标注 |
| **M8** | 标注 + 索引 invariant | 划词加批注并持久；`document → sessions` 索引的 invariant 通过（含一次人为漂移后的重建）；右栏「清空」可用（旧会话移出索引、卡片挂上新会话、旧对话不可再达），导出按 V14 余项裁定的语义落实 | — |

> **V9 裁定「一张卡一条会话」之后，V2 随之落定为选项乙**（§4.4）：一个文档的会话数等于卡片数，不再有「常驻会话」这个概念，也没有 fork 入口。乙原本的弱点（列表淹没）由 §9.3 的三条缓解承担，其中第三条记为 **V13，阻塞 M7**。
>
> **右栏「清空」的语义已定**（§8）；**保存 / 导出的余项记为 V14**，阻塞 M8 的对应验收，不阻塞 M6–M7 的右栏骨架。

**判据 3（模型可用）在 M6 之后由 `dsh-tool-documents` 兑现，不占里程碑**——它是接缝设计的验证：加它不应触碰前面任何一个包。

---

## 13. 剩余未决

| # | 问题 | 状态 | 结论 / 怎么定 |
| --- | --- | --- | --- |
| **V1** | `reading/attach` 是否 `ignorable: true` | **已查实，问题作废** | **仓外插件根本不能新增持久会话事件类型**（§2.6）：`Session.append` 无 `ignorable` 入口，仓外类型不在 `KNOWN_SESSION_EVENT_TYPES` 里「by construction」，重载必被拒。→ 改为搭载 `user/message.source`（§4.2），已核实 `source.kind` 不受封闭校验 |
| **V2** | 阅读会话的创建时机与生命周期 | **已随 V9 落定（2026-09-07）** | 选项**乙**：一张卡一条会话，会话数等于卡片数。§4.4 记录的三个选项与边界事实仍然成立，只是乙的弱点（列表淹没）改由 §9.3 的三条缓解承担，而非靠选甲/丙回避。「常驻会话」与 fork 入口不再存在 |
| **V3** | 独立仓库如何消费 `dsh.client.platform: web` 客户端插件构建链 | **已查实** | `package.json` 声明 `dsh.client.platform: 'web'` + 导出 `./client` bundle，Host 侧**扫描已启用的 Loader 条目并在 `/plugins` 下供给，无需逐插件接线**。约束：① 启动前 `lib/client.js` 必须已构建，缺失则**响亮失败**；② 共享模块基线 `PLATFORM_MODULES` 只有 `react` / `react-dom` / `@deepseek-ai/cordis` / `client-store` / `client-ui-slots` / **`client-ui-primitives`**——`micromark`/`mdast-util-*`/`mermaid` 不在其中，须打进自己的 bundle（tsdown 默认行为）；③ `ui-primitives` 在基线内 ⇒ `CodeBlock` 复用零成本 |
| **V4** | 图片字节如何暴露给浏览器且带鉴权 | **已查实** | **不建 HTTP 路由**，经 Typert `@Remote` 走已鉴权的 `/api`，见 §7.4。附带发现：授权必须照抄 `referencedImage` 的引用校验，否则构成任意文件读 |
| **V5** | `@Remote` / Typert 是否为客户端调用 Host 的正路 | **已查实** | **是**。[adding-a-remote-api.md](../../../docs/cookbook/adding-a-remote-api.md) 五步法：Host 服务 `extends TypertRemoteService`（服务键与线命名空间绑定），方法标 `@Remote`；签名不合线约定时写 `remoteExport*` 适配器；`Agent`/`Session` 这类查找对象只能占顶层参数位；支持取消的方法以 `signal: AbortSignal` 收尾 |
| **V9** | 助读卡片是调用还是会话 | **已裁定（2026-09-07）** | **一张卡就是一条会话**（`ctx.agents.create`，卡片专属最小组合）。得到追问连续、刷新可重建、工具可用、Agent 与记忆系统可见，并且**「升级」机制整个不再产生**；代价三条见 **§9.3**。memo D2「助读进会话日志 ⇒ 记忆系统可直接读日志」的理由成立 |
| **V12** | 卡片会话的首字延迟与跨卡前缀缓存 | 未测 | `ctx.agents.create()` + 一次 turn 对比裸补全的 TTFT；以及卡片 agent 组合相同时，供应商侧前缀缓存能否跨会话命中。已知量：`text-turn` profile 系统提示词 4,712 B + 工具 schema 36,003 B ≈ 40.7 KB，`agent-instructions` 达 82.6 KB（录制快照实测）。不阻塞里程碑，影响是否需要为开场动作做优化 |
| **V13** | 是否摘掉 `ui-workspace` | **已查实可行，待裁定** | 摘掉即消除全局列表污染。**代价**：`uiWorkspace` 是 `ui-sidebar` / `ui-directory-picker-native` / `ui-directory-picker-browse` / `ui-agent-preset` 四者的硬注入依赖 ⇒ 连带停掉侧栏外壳与两个目录选择器，§7.1 的复用随之作废，须自建目录选择器。其余引用是 `import type {}`，纯编译期。**阻塞 M7** |
| **V14** | 右栏「清空 / 保存 / 导出」的语义 | **清空已裁定（2026-09-07），保存/导出待裁定** | **清空 = 该卡改挂新会话 + 旧会话移出 `document → sessions` 索引**。已查实 dsh **没有删除会话的 API**（`dispose()` 只 detach、无 archived 态、JSONL 规则禁止删除已提交 generation），所以旧记录留在磁盘成为不可达记录，控件因此不叫「删除」。余项：是否保留独立的「保存」控件还是只留「导出」。见 §8。**余项阻塞 M8 的对应验收** |
| V10 | 卡片重建的保真度 | 未查 | 刷新后按 `document → sessions` 索引重建卡片，轮次来自会话日志、位置来自 `reading-store`，但视口可能已变。是照搬旧坐标再钳制，还是按当前视口重新级联？影响 M6 的重建验收 |
| V6 | 大文档（>1MB）的分窗策略与 `streamText` 的配合 | 未查 | 实测 |
| **V7** | 阅读会话在原生 dsh 中的降级观感 | **已查实** | **优雅降级**：未知 `source.kind` 走 `contextProvenance` 的文档化默认 `{ role: 'inject', label: kind }`，显示为标着 `reading-selection` 的注入上下文。**附带发现**：非 `'user'` 来源一律归为 `context` 节点 ⇒ 助读须拆成两条消息，见 §7.2 |
| **V8** | 自注册的 `reading-selection` 节点定义与 `ui-chat` 的 `messageDefinition` 同时 `match` 时的优先级 | 未查 | 读 `ConversationDefinitionRegistry` 的匹配顺序；影响 M7 的卡片渲染 |

---

## 延伸阅读

- [`memos/2026-09-06-reader-migration-context.md`](../../memos/2026-09-06-reader-migration-context.md) — 上下文、规模测算、ow 侧 ADR 清单
- [`docs/subsystems/session-projection.md`](../../../docs/subsystems/session-projection.md) — 投影单元契约（**权威**）
- [`docs/subsystems/session-query.md`](../../../docs/subsystems/session-query.md) — 过滤器与检索（**权威**）
- [`docs/subsystems/slots.md`](../../../docs/subsystems/slots.md) — 插槽层级与扩展规则（**权威**）
- [`docs/capability-seams.md`](../../../docs/capability-seams.md) — 全仓接缝图谱（生成产物）
