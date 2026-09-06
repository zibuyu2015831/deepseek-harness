---
title: 阅读能力架构设计
summary: 把「Web 阅读」作为 dsh 的一等能力落地的架构设计：文档访问接缝 ctx.documents 的三角角色、document ↔ session 双向关系的两套机制（会话投影 + 可重建索引）、锚点与阅读状态的归属、UI 在 shell.overlay 的落位、六个包的划分与里程碑。
keywords: reading | documents | 阅读器 | capability-seam | session-projection | shell.overlay | anchor | message-source | 架构设计
scope: 本 fork 派生产品中「阅读」能力的架构设计与实施计划
related_files: docs/subsystems/session-projection.md | docs/subsystems/session-query.md | docs/subsystems/persistence.md | docs/subsystems/storage.md | docs/subsystems/slots.md | docs/subsystems/filesystem.md | packages/core/session/src/known-event-types.ts | packages/session/session-persistence/src/storage-contract.ts | packages/client/ui-layout/src/client/index.ts | packages/client/ui-primitives/src/markdown/render.tsx | packages/client/web/src/platform.ts | packages/api/session-controller/src/commands.ts
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
| v1 范围 | 只读、只支持 md/txt、只搬 `LocalReader`、放弃 URL 路由 | §10 |

**未定，阻塞实施**：

| | 问题 | 阻塞 |
| --- | --- | --- |
| **V2** | 阅读会话的生命周期：甲每文档一条 / 乙每次新建 / 丙常驻+fork | **M6**。分析已完成见 §4.4，倾向丙，**待产品裁定** |

其余未决（V6 大文档分窗、V8 节点定义优先级）见 §12，均不阻塞 M1。

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

- 每个附着文档的会话都在文档的会话列表里，**无论它是划词助读还是深度长谈**。
- 「多个 session 关于同一个文档」不是被实现出来的，是**从机制里掉出来的**。
- 阅读状态（位置/书签/标注）不进会话日志，**生命周期彻底独立于 session**——判据 1 成立。
- 会话在原生 dsh 里**可以正常打开**（§4.2 已核实）：它只是一批普通 `user/message`，`source.kind` 不被识别时退化为一个未知来源标签，对话本身完整可读、可续。

> ✅ **原 V1 已查实并作废**：曾计划的 `reading/attach` 独立事件在 §2.6 的约束下**根本不可行**——仓外事件类型重载必被拒，且 `Session.append` 没有 `ignorable` 入口。现方案不新增任何事件类型。

### 4.4 阅读会话的生命周期（**V2，未裁定**）

> **状态：待产品裁定。** §4.2 定义了「会话属于哪个文档」，本节回答**一个文档该有几个会话、它们何时被创建**。下面记录已查实的边界事实、三个选项的完整比较、当前倾向与其代价。**尚未决定，M6 之前必须定。**

#### 框定选项空间的三条事实

1. **会话藏不掉**（§2.7）。没有第三方可用的隐藏机制，每个被创建的会话都进列表。
2. **fork 是一等能力，且可反查**。`ctx.agents.create({ seed, inheritedEventCount, meta: { isSeeded: true, parentSession } })` 创建分叉；`SessionResultFilter` 含 `{ kind: 'parent'; values }`（§2.1），所以**父子关系不需要我们自己维护索引**。
3. **一个会话只属于一个文档**（§4.2 的投影取首条阅读来源消息）。

#### 三个选项

| | 甲：每文档一条常驻 | 乙：每次划词新建 | 丙：一条常驻 + 显式 fork 深度会话 |
| --- | --- | --- | --- |
| 会话数 | = 文档数 | **爆炸**（读一本书可达 50+） | 1 + N，N 由用户显式创建 |
| 列表可读性 | 好 | **被淹没**，且无法隐藏（事实 1） | 好，且有父子层级 |
| 上下文连续性 | 连续：第 20 次提问能引用第 3 次的结论 | 每次冷启动，无上下文 | 常驻连续；fork 继承到分叉点 |
| KV cache | 前缀稳定，命中率高 | **每次重建，成本最高** | 同甲 |
| 压缩影响 | 会话会长，早期划词迟早被压缩掉 | 不涉及 | 常驻会长（可接受）；深度会话短，不触发 |
| 父子关系 | — | — | **`parentSession` 白送，可 `filterSessions` 直查** |

#### 当前倾向：丙

三条具体理由，不是折中：

1. **化解甲的压缩焦虑**。常驻会话定位为「划词流水」，早期内容被压缩可以接受；真正值得留存的思考在 fork 出的深度会话里，那些会话短、不触发压缩。
2. **把会话数交还给用户**。每条深度会话都是显式「展开为独立会话」的产物，所以列表里每一条都有意义；乙的 50 条里有 49 条是噪音。
3. **父子关系走 dsh 的一等机制**。我们的索引只回答「哪些会话附着这个文档」，「这条从哪儿分出来」由 `parentSession` 承担。**少维护一份数据就少一份会腐烂的数据**——与 §4.1 否决 `documentRegistry` 是同一条理由。

#### 丙的代价（明写）

- **fork 是有约束的操作**，不是随手调用：需要 `inheritedEventCount` 精确切分、`meta.isSeeded: true`，且 `seed` 必须**恰好等于**继承前缀——构造器会在切点追加子会话自有的 end-seed 标记（[persistence.md](../../../docs/subsystems/persistence.md#createsessionoptions--seeding-and-metadata)）。落在 M8，前面的里程碑不碰。
- **多了一个用户需要理解的概念**（常驻 vs 深度）。若实测发现用户从不用「展开」，丙就退化成甲，那时删掉 fork 入口即可——**这个方向的退化是无损的**，反过来（从乙收敛回丙）则需要迁移已产生的大量会话。这是选丙而非选乙的一个额外安全边际。

#### 延伸问题（不阻塞 M1–M8）

常驻阅读会话的**压缩策略**。dsh 有 compaction 接缝，阅读流水可能需要比对话更激进的压缩，或者「读完一本归档并新建」。等实际用起来产生真实的长会话之后再定，不预设。

`ctx.documents` 的类型面。**这是本设计里唯一需要一次想对的东西**——它同时是阅读界面、存储层和未来模型工具的共同语言。

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

**取舍（明写）**：`workspace` 选 uuid 而非路径，理由是「路径规范化会重写路径，而引用锚点必须稳定」。文档这里**反向选择**——用路径派生标识，换掉整个注册表。代价是**重命名/移动会断开关系**。v1 接受，记入已知限制；修复路径见 §10。

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
| `dsh-reading-store` | 持久数据形态 | storage domain：位置 / 书签 / 标注 / 最近打开 + `document → sessions` 索引。**发布 `./invariant`**（§4.2）。 |
| `dsh-reading-session` | **Consumer**（会话侧） | 拥有 `reading-selection` 的 `MessageSourceMap` 扩展、`reading` 投影单元、系统提示词段、面向客户端的 `@Remote` 方法（含 §7.4 的资源读取）。**不新增会话事件类型**（§2.6）。 |
| `dsh-client-ui-reading` | **Consumer**（界面） | `shell.overlay` 占位者：书库 / 文件树 / 阅读视图 / 大纲 / chrome / 键位 / 自有 markdown 渲染臂。 |
| `dsh-client-ui-reading-assist` | **Consumer**（界面） | 选区模型、动作菜单、助读窗口与 dock。 |

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
  → @Remote assist({ document, anchor, excerpt, action, question? })
reading-session：
  ① 取或建「该文档的阅读会话」（ctx.agents；生命周期见 V2）
  ② append 两条 user/message，顺序照 dsh 自己的 [...claimed, context] 惯例：
     a. 用户的提问或动作文本  → source.kind: 'user'
     b. 选中的原文 + 定位     → source.kind: 'reading-selection'   ← 投影来源
  ③ 正常 turn，模型可用全套工具
  ④ 流式结果经既有 session/event 推送回界面
```

**为什么拆成两条**（V7 查证的结果）：`ui-chat` 的 `messageDefinition` 在 [`message.ts:55`](../../../packages/client/ui-chat/src/client/conversation-nodes/message.ts) 判 `source.kind !== 'user'` 就归为 **`context` 节点**（"Non-user context injected into model history"）。若把提问也塞进 `reading-selection` 一条里，它在任何 dsh 客户端里都会被渲染成"注入的上下文"而不是用户发言——**语义错了**。拆开之后：提问在哪都是正常的用户气泡，原文在哪都是注入上下文，与 dsh 对这两件事的既有建模完全一致。

**降级观感已核实**：未知 `source.kind` 落到 [`event-projection.ts:73-76`](../../../packages/client/ui-chat/src/client/conversation-nodes/event-projection.ts) 的文档化默认——

```ts
default:
  // MessageSourceMap is merge-extensible; keep an unknown producer
  // visible by its durable kind.
  return { role: 'inject', label: kind }
```

⇒ 原生 dsh 里显示为一条标着 `reading-selection` 的注入上下文，**可见、可读、不报错**。我们自己的产品用 `ctx.uiConversation.events.register()` 注册专属节点定义，把这一对渲染成一张阅读卡片。

**「模型可见 ⟺ 已记录」检查**：进入模型请求的是 excerpt（在消息内容里）与文档标题（在系统提示词段里）。两者都在同一条 `user/message` 里落盘——excerpt 在 `content`，标题在 `source.title`。✓

**KV cache 检查**：文档正文**绝不进系统提示词**。系统段只含稳定的「用户正在阅读《标题》」——一个会话绑一个文档，该段在会话生命周期内不变 ⇒ 前缀稳定。选区以追加消息进入 ⇒ 天然缓存友好。（依据：实测该 Web 会话缓存命中 89%，注入策略必须保住它。）

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

- **两级助读**。轻量层：阅读器内的助读卡片/dock，短问答，自有精简 transcript（ow 的 `AssistWindow` 证明这是不同的交互，不是对话的缩水版）。深度层：「转到完整会话」——关闭阅读器接管，落到常规 `conversation` 外壳。两者**是同一批会话**（§4.3），只是呈现不同。
- 侧栏在接管期不可见 ⇒ 阅读器需自带返回入口。

**放弃的备选**：fork `ui-layout` 增第四栏——产品形态更好，但持续承担上游合并成本。**推迟到接管形态被实际使用否决之后再考虑**，不预支。

**路由**：dsh 客户端**没有任何路由**（全量 grep `pushState`/`popstate`/`location.*` 在产品代码零命中）。阅读器开合与当前文档是客户端 store 状态。ow 的 ADR-038（URL 状态）v1 放弃——阅读位置本就durable 在 `reading-store`，「重开回到上次位置」不依赖 URL。

---

## 9. 与仓库规则的逐条自查

| 规则（[AGENTS.md](../../../AGENTS.md)） | 本设计 |
| --- | --- |
| 能力接缝是完整三角，不是单一角色 | `documents`（Definition）+ `documents-local`（Provider）+ `reading-session`/`client-ui-reading`（Consumer）v1 同时存在 ✓ |
| 模型可见 ⟺ 已记录 | §7.2 逐项对照：提问在 `source.kind: 'user'` 的消息里，excerpt 与文档标题在 `source.kind: 'reading-selection'` 的消息里，两条都是 `user/message`，全部落盘 ✓ |
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

## 10. 已知限制与取舍

**这些是设计的边界，不是待办清单。**

- **路径派生标识 ⇒ 重命名断链**（§5.1）。修复路径（v2）：`reading-store` 保存附着时的内容指纹，检测到「旧 id 消失 + 新 id 出现且指纹相同」时提供合并操作。**不做自动合并**——静默改写关系比断链更糟。
- **`document → sessions` 是缓存，不是真相**。它可能落后、可能漂移；invariant 检测，重建修复。这是刻意的：真相在会话日志里（§4.2）。
- **阅读器接管全屏**（§8）。侧栏与对话在阅读期不可见。
- **自有 markdown 渲染臂意味着自担安全策略**。`MarkdownText` 的白名单（协议白名单、原始 HTML 字面化、KaTeX 无信任命令）必须**逐条复刻**，本地图片是唯一有意放开的口子，且只经 §7.4 的 `@Remote asset()`——浏览器永不直接拿到本地路径，且服务端必须做引用校验。
- **不能新增持久会话事件类型**（§2.6）。这不是本设计的选择，是仓外插件的结构性约束。任何「给会话加一个新的durable事实」的需求，都只能搭载在已知事件的载荷里，或接受该会话在原生 dsh 中无法重载。
- **锚点在文档被大改时会 `lost`**。孤儿标注保留、可见、不自动删除。
- **v1 只读**。改文件让用户对 Agent 说，`edit`/`write` 工具与权限层本就在。
- **只支持 md/txt**。epub/pdf 是新提供方，不是新接缝。

---

## 11. 里程碑

每个里程碑一个 PR，浏览器内人工验收后再合。**验收条件必须能被一个人在界面上点出来。**

| # | 里程碑 | 验收条件 | 不做什么 |
| --- | --- | --- | --- |
| **M1** | 接缝与本地提供方 | Host 单测：`resolve` 一个 md 文件得到正确 title/mediaType/version；`list` 列出目录；`read` 返回全文 | 无界面、无大纲、无锚点 |
| **M2** | 阅读视图落地 `shell.overlay` | 浏览器里打开阅读器 → 选目录 → 点文件 → 看见渲染后的正文；关闭回到对话 | 无大纲、无位置、无助读 |
| **M3** | 大纲与渲染补全 | 点大纲条目跳到对应位置；本地图片显示；代码高亮；mermaid 渲染 | 无位置持久化 |
| **M4** | 阅读状态 | 关掉重开回到上次位置；书签可加可跳；最近目录可用 | 无标注、无助读 |
| **M5** | 锚点 | 外部改文档后重开，书签仍在正确位置或被标记为孤儿 | 无标注 UI |
| **M6** | 助读 turn | 划词 → 提问 → 看见流式回答；带 `reading-selection` 来源的消息落日志；投影可见；**该会话在原生 dsh 里仍能打开** | 无 dock、无多窗口 |
| **M7** | 助读界面 | 多助读窗口、dock、专注模式 | 无标注 |
| **M8** | 标注 + 文档会话列表 | 划词加批注并持久；文档侧栏列出该文档的全部会话；invariant 通过；**若 V2 选丙**：「展开为独立会话」可用且 `parentSession` 层级正确 | — |

> **M6 与 M8 的形态取决于 V2**（§4.4，未裁定）。M6 需要知道「取或建阅读会话」的规则；M8 的 fork 入口只在选丙时存在。**V2 是 M6 的前置**，其余里程碑不受影响。

**判据 3（模型可用）在 M6 之后由 `dsh-tool-documents` 兑现，不占里程碑**——它是接缝设计的验证：加它不应触碰前面任何一个包。

---

## 12. 剩余未决

| # | 问题 | 状态 | 结论 / 怎么定 |
| --- | --- | --- | --- |
| **V1** | `reading/attach` 是否 `ignorable: true` | **已查实，问题作废** | **仓外插件根本不能新增持久会话事件类型**（§2.6）：`Session.append` 无 `ignorable` 入口，仓外类型不在 `KNOWN_SESSION_EVENT_TYPES` 里「by construction」，重载必被拒。→ 改为搭载 `user/message.source`（§4.2），已核实 `source.kind` 不受封闭校验 |
| **V2** | 阅读会话的创建时机与生命周期 | **未裁定，分析已完成** | 三条边界事实、三个选项的完整比较、倾向（丙：一条常驻 + 显式 fork）与其代价，全部记录在 **§4.4**。**需产品裁定，M6 之前必须定** |
| **V3** | 独立仓库如何消费 `dsh.client.platform: web` 客户端插件构建链 | **已查实** | `package.json` 声明 `dsh.client.platform: 'web'` + 导出 `./client` bundle，Host 侧**扫描已启用的 Loader 条目并在 `/plugins` 下供给，无需逐插件接线**。约束：① 启动前 `lib/client.js` 必须已构建，缺失则**响亮失败**；② 共享模块基线 `PLATFORM_MODULES` 只有 `react` / `react-dom` / `@deepseek-ai/cordis` / `client-store` / `client-ui-slots` / **`client-ui-primitives`**——`micromark`/`mdast-util-*`/`mermaid` 不在其中，须打进自己的 bundle（tsdown 默认行为）；③ `ui-primitives` 在基线内 ⇒ `CodeBlock` 复用零成本 |
| **V4** | 图片字节如何暴露给浏览器且带鉴权 | **已查实** | **不建 HTTP 路由**，经 Typert `@Remote` 走已鉴权的 `/api`，见 §7.4。附带发现：授权必须照抄 `referencedImage` 的引用校验，否则构成任意文件读 |
| **V5** | `@Remote` / Typert 是否为客户端调用 Host 的正路 | **已查实** | **是**。[adding-a-remote-api.md](../../../docs/cookbook/adding-a-remote-api.md) 五步法：Host 服务 `extends TypertRemoteService`（服务键与线命名空间绑定），方法标 `@Remote`；签名不合线约定时写 `remoteExport*` 适配器；`Agent`/`Session` 这类查找对象只能占顶层参数位；支持取消的方法以 `signal: AbortSignal` 收尾 |
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
