---
title: DeepSeek Harness 开发文档体系生成方案
summary: 记录 deepseek-harness 仓库首次生成 dev_docs 文档体系的项目检测结果、策略决策、子文档清单、批次计划、证据链与准入条件；本方案停在 Phase 1 人工审核门，用户确认后方可进入正式文档生成。
keywords: generation-plan | deepseek-harness | monorepo | cordis | agent-harness | dev-docs
scope: deepseek-harness 仓库 dev_docs 文档体系首次生成方案（packages/ apps/ vendor/ docs/）
related_files: package.json | pnpm-workspace.yaml | AGENTS.md | docs/architecture.md | docs/AGENTS.md | docs/testing.md | docs/development.md | packages/README.md | scripts/translation-pairing.ts | scripts/verify-md-links.ts | scripts/verify-md-wrap.ts | scripts/verify-doc-budgets.ts | scripts/run-gates.ts | packages/llm/README.md | packages/session/session-telemetry-otel/README.md | packages/bundle/base/README.md | apps/cli/src/bin.ts | vitest.config.ts
dependencies: dev_docs/_analysis/project_analysis_report.md | dev_docs/_analysis/generation_progress.md
verified_at: 2026-08-18
---

# DeepSeek Harness - AI 文档生成方案

## 📋 方案元信息

- **项目名称**: DeepSeek Harness (`dsh`) / `@deepseek-ai/dsh-root`
- **项目类型**: AI/LLM 应用（Agent Harness 运行时）× Monorepo；次级类型 CLI 工具 + 库/SDK + Web 全栈
- **主要技术栈**: TypeScript 6 (ESM) + Cordis 插件框架（vendored）+ pnpm 11 workspaces + Vitest 4 + tsdown/tsc 双面构建 + React/Vite Web 前端 + Python SDK
- **方案创建日期**: 2026-08-18
- **预计执行耗时**: 18-26 小时（分 7 批）
- **当前阶段**: Phase 1 方案复查完成，等待人工审核（**未获正式生成授权**）

---

## 🎯 任务复杂度评估 (Complexity Assessment)

### 复杂度评级

**综合评级**: ⭐⭐⭐⭐⭐ (5 星 / 超大型)

### 原因分析

1. **代码规模**

   - 文件数量: 7,238 个（已排除框架、`node_modules/`、`.git/` 等标准排除目录）
   - 代码行数: TypeScript 501,274 行 + TSX 66,872 行 + Python 4,373 行；Markdown 170,752 行
   - 影响: **高**

2. **架构复杂度**

   - 架构特点: Monorepo（219 个 workspace 包）+ 插件化运行时（"一切皆插件"）+ 能力接缝（Service Definition / Provider / Consumer 三角）+ 双编译面（host / client）+ 多进程协议（JSON-RPC / ACP）
   - 模块数量: `packages/` 下 49 个包组，219 个 workspace 包，9 个 vendored 包，2 个 app
   - 影响: **高**

3. **依赖复杂度**
   - 核心依赖: Cordis（vendored 而非 npm 安装）、`@earendil-works/pi-ai`、`node-pty`（打补丁）、`koffi`、OpenTelemetry SDK
   - 特殊依赖: `vendor/` 为固定源码副本并 rescope 为 `@deepseek-ai/*`；`native/landlock-run` 为原生插件
   - 影响: **高**

### 复杂度因子计算

```
基础级别 = 超大型 (3 级)  [7,238 文件 > 500]
+ Monorepo (+1)
+ 混合语言 (+0.5)  [TypeScript / TSX / Python / YAML / 原生]
= 4.5 级 → 已封顶于超大型项目策略
```

### 预计工作量

- **总计**: 18-26 小时
- **阶段一 (项目分析)**: 已完成（约 1.5 小时）
- **阶段二 (核心代码模式提取)**: 2-3 小时
- **阶段三 (主文档生成)**: 2-3 小时
- **阶段四 (子文档生成, 17 篇)**: 12-16 小时
- **阶段五 (质量验收)**: 1.5-2 小时

### 风险点

- [x] **大文件**: 存在超过 800 行的源文件，需先取大纲再分段读取
- [x] **复杂依赖**: 219 包依赖图需依赖生成产物 `docs/module-graph.md` 而非手工推断
- [ ] **文档不足**: 不成立——仓库文档极其充分（1,390 篇 Agent Notes + 分层 `docs/`）
- [x] **特殊架构**: Cordis 时空可组合性范式、能力接缝、双编译面属于非常规架构
- [x] **其他**: 仓库自带被 CI 强制的文档治理体系，`dev_docs/` 存在事实归属地重复风险（详见 [问题报告](./project_analysis_report.md) 🔴 问题 1）

---

## 🤝 交互策略 (Interaction Strategy)

### 1. 阶段性确认点

- [x] ✅ **完成阶段一后**: 已请求用户审查（本文档即审查对象）
- [ ] ✅ **完成批次 1 后**: 请求用户审查主文档与架构文档质量再继续
- [ ] ✅ **批次 6 开始前**: 必须先解决 `dev_docs/**/README.md` 与 `verify-translation-pairing` 的作用域冲突

### 2. 大文件处理策略

- 超过 800 行的文件先取结构大纲，再按需分段读取
- 依赖关系、事件表、工具表、配置表一律引用仓库**生成产物**（`docs/module-graph.md`、`docs/tool-catalog.md`、`docs/config-catalog.md`、`docs/persistence-catalog.md`、`docs/event-producer-consumer.md`），不手工重述——手工重述会立刻与生成器漂移

### 3. 不确定信息处理

**必须询问用户的情况**: 见本文档"需要人工确认的项目特性"。

**不应该做的**:

- ❌ 臆测架构或业务语义
- ❌ 编造代码示例
- ❌ 使用占位符代替实际数据
- ❌ 将 `AI-Coding-Context/` 框架自身内容纳入项目分析结果

### 4. 复杂模块处理

- 先用 mermaid 绘制包组依赖与回合流程图
- 每批产出后提交用户审查
- 架构断言一律标注 `docs/` 或源码出处

---

## 📐 文档生成原则 (Documentation Principles)

### 1. 代码示例要求

所有示例必须来自真实代码并注明 `文件路径:行号`，格式：

```markdown
### 示例标题

**文件位置**: `packages/core/agent-loop/src/loop.ts:120-148`

​```typescript
// 实际代码
​```
```

### 2. 链接格式规范

- 跨 `dev_docs` 文档引用使用相对路径 `./other_doc.md`
- 引用仓库文档使用相对路径 `../docs/architecture.md`
- 外部资源使用完整 URL

### 3. 可视化要求

必须使用 mermaid 的场景：整体架构图、回合（turn/step）流程图、包组依赖图、能力接缝三角图、会话事件流。

### 4. Markdown 格式要求

- 代码块标注语言类型
- 结构化信息使用表格
- 使用 GitHub Flavored Markdown

> **注意**: `dev_docs/` **不适用**仓库 `docs/AGENTS.md` 的"一段一物理行"（`verify-md-wrap`）规则——该门禁的 `PATTERNS` 不包含 `dev_docs/`（证据见"关键事实记录" F12）。但为降低未来纳入门禁的成本，本方案仍要求段落不做硬换行。

---

## 🎯 第一阶段：项目基础分析（已完成）

### 1.1 项目类型识别

**分析结果**:

```
项目类型: AI/LLM 应用（Agent Harness 运行时）× Monorepo
          次级类型: CLI 工具 + 库/SDK + Web 全栈
主框架:   Cordis（vendored 源码副本，非 npm 依赖）
构建工具: tsc -b（类型与 lib）+ tsdown（运行时打包）；Vite 用于 Web 前端
包管理:   pnpm@11.7.0，Node ^22.19.0 || >=24.0.0
其他核心库: TypeScript ^6.0.3、Vitest ^4.1.8、oxlint 1.76.0、@earendil-works/pi-ai、node-pty（打补丁）
```

**与 AICC 标准类型配置的偏离说明**（必须保留）:

`core/project_types/ai_llm_app.md` 推荐的 `rag_architecture.md` 与 `vector_database.md` **不适用于本项目**。证据：对 `packages/**/package.json` 执行 `pinecone|weaviate|chroma|qdrant|milvus|pgvector|embedding` 检索，命中数为 0（E4）。本项目是 Agent 运行时而非 RAG 应用，因此这两篇被替换为 `capability_seams.md` 与 `session_and_events.md`。

**验证方式**:

- [x] 检查 `package.json` 的 devDependencies 与 engines
- [x] 检查 `pnpm-workspace.yaml` 的 workspace 成员与 overrides
- [x] 检查 `docs/architecture.md`、`packages/README.md`
- [x] 扫描 `packages/`、`apps/`、`vendor/`、`python/`

---

### 1.2 项目规模统计

**统计结果**:

```
总文件数: 7,238 个；总目录数: 1,211 个；最大深度: 9
工作区包:
├─ packages/<group>/<pkg>: 219 个
├─ vendor/*:                9 个
└─ apps/*:                  2 个 (cli, web)

代码行数:
├─ TypeScript (.ts):  501,274 行
├─ TSX (.tsx):         66,872 行
├─ Markdown (.md):    170,752 行
├─ JSON (.json):       42,657 行
├─ YAML (.yml):         8,319 行
└─ Python (.py):        4,373 行

测试资产:
├─ 测试目录: 224 个（`tests/` `test/` `__tests__/` `spec/` 等标准目录）
├─ 测试目录下文件总数: 1739 个（含快照期望输出与夹具）
└─ packages/ 下 *.spec.ts / *.test.ts 源文件: 643 个

决策记录:
└─ .agents/notes/**/*.md: 1,390 篇
```

**验证命令**:

```bash
python3 AI-Coding-Context/tools/py/project_scanner.py . --mode summary --exclude-standard
find packages -maxdepth 3 -name package.json -not -path "*/node_modules/*" | wc -l
find vendor -maxdepth 2 -name package.json -not -path "*/node_modules/*" | wc -l
find . -path ./node_modules -prune -o -path ./AI-Coding-Context -prune -o -path ./.git -prune -o -name "*.ts" -print | grep -v node_modules | xargs wc -l | tail -1
find packages -name "*.spec.ts" -o -name "*.test.ts" | grep -v node_modules | wc -l   # 643
find .agents/notes -name "*.md" | wc -l
```

**⚠️ 人工验证点**:

- 文件数 7,238 已排除框架目录 `AI-Coding-Context/`（符号链接）、`node_modules/`、`.git/`
- `.ts` 行数含 `vendor/` 与 `scripts/`；正式文档中引用该数字时必须同时说明统计范围

---

### 1.3 目录结构分析

**核心目录清单**（1-2 级）:

```
deepseek-harness/
├── packages/           - 219 个 workspace 包，按 49 个能力组划分（详见 packages/README.md）
├── apps/               - 产品装配层：cli（`dsh` 命令）、web（Web 应用）
├── vendor/             - Cordis 生态固定源码副本（9 包），rescope 为 @deepseek-ai/*
├── native/             - landlock-run 原生启动器（Linux 进程约束）
├── python/             - Python SDK 与打包运行时（sdk、sdk-runtime）
├── docs/               - 仓库自有分层文档（双语），含生成产物目录
├── examples/           - 可运行 cordis.yml 叶子示例
├── scripts/            - 仓库门禁与生成器（verify-* / gen-* / run-gates）
├── website/            - VitePress 文档站点
├── .agents/            - Agent Notes（1,390 篇）与 Skills
└── dev_docs/           - 【本次新增】AICC 文档体系
```

**验证方式**:

- [x] `ls -1`、`ls -1 packages/`、`ls -1 docs/`、`ls -1 vendor/`
- [x] 目录用途来自 `AGENTS.md` "Repository layout" 章节与 `packages/README.md` 层级表

---

### 1.3B 项目定位与不可破坏约束

| constraint | evidence | documentation_impact | ai_rules_impact | status |
| --- | --- | --- | --- | --- |
| 开发者预览期，明确声明会有破坏性变更 | `README.md`（Developer preview 章节） | 主文档、`deployment_guide.md` 必须标注稳定性预期 | 不得承诺 API 向后兼容 | confirmed |
| 预发布阶段"foundation over blast radius"：无外部消费者，禁止兼容性垫片 | `AGENTS.md`（Pre-release stance 章节） | `architecture_overview.md`、`AI_RULES.md` | 重命名/重构时必须同步更新全部引用，后端拒绝旧磁盘格式 | confirmed |
| "一切皆插件"，无特权内核 | `docs/architecture.md:11-13` | `architecture_overview.md`、`plugin_development_guide.md` | 新行为必须挂在文档化扩展点，改 `agent-loop` 需同步更新 `docs/architecture.md` | confirmed |
| Model-visible ⟺ logged：任何进入模型请求的内容必须可从会话日志重建 | `docs/architecture.md:96`、`AGENTS.md` Conventions | `session_and_events.md`、`agent_loop_and_tools.md` | 新增模型可见输入必须同时新增会话事件 | confirmed |
| 注册即效果：所有贡献通过 `ctx.effect()` / `ctx.on()`，registry 的 `register()` 返回 disposer | `AGENTS.md` Conventions | `plugin_development_guide.md`、`capability_seams.md` | 禁止不可撤销的全局注册 | confirmed |
| 能力接缝是三角整体（Definition/Provider/Consumer），不得只做一个角色 | `docs/architecture.md:100`、`docs/glossary.md` | `capability_seams.md` | 新增能力必须设计三个角色 | confirmed |
| 全仓 ESM（`"type": "module"`），CLI 源码启动走 tsx ESM-only 钩子 | `package.json`、`AGENTS.md` Conventions | `monorepo_and_build.md` | 禁止引入 CJS-only 导出 | confirmed |
| 文档"一个事实一个归属地"分层治理，由 CI 门禁强制 | `docs/AGENTS.md:17-34` | **全部 dev_docs 子文档** | `dev_docs` 必须标注上游事实源，禁止与 `docs/` 分叉 | confirmed（冲突已知，见问题报告 🔴 问题 1） |
| 非平凡改动必须在同一 PR 内附 Agent Note | `AGENTS.md` Conventions、`.agents/notes/README.md` | `quality_gates.md`、`AI_RULES.md` | AI 提交非平凡改动必须写 Agent Note | confirmed |
| 覆盖率门禁为 `test:coverage`（`packages/*/*/src` 每文件 100%），不是 `test` | `package.json` scripts、`docs/testing.md` | `testing_guide.md`、`quality_gates.md` | 不得以 `pnpm run test` 通过冒充覆盖率通过 | confirmed |
| 双语文档契约：`docs/`、`python/`、`.agents/notes/`、任意 README 需成对翻译 | `docs/AGENTS.md`、`scripts/translation-pairing.ts:180-186` | `quality_gates.md` | 改动上述范围的源文档必须同步更新配对 | confirmed |
| MIT 许可，第三方依赖须在 `THIRD_PARTY_NOTICES.md` 披露 | `LICENSE`、`README.md` | `deployment_guide.md` | 新增依赖需跑 `gen-third-party-notices` | confirmed |
| 跨平台：macOS / Linux / Windows，CI 拥有平台矩阵 | `vitest.config.ts:22-46`、`package.json` `check:ci:windows-*` | `testing_guide.md` | 平台相关代码需考虑 Windows 分支 | confirmed |

---

### 1.3C AI/外部服务边界

| boundary | evidence | data_sent_or_stored | user_authorization_or_config | documentation_impact | status |
| --- | --- | --- | --- | --- | --- |
| DeepSeek 官方模型 API（直连适配器） | `packages/llm/README.md:12`（`llm-deepseek` 注册于 `ctx.llm`）；`AGENTS.md` Secrets 章节 | 会话消息、系统提示词、工具 schema 与工具结果 | `DEEPSEEK_API_KEY`，可选 `DEEPSEEK_BASE_URL`，根 `.env` | `model_configuration.md`、`prompt_management.md`、`AI_RULES.md` | confirmed |
| pi-ai 多 Provider 适配器 | `packages/llm/README.md:13`（`llm-pi-ai`）；`pnpm-workspace.yaml` 中 `@earendil-works/pi-ai@0.82.1` | 同上，目标端点取决于所选 provider | 由 cordis.yml 配置行选择 | `model_configuration.md` | confirmed |
| 凭证引用能力（env 优先于 `.env`） | `packages/credentials/`；`AGENTS.md` Repository layout | API key 等凭证从环境/`.env` 读取 | 用户自行提供，仓库禁止提交凭证 | `security_and_sandbox.md` | confirmed |
| Web 能力（search / fetch providers） | `packages/web/`；`packages/README.md` `web/` 行 | 模型发起的检索关键词与抓取 URL 出站 | 由 profile/bundle 组合决定是否挂载 | `security_and_sandbox.md` | confirmed |
| OpenTelemetry 遥测上传 | `packages/session/session-telemetry-otel/README.md:5` | 上传模式下：会话事件账本记录 + 运行记录；资源标识含 `service.name`/`service.version` 与匿名 `user.id` | `mode` 配置决定"实时上传/仅在反馈时回放/仅本地"；匿名 id 存于 `$DSH_HOME/.anonymous-user-id`，删除文件即重置 | `security_and_sandbox.md`（必须写明三种模式与匿名 id 重置方式） | confirmed |
| E2B 远程沙箱（POC） | `packages/e2b/`；`AGENTS.md` Repository layout | 文件系统与子进程操作转发至远程沙箱 | 仅在挂载 E2B provider 时生效，标注为 POC | `security_and_sandbox.md` | confirmed |
| 本地进程约束（bwrap / Landlock / Seatbelt / Windows ACL） | `packages/sandbox/`（`sandbox-local`、`sandbox-policy`、`sandbox-windows-acl`）；`native/landlock-run` | 无外传；限制子进程可访问范围 | 由 base bundle 的 policy 行配置 | `security_and_sandbox.md` | confirmed |
| **无向量数据库 / 无 RAG / 无 embedding 流水线** | 对 `packages/**/package.json` 执行 `grep -rilE` 多关键词检索（pinecone / weaviate / chroma / qdrant / milvus / pgvector / embedding），命中 0 | 不适用 | 不适用 | 明确排除 `rag_architecture.md` 与 `vector_database.md` | confirmed |

---

### 1.4 业务模块识别

以包组为模块单位（完整表以 `packages/README.md` 为事实源，本表仅列文档规划相关的核心组）:

| 模块名称 | 目录位置 | 职责 | 关联技术 |
| -------- | -------- | ---- | -------- |
| 产品 API 主干 | `packages/core/` | 会话、系统提示词、工具注册与执行、Agent 接口与默认循环、per-agent 作用域 | `ctx.sessions` / `ctx.systemPrompt` / `ctx.tools` / `ctx.agents` / `ctx.agentLoop` |
| LLM 能力族 | `packages/llm/` | 消息与流协议、适配器接缝、DeepSeek/pi-ai 适配器、重试策略、token 计量 | `ctx.llm` / `ctx.tokenMeter` |
| 会话数据面 | `packages/session/` | 持久化接缝 + JSONL/SQLite 后端、投影接缝、日志派生标题、遥测 | `SessionEventMap` |
| 会话检索 | `packages/session-query/` | 逻辑语料、有界读取、血缘、事件关系、SQLite 全文检索 | — |
| 执行能力族 | `packages/shell/` `subprocess/` `terminal/` `code-runtime/` `fs/` `lsp/` | bash、进程树、持久 PTY、worker 代码执行、文件系统、语言服务 | `ctx.shell` / `ctx.subprocess` / `ctx.terminals` / `ctx.fs` |
| 约束与策略 | `packages/sandbox/` `guard/` `interaction/` | 进程约束后端、循环卫生与工具超时、审批与权限预设、命令 | `ctx.sandbox` / `ctx.commands` |
| 组合与分发 | `packages/bundle/` `preset/` `boot/` | profile/bundle 补丁层、按会话 Agent 组合、应用启动胶水 | `dsh.profile` / `dsh.bundle` |
| 协议与外部集成 | `packages/sdk/` `acp/` `hooks/` `api/` | JSON-RPC 协议+客户端+服务端插件、ACP 服务器、Claude Code/Codex 钩子桥、Typert RPC 网关 | — |
| Web GUI | `packages/host/` `packages/client/` `apps/web/` | API 网关与 HTTP 路由；浏览器侧 shell、wire、object services、slots、`ui-*` 插件 | 双编译面（host / client） |
| 自修改运行时 | `packages/extensions/` | Agent 检视并挂载/卸载自身插件 | — |

**识别依据**: `packages/README.md` 层级表 + `docs/architecture.md` "Core packages" 表 + 目录扫描。

---

### 1.5 架构特点识别

1. **Cordis 插件树：一切皆插件，无特权内核**

   - **识别依据**: `docs/architecture.md:11-13`；`vendor/cordis`（固定源码副本）
   - **影响范围**: 全部包
   - **实现方式**: 插件向共享 context 贡献服务、类型化事件与可撤销效果；模型适配器、工具注册表、会话日志、Agent 循环本身都是插件，均可从配置替换

2. **Profile / Bundle 分层组合**

   - **识别依据**: `docs/architecture.md:15-37`；`packages/bundle/{base,web-app,headless}`；`package.json` 的 `dsh` 字段
   - **影响范围**: 启动装配
   - **实现方式**: 空条目列表上依次施加：profile 列出的各 bundle → profile 的 `cordis.patch.yml` → home 级补丁 → `--patch` 覆盖层；补丁按 id 替换整行 config 或插入新行

3. **能力接缝（Capability Seam）三角**

   - **识别依据**: `docs/architecture.md:98-102`；`docs/capability-seams.md`
   - **影响范围**: fs / shell / subprocess / terminal / lsp / web / subagent / sandbox / compaction / workflow 等
   - **实现方式**: Service Definition（接口）+ Service Provider（实现）+ Consumer（使用者，常为模型可见工具）。换一个 provider 即改变整个产品行为——文件系统与子进程 provider 共享同一执行世界，指向远程沙箱即可整体迁移 Bash/PTY/LSP

4. **回合（turn）/ 步（step）流程与瀑布式事件**

   - **识别依据**: `docs/architecture.md:63-90`
   - **影响范围**: `agent-loop` 及全部拦截型插件
   - **实现方式**: `turn/start` → 认领输入 → 组装提示词与工具 schema → `agent/pre-step` → `step/start` → `agent/request` → `llm/stream` → `assistant/*` → `tool/call*` → `tools/pre-execute|execute|post-execute` → `tool/result*` → `step/end` → `agent/turn-stopping` → `turn/end`。瀑布监听器**必须调用 `next()`**，否则短路整条链

5. **会话日志为模型上下文的唯一来源**

   - **识别依据**: `docs/architecture.md:92-96`
   - **影响范围**: 分叉、恢复、转录、遥测、持久化
   - **实现方式**: `deriveMessages()` 从追加式日志投影模型历史；原始 `assistant/chunk` 事件保留回放与 UI 保真度；运行时不变量断言"模型可见即已记录"

6. **双编译面（host / client）**

   - **识别依据**: `tsconfig.host.json:2-4` 与 `tsconfig.client.json:2-6` 的注释
   - **影响范围**: 类型检查与构建
   - **实现方式**: host 与 client 两侧在**相同 ctx key**（`sessions`、`loader`）上合并不同服务，单个 TS program 无法同时看到两者，因此拆成两个聚合程序，共享叶子包各自被两个程序引用

7. **Monorepo + vendored 依赖 + 生成产物门禁**

   - **识别依据**: `pnpm-workspace.yaml`；`package.json` 中 30+ 个 `verify-*` 与 `gen-*` 脚本；`scripts/run-gates.ts`
   - **影响范围**: 全仓开发流程
   - **实现方式**: `vendor/` 为固定上游源码副本并 rescope；依赖图、工具目录、配置目录、持久化目录、模块图等均由生成器产出并在 CI 做新鲜度门禁

**验证方式**:

- [x] 每个特点均给出 `docs/` 或配置文件出处
- [x] 未出现无出处的架构断言

---

## ⚠️ 代码脱敏规范

本项目为 MIT 开源仓库，代码本身公开，脱敏重点收敛为：

**必须脱敏**:

- `DEEPSEEK_API_KEY`、`DEEPSEEK_BASE_URL` 的**真实值**——文档中一律写环境变量名或 `sk-***`
- 本机绝对路径（如 `/Users/<user>/...`）——统一改写为 `<repo-root>/` 或 `$DSH_HOME/`
- `.env`、`$DSH_HOME/.anonymous-user-id` 的真实内容
- 任何 OTLP endpoint 的真实地址

**可以保留**:

- 技术栈与版本、包名与 ctx key、配置项与默认值
- 仓库内真实文件路径与行号（公开仓库）
- 公开协议与端点形态（JSON-RPC、ACP、OTLP/HTTP）

**脱敏检查清单**（每篇文档生成后执行）:

- [ ] 不包含 API key / token 真实值
- [ ] 无本机用户名与绝对路径
- [ ] 无 `.env` 内容
- [ ] 保留技术实现细节，代码仍完整可读

---

## 📝 第二阶段：核心代码模式提取（待执行）

> 本阶段在**用户确认后**执行。以下为待提取清单与来源，尚未粘贴真实代码——Phase 1 禁止编造示例。

### 2.1 插件定义模式

- **来源**: `packages/core/agent-loop/src/`、`packages/todo/`（最小工具插件）、`packages/guard/`
- **待提取**: 函数插件与 `Service` 子类两种形态；`ctx.effect()` / `ctx.on()` 注册及其 disposer

### 2.2 模型可见工具注册模式

- **来源**: `packages/todo/`（`todo_write`）、`packages/fs/`、`docs/cookbook/adding-a-tool.md`
- **待提取**: 工具 schema、UI 渲染意图（`generic`/`terminal`/`diff`、`locations`）、呈现方法为 `args` 的纯函数

### 2.3 能力接缝三角模式

- **来源**: `packages/shell/`（request/spec 拆分为模板）、`packages/subprocess/`
- **待提取**: Service Definition 接口、Provider 实现、Consumer 工具三段；`resolve(request): Spec` 显式默认值步骤

### 2.4 会话事件扩展模式

- **来源**: `packages/session/`、`docs/subsystems/session.md`
- **待提取**: `SessionEventMap` 声明合并、事件 JSDoc 的 `@mode` 与 `@param`、`ignorable: true` 语义

### 2.5 LLM 适配器模式

- **来源**: `packages/llm/llm-deepseek/`、`docs/subsystems/llm-streaming.md`、`docs/cookbook/adding-an-llm-adapter.md`
- **待提取**: `ctx.llm` 注册、`StreamChunk` 协议、`llm/stream` 瀑布

### 2.6 配置与组合模式

- **来源**: `packages/bundle/base/cordis.patch.yml`、`examples/*/cordis.yml`
- **待提取**: cordis.yml 行结构、`!!js` 在 `config` 与 `disabled` 下的允许边界、profile 补丁层叠

**验证方式（执行时逐条勾选）**:

- [ ] 每个示例注明 `文件路径:行号`
- [ ] 代码为原样粘贴，未作简化
- [ ] 示例代表项目通用模式而非孤例

---

## 📚 第三阶段：子文档规划（待审核）

### 3.1 必需子文档清单（🔴 P0，7 篇）

- [ ] `architecture_overview.md` - 架构总览（中文深度解析）
  - **内容来源**: `docs/architecture.md`、`docs/cordis-primer.md`、`packages/README.md`、`vendor/README.md`
  - **预计行数**: 400-550
  - **关键章节**: Cordis 插件树 / Profile 与 Bundle 分层 / 核心包与 ctx key / 事件三域 / 双编译面 / vendoring 策略

- [ ] `capability_seams.md` - 能力接缝设计
  - **内容来源**: `docs/capability-seams.md`、`docs/glossary.md`、`packages/shell/`、`packages/subprocess/`
  - **预计行数**: 300-400
  - **关键章节**: 三角角色定义 / 何时拆分角色 / provider 互换的产品级后果 / request-spec 拆分模板 / 反模式

- [ ] `session_and_events.md` - 会话日志与事件系统
  - **内容来源**: `docs/subsystems/session.md`、`docs/event-producer-consumer.md`、`docs/persistence-catalog.md`、`packages/session/`
  - **预计行数**: 400-500
  - **关键章节**: 追加式日志 / `deriveMessages()` / `SessionEventMap` 声明合并 / 模型可见⟺已记录 / `SESSION_FORMAT_VERSION` 与 SQLite `SCHEMA_VERSION` / 分叉与恢复

- [ ] `agent_loop_and_tools.md` - Agent 循环与工具执行管线
  - **内容来源**: `docs/agent-lifecycle.md`、`docs/tool-execution-pipeline.md`、`docs/subsystems/core.md`、`docs/subsystems/tools.md`
  - **预计行数**: 400-500
  - **关键章节**: turn/step 语义 / 瀑布事件与 `next()` 契约 / inbox 与注入 / 取消与错误恢复 / 工具执行三阶段 / 超时守卫

- [ ] `plugin_development_guide.md` - 插件开发指南
  - **内容来源**: `docs/cookbook/`（adding-a-package / adding-a-tool / adding-an-llm-adapter / extension-cookbook）、`docs/cordis-tutorial/`
  - **预计行数**: 450-600
  - **关键章节**: 新包落位与命名 / 注册即效果 / 扩展点选择表 / 工具 UI 渲染意图 / 包 README 与 Model Experience 要求 / cordis.yml 挂载

- [ ] `prompt_management.md` - 系统提示词与上下文组装
  - **内容来源**: `docs/subsystems/system-prompt.md`、`packages/core/system-prompt/`、`packages/context/`、快照期望输出 `**/system-prompt.expected.md`
  - **预计行数**: 300-400
  - **关键章节**: 提示词分段注册 / 工具 schema 并入 / workspace instructions 与时间上下文 / 快照测试如何锁定提示词 / 压缩（compaction）对上下文的影响

- [ ] `model_configuration.md` - 模型配置与 LLM 适配器
  - **内容来源**: `packages/llm/README.md`、`docs/subsystems/llm-streaming.md`、`docs/subsystems/token-meter.md`、`docs/config-catalog.md`
  - **预计行数**: 300-400
  - **关键章节**: `ctx.llm` 适配器契约 / DeepSeek 直连与 pi-ai 多 provider / 重试策略 / token 计量与回放感知 / 默认模型选择 / 密钥与 base URL 配置

### 3.2 推荐子文档清单（🟡 P1，7 篇）

- [ ] `monorepo_and_build.md` - Monorepo 结构与构建体系
  - **推荐理由**: 219 包 + 双编译面 + vendored 依赖是本仓库最高频的认知门槛
  - **内容来源**: `pnpm-workspace.yaml`、`tsconfig.*.json`、`tsdown.config.ts`、`docs/development.md`、`docs/rescope.md`、`vendor/README.md`
  - **预计行数**: 350-450

- [ ] `testing_guide.md` - 测试策略
  - **推荐理由**: 四层测试（unit / coverage / snapshot / e2e）+ 平台矩阵，且覆盖率门禁易被误解
  - **内容来源**: `docs/testing.md`、`vitest.config.ts`、`vitest.snapshot.config.ts`、`vitest.e2e.config.ts`、`package.json` scripts
  - **预计行数**: 350-450

- [ ] `quality_gates.md` - 质量门禁清单
  - **推荐理由**: 30+ 个 `verify-*` / `gen-*` 脚本，缺少中文导航时贡献者极易漏跑
  - **内容来源**: `scripts/run-gates.ts`、`package.json` scripts、`.agents/skills/dsh-pre-push-checks/SKILL.md`
  - **预计行数**: 300-400

- [ ] `apps_cli_and_web.md` - CLI 与 Web 应用装配
  - **推荐理由**: `dsh` CLI 与 Web UI 是两个面向人类用户的产品入口；面向自动化的入口（ACP、JSON-RPC SDK、Python SDK、hooks）由 `sdk_and_protocols.md` 覆盖
  - **内容来源**: `apps/cli/src/`、`apps/web/`、`packages/host/`、`packages/client/`、`docs/user/guide/`、`docs/subsystems/client-modules.md`
  - **预计行数**: 300-400

- [ ] `sdk_and_protocols.md` - SDK 与外部协议
  - **推荐理由**: JSON-RPC SDK / ACP / hooks / Python SDK 四条外部集成路径
  - **内容来源**: `packages/sdk/`、`packages/acp/`、`packages/hooks/`、`packages/api/`、`python/README.md`、`python/development.md`
  - **预计行数**: 350-450

- [ ] `security_and_sandbox.md` - 安全、沙箱与数据边界
  - **推荐理由**: 覆盖 1.3C 全部外部服务边界；agent harness 会执行任意命令，安全边界是核心契约
  - **内容来源**: `packages/sandbox/`、`native/landlock-run/`、`packages/interaction/user-approval/`、`packages/credentials/`、`packages/session/session-telemetry-otel/README.md`、`docs/subsystems/approval.md`
  - **预计行数**: 400-500

- [ ] `deployment_guide.md` - 发布与分发
  - **推荐理由**: npm 发布、单文件可执行、Python 运行时分发三条链路
  - **内容来源**: `scripts/release/`、`package.json` `release:*` / `publish:*` scripts、`python/sdk-runtime/`、`THIRD_PARTY_NOTICES.md`
  - **预计行数**: 250-350

### 3.3 可选子文档（🟢 P2，3 篇）

- [ ] `cost_optimization.md` - Token 成本与上下文压缩
  - **内容来源**: `packages/llm/token-meter/`、`packages/compaction/`、`packages/spill/`、`docs/subsystems/compaction.md`
- [ ] `evaluation_metrics.md` - 评测与基准
  - **内容来源**: `BENCHMARK.md`、`vitest.snapshot.config.ts`、`examples/*/tests/snapshots/`
- [ ] `troubleshooting.md` - 常见故障排查
  - **内容来源**: `docs/postmortem/`、`docs/defensive-patterns.md`、`packages/runtime-diagnostics/`

### 3.4 项目定位触发项覆盖

| 项目定位信号 | 证据文件 | 文档规划影响 | 覆盖方式 |
| ------------ | -------- | ------------ | -------- |
| 开源维护/贡献流程 | `CONTRIBUTING.md` / `CONTRIBUTING.zh.md` / `LICENSE` | 贡献者流程、PR 与标签规范、Agent Note 义务 | 合并到 `quality_gates.md` + `AI_RULES.md` |
| 用户手册/使用指南 | `docs/user/guide/index.md`、`README.md` Run 章节 | Web UI 与 CLI 使用说明 | 合并到 `apps_cli_and_web.md`（正文链接至 `docs/user/`，不重述） |
| 自托管/部署运维 | `scripts/release/`、`python/sdk-runtime/`、`package.json` `release:*` | 打包、发布、运行时分发 | 单独文档 `deployment_guide.md` |
| 外部 API/数据授权 | `packages/llm/`、`packages/web/`、`packages/session/session-telemetry-otel/`、`packages/e2b/` | Provider、授权、上传数据与匿名 id 边界 | 单独文档 `security_and_sandbox.md` |
| 原生代码与平台约束 | `native/landlock-run/`、`packages/sandbox/sandbox-windows-acl/`、`package.json` `check:ci:windows-*` | 平台差异与原生构建 | 合并到 `security_and_sandbox.md`（约束）+ `monorepo_and_build.md`（构建） |
| 决策记录体系 | `.agents/notes/README.md`（1,390 篇） | AI 修改代码时的决策查阅与撰写义务 | 合并到 `quality_gates.md` + `AI_RULES.md` |

---

## 🎯 第四阶段：主文档章节规划

### 4.1 必需章节检查清单

- [ ] **📊 项目概览** - 数据来源: `package.json`、本方案 1.2 节统计命令
- [ ] **📂 关键目录速查** - 数据来源: 本方案 1.3 节、`AGENTS.md` Repository layout
- [ ] **🎯 场景快速导航** - 数据来源: 4.2 节
- [ ] **🚀 文档索引** - 数据来源: 3.1-3.3 节子文档清单 + `docs/` 上游索引
- [ ] **💻 核心代码模式** - 数据来源: 第二阶段提取结果
- [ ] **🛠️ 开发流程规范** - 数据来源: `AGENTS.md`、`CONTRIBUTING.md`、`.agents/skills/dsh-pre-push-checks/`
- [ ] **📋 命名规范** - 数据来源: `AGENTS.md` Conventions（`@deepseek-ai/dsh-<name>`、`packages/<group>/<pkg>/`、品牌化 id）
- [ ] **🏢 业务模块映射** - 数据来源: 本方案 1.4 节 + `packages/README.md`
- [ ] **⚠️ AI 编码禁忌** - 数据来源: `AGENTS.md` Conventions + 1.3B 不可破坏约束（**需人工复核补充**）
- [ ] **🔧 常见任务速查** - 数据来源: `docs/architecture.md` "Where new behavior goes" 表 + `docs/cookbook/`

### 4.2 场景快速导航规划

| 场景描述 | 对应文档 | 数据来源 |
| -------- | -------- | -------- |
| 我要新增一个模型可见工具 | `plugin_development_guide.md` | `docs/cookbook/adding-a-tool.md` + `packages/todo/` |
| 我要接入一个新的模型提供方 | `model_configuration.md` | `docs/cookbook/adding-an-llm-adapter.md` + `packages/llm/` |
| 我要新增一种能力（如新的执行后端） | `capability_seams.md` | `docs/capability-seams.md` + `packages/shell/` |
| 我要让某个信息进入模型上下文 | `prompt_management.md` + `session_and_events.md` | `docs/architecture.md:96` + `packages/context/` |
| 我要拦截或修改某次模型请求 | `agent_loop_and_tools.md` | `docs/agent-lifecycle.md` |
| 我要新增一个 npm 包 | `monorepo_and_build.md` + `plugin_development_guide.md` | `docs/cookbook/adding-a-package.md` |
| 我要改 Web UI | `apps_cli_and_web.md` | `docs/subsystems/client-modules.md` + `packages/client/` |
| 我提交前该跑哪些检查 | `quality_gates.md` | `.agents/skills/dsh-pre-push-checks/SKILL.md` + `scripts/run-gates.ts` |
| 我要写测试 / 快照不通过 | `testing_guide.md` | `docs/testing.md` + `vitest.snapshot.config.ts` |
| 我要理解沙箱与审批为什么拦了我 | `security_and_sandbox.md` | `packages/sandbox/` + `docs/subsystems/approval.md` |
| 我要发布一个版本 | `deployment_guide.md` | `scripts/release/` |
| 上下文爆了 / token 成本过高 | `cost_optimization.md` | `packages/compaction/` + `packages/llm/token-meter/` |

**⚠️ 人工验证点**: 上述 12 个场景是否覆盖 80% 常见开发需求；是否遗漏 subagent / workflow / skill 三条链路（当前合并在 `plugin_development_guide.md`，若用户认为需独立成文请在审核意见中指出）。

---

## ⚠️ 风险点与注意事项

### 已识别的风险

1. **事实归属地重复（最高风险）**: `dev_docs/` 子文档会为架构、测试、发布等事实建立第二个归属地，与 `docs/AGENTS.md:17` 的"一个事实一个归属地"直接冲突

   - **影响**: 长期必然与 `docs/` 漂移；漂移后 AI 读到过期中文文档会产出错误改动
   - **缓解措施**: ① 每篇子文档 frontmatter 的 `dependencies` 必须指向其上游 `docs/` 事实源；② 正文每个章节开头标注"上游事实源: `docs/xxx.md`"；③ 事件表/工具表/配置表/依赖图一律链接生成产物，禁止重述；④ 后续维护走 AICC 路径 C（`@commit` 增量更新）
   - **用户决策**: 用户已明确选择"AICC 标准全量体系"，本风险为**已接受风险**

2. **`dev_docs/**/README.md` 触发双语配对门禁**: `scripts/translation-pairing.ts:127` 的 `README_ARTIFACT` 正则匹配任意路径下的 `README.md`，且 `isTranslationScopeFile`（`:180-186`）未排除 `dev_docs/`

   - **影响**: 批次 6 创建 `plans/README.md`、`memos/README.md`、`knowledge/README.md` 后，`pnpm run verify-translation-pairing` 与 `pnpm run doc-sync` 将要求补 `.zh.md` + `.i18n.yaml` 配对
   - **缓解措施**: 二选一——(a) 将三个 README 加入 `scripts/translation-pairing.manifest.json` 的 `excluded` 数组；(b) 在 `TRANSLATION_SCOPE_GLOB_EXCLUDES`（`:146`）与 `isTranslationSourceExcluded` 中加入 `dev_docs/`。**方案 (b) 更彻底**，因为它同时覆盖未来新增的任何 `dev_docs` README
   - **状态**: 待用户选择修复路径，**阻断批次 6**（不阻断批次 1-5）

3. **无法运行仓库门禁验证 dev_docs 的真实影响**: `node_modules/` 不存在，`pnpm install` 未执行

   - **影响**: 风险 2 的结论目前是 E3（源码静态阅读）而非 E4（实际运行）
   - **缓解措施**: 正式生成前执行 `pnpm install`，然后运行 `pnpm run verify-translation-pairing` 与 `pnpm run verify-md-links` 取得 E4 证据

4. **文档生成量大导致会话中断**: 22 个产物、18-26 小时

   - **缓解措施**: 强制使用 `generation_progress.md` 逐产物记录；每批结束请求用户确认；支持断点续传

5. **生成产物类文档易过期**: 工具目录、配置目录、模块图由生成器产出并做 CI 新鲜度门禁

   - **缓解措施**: `dev_docs` 一律链接不复制；若必须摘录，标注"截至 `verified_at`，权威源为 `docs/xxx.md`"

6. **框架边界污染**: `AI-Coding-Context` 是指向仓库外绝对路径的符号链接，且当前**未被 gitignore**

   - **影响**: 直接提交会在其他开发者机器上断链
   - **缓解措施**: 用户已确认将其加入 `.gitignore`（见问题报告 🔴 问题 2）

### 需要人工确认的项目特性

**准入规则**: 以下各项均已确认无法由代码、配置、锁文件、README 或现有项目文档回答。

1. **`dev_docs` 与 `docs/` 的长期维护责任归属**

   - **当前保守结论**: `dev_docs` 定位为中文文档体系，`docs/` 保持为英文/中文双语权威源；两者冲突时以 `docs/` 为准
   - **已检查证据**: `docs/AGENTS.md:17-34` 分层表；`AGENTS.md` Conventions；`scripts/doc-budgets.manifest.json`（`dev_docs` 不在预算清单内）
   - **为什么代码或仓库文档无法回答**: 属于团队流程与所有权决策，仓库未记录 `dev_docs` 这一新层的维护约定
   - **blocks_phase1**: false
   - **回写目标**: `AI_RULES.md` + `architecture_overview.md` 开头的"上游事实源"声明
   - **建议做法**: 在 `AI_RULES.md` 中写死一条硬规则——"`dev_docs` 与 `docs/` 冲突时以 `docs/` 为准，并立即更新 `dev_docs`"

2. **双语配对门禁的修复路径选择**

   - **当前保守结论**: 采用方案 (b)，在 `TRANSLATION_SCOPE_GLOB_EXCLUDES` 与 `isTranslationSourceExcluded` 中排除 `dev_docs/`
   - **已检查证据**: `scripts/translation-pairing.ts:127,146,180-186`；`scripts/translation-pairing.manifest.json`（当前 `excluded` 仅 8 项，全为 `docs/` 与 `.agents/notes/` 下的文件）
   - **为什么代码或仓库文档无法回答**: 修改仓库门禁脚本属于维护者治理决策，且需要一篇 Agent Note；仓库文档未规定第三方文档体系如何接入门禁
   - **blocks_phase1**: false（**阻断批次 6**）
   - **回写目标**: `generation_plan.md` 批次 6 前置条件 + `quality_gates.md`
   - **建议做法**: 在批次 6 前提交一个独立 PR，修改 `scripts/translation-pairing.ts` 并附 Agent Note

3. **`dev_docs` 是否需要英文对照版**

   - **当前保守结论**: 仅中文，不生成英文版
   - **已检查证据**: 用户明确指示"文档使用中文"；仓库 `docs/` 与 `README` 均为双语，`docs/i18n/README.md` 定义配对契约
   - **为什么代码或仓库文档无法回答**: 属于产品与社区策略——本仓库面向国际开源社区（Discord、GitHub Discussions 均为英文），中文独有文档层是否可接受由维护者决定
   - **blocks_phase1**: false
   - **回写目标**: `AI_RULES.md`
   - **建议做法**: 若未来需要英文版，走 `docs/i18n/` 既有配对流程，而非在 `dev_docs` 内另建机制

4. **subagent / workflow / skill 三条扩展链路是否需独立成文**

   - **当前保守结论**: 合并进 `plugin_development_guide.md` 的"扩展点选择"章节
   - **已检查证据**: `packages/subagent/`、`packages/workflow/`、`packages/skill/` 各自有 README 与 `docs/subsystems/` 页；`docs/architecture.md:108-128` 扩展点表已覆盖
   - **为什么代码或仓库文档无法回答**: 属于文档粒度偏好，取决于团队对这三条链路的实际使用频率
   - **blocks_phase1**: false
   - **回写目标**: `generation_plan.md` 3.1/3.2 节子文档清单
   - **建议做法**: 若团队高频开发 subagent 或 workflow，在审核意见中要求拆出 `subagent_and_workflow.md`

---

## 📊 质量保证措施

### 数据来源追溯

- [x] 项目规模 → `project_scanner.py` + `find`/`wc` 命令输出（见 1.2 节）
- [x] 目录结构 → `ls -1` 输出 + `AGENTS.md` Repository layout
- [ ] 代码示例 → 实际文件路径+行号（第二阶段执行）
- [x] 业务模块 → `packages/README.md` 层级表
- [x] 架构特点 → `docs/architecture.md` 具体行号

### 验证检查点

生成正式文档前必须验证:

- [ ] 所有代码示例都来自真实文件
- [x] 所有引用的文件路径都已验证存在
- [x] 所有统计数据都有命令支持
- [x] 所有架构特点都有出处
- [x] 所有业务模块都已确认

---

## 🧾 证据与验证记录

### 证据等级

| 等级 | 证据来源 | 允许措辞 |
| ---- | -------- | -------- |
| E1 | 目录结构、文件名、文件数量 | "疑似""风险假设""建议后续验证" |
| E2 | 配置文件、锁文件、README、项目文件 | "已从配置确认""技术栈事实" |
| E3 | 源码片段、协议、关键函数、调用链 | "代码显示""实现方式为" |
| E4 | 构建、测试、脚本运行、工具检查结果 | "已验证""检查通过/失败" |

### 关键事实记录

| 编号 | 事实 | 证据等级 | 来源文件 | 验证方式 | 当前结论 |
| --- | ---- | -------- | -------- | -------- | -------- |
| F1 | 项目共 7,238 文件 / 1,211 目录，complexity_level=advanced | E4 | — | `project_scanner.py --mode summary --exclude-standard` | 已确认 |
| F2 | `packages/` 下 219 个 workspace 包 | E4 | `packages/` | `find` + `wc -l` 统计 `packages/` 下 package.json | 已确认 |
| F3 | TypeScript 501,274 行 / TSX 66,872 行 / Markdown 170,752 行 | E4 | 全仓 | `find` + `xargs wc -l` 汇总行数 | 已确认（含 vendor 与 scripts） |
| F4 | 224 个测试目录、1739 个测试文件（测试目录下全部文件，含快照与夹具）；其中 `packages/` 下 `*.spec.ts`/`*.test.ts` 为 643 个 | E4 | 全仓测试目录 | `semantic_review_checker.scan_test_topology()` + `find` 复核 | 已确认 |
| F5 | Agent Notes 1,390 篇 | E4 | `.agents/notes/` | `find` + `wc -l` 统计 `.agents/notes/` 下 Markdown | 已确认 |
| F6 | pnpm@11.7.0，Node 引擎范围为 `^22.19.0` 或 `>=24.0.0`，TypeScript `^6.0.3`，Vitest `^4.1.8` | E2 | `package.json` | 读取 | 已确认 |
| F7 | workspace 成员含 `vendor/*`、`packages/*/*`、`native/landlock-run`、`apps/*`、`website`、`examples`、`python/sdk-runtime` | E2 | `pnpm-workspace.yaml` | 读取 | 已确认 |
| F8 | Cordis 为 vendored 源码副本（9 包），`@deepseek-ai/cosmokit`/`schemastery` 通过 `overrides` link 到 vendor | E2 | `pnpm-workspace.yaml`、`vendor/` | 读取 + `ls` | 已确认 |
| F9 | 一切皆插件、无特权内核；模型适配器/工具注册表/会话日志/Agent 循环本身均为插件 | E2 | `docs/architecture.md:11-13` | 读取 | 已确认 |
| F10 | 模型可见即已记录，由运行时不变量断言 | E2 | `docs/architecture.md:96` | 读取 | 已确认 |
| F11 | 瀑布监听器必须调用 `next()`，否则短路 | E2 | `docs/architecture.md:84` | 读取 | 已确认 |
| F12 | `verify-md-wrap` / `verify-md-links` 的 `PATTERNS` 不含 `dev_docs/` | E3 | `scripts/verify-md-wrap.ts:18-29`、`scripts/verify-md-links.ts:19-31` | 源码读取 | 已确认 |
| F13 | `verify-doc-budgets` 由 `scripts/doc-budgets.manifest.json` 驱动，仅检查清单内文件 | E3 | `scripts/verify-doc-budgets.ts:1-46` | 源码读取 | 已确认，`dev_docs` 不受影响 |
| F14 | 任意路径下的 `README.md` 均落入双语配对作用域 | E3 | `scripts/translation-pairing.ts:127,180-186` | 源码读取 | 已确认（**未运行时验证**，`node_modules` 缺失） |
| F15 | 无向量数据库 / RAG / embedding 依赖 | E4 | `packages/**/package.json` | `grep -rilE` 七关键词检索 → 0 命中（完整命令见"最终验证命令"） | 已确认 |
| F16 | LLM 适配器为 `llm-deepseek`（直连）与 `llm-pi-ai`（多 provider），另有 `llm-retry`、`token-meter` | E2 | `packages/llm/README.md:10-13` | 读取 | 已确认 |
| F17 | OTel 遥测存在上传模式，资源标识含匿名 `user.id`（`$DSH_HOME/.anonymous-user-id`，删除即重置） | E2 | `packages/session/session-telemetry-otel/README.md:5` | 读取 | 已确认 |
| F18 | 覆盖率门禁为 `test:coverage`（`packages/*/*/src` 每文件 100%），非 `test` | E2 | `package.json` scripts、`AGENTS.md` Commands、`docs/testing.md` | 读取 | 已确认 |
| F19 | host / client 双编译面因两侧在相同 ctx key 上合并不同服务而拆分 | E2 | `tsconfig.host.json:2-4`、`tsconfig.client.json:2-6` | 读取 | 已确认 |
| F20 | `node_modules/` 不存在，仓库门禁当前不可运行 | E4 | 仓库根 | `test -d node_modules` → 失败 | 已确认 |
| F21 | `AI-Coding-Context` 符号链接未被 gitignore | E4 | `.gitignore` | `git check-ignore -v AI-Coding-Context` → exit 1 | 已确认 |
| F22 | 环境：macOS 26.6 (Darwin 25.6.0)、Python 3.9.6、Node v24.13.0 | E4 | — | `env_diagnosis.py` | 已确认 |

### 量化声明来源

| 声明 | 数值 | 来源命令/文件 | 记录位置 |
| ---- | ---- | ------------- | -------- |
| 总文件数 | 7,238 | `project_scanner.py --mode summary --exclude-standard` | 本文档 1.2 节 / F1 |
| 总目录数 | 1,211 | 同上 | 1.2 节 / F1 |
| workspace 包数 | 219 | `find` + `wc -l` 统计 `packages/` 下 package.json | 1.2 节 / F2 |
| vendored 包数 | 9 | `find` + `wc -l` 统计 `vendor/` 下 package.json | 1.2 节 |
| TypeScript 行数 | 501,274 | `find` + `xargs wc -l` 汇总 `*.ts` 行数 | 1.2 节 / F3 |
| TSX 行数 | 66,872 | 同上（`*.tsx`） | 1.2 节 / F3 |
| Markdown 行数 | 170,752 | 同上（`*.md`） | 1.2 节 / F3 |
| Python 行数 | 4,373 | 同上（`*.py`） | 1.2 节 |
| 测试目录数 | 224 | `semantic_review_checker --check-test-topology` | 1.2 节 / F4 |
| 测试文件数 | 1739 | 同上（测试目录下全部文件） | 1.2 节 / F4 |
| `packages/` 下 spec/test 源文件数 | 643 | `find packages` 按 `*.spec.ts` / `*.test.ts` 计数 | 1.2 节 / F4 |
| Agent Notes 数 | 1,390 | `find` + `wc -l` 统计 `.agents/notes/` 下 Markdown | 1.2 节 / F5 |
| 计划产物数 | 22 | 本方案 3.1-3.3 节 + 批次计划 | 执行计划章节 |

> **回写规则**: 上述任一数值变化时，必须全文检索旧值与同义表述（如"约 50 万行"、"两百余包"）并同步更新 `project_analysis_report.md` 与 `generation_progress.md`。

### 测试资产扫描结果

- **扫描范围**: `packages/*/*/tests/`、`apps/*/tests/`、`examples/**/tests/`、`python/**`
- **发现结果**: 650 个 `*.spec.ts` / `*.test.ts`；另有独立的快照测试配置（`vitest.snapshot.config.ts`）、e2e 配置（`vitest.e2e.config.ts`）、Web 与压力测试配置（`vitest.web*.config.ts`）；`examples/*/tests/snapshots/` 下存在样例工程快照期望输出；`pytest.ini` 表明存在 Python 测试
- **已纳入 testing_guide / 主文档**: 是（`testing_guide.md` 为 P1 必生成项；四层测试拓扑写入主文档"常见任务速查"）
- **完整拓扑清单**: 见本文档[附录 A：测试目录拓扑清单](#-附录-a测试目录拓扑清单生成产物)

### 推荐实践事实源

| 主题 | 事实源文件 | 证据位置 | 说明 |
| ---- | ---------- | -------- | ---- |
| 提交前检查选择 | `.agents/skills/dsh-pre-push-checks/SKILL.md` | 全文 | 维护者明确要求"按 diff 选最小检查集"，禁止反射式跑全量套件 |
| 覆盖率门禁语义 | `docs/testing.md`、`package.json` | `test:coverage` script | 覆盖率门禁是 `test:coverage` 而非 `test` |
| 文档落位判断 | `docs/AGENTS.md` | `:15-34` 分层表 | 一个事实一个归属地 |
| Agent Note 义务 | `AGENTS.md`、`.agents/notes/README.md` | Conventions 章节 | 非平凡改动必须同 PR 附 Agent Note |
| 扩展点选择 | `docs/architecture.md` | `:108-128` 表 | 新行为应挂在哪个机制上 |
| 防御性模式 | `docs/defensive-patterns.md` | 全文 | 生命周期、并发、子进程、teardown 工作前必读 |

> **治理约束声明**: `CONTRIBUTING.md`、`AGENTS.md`、`docs/AGENTS.md`、`.agents/skills/dsh-pre-push-checks/` 构成本仓库的治理事实源。本方案的测试、分支、PR、格式化与文档建议均不得与其冲突；`dev_docs` 的任何流程建议若与之抵触，以仓库治理文件为准。

### 最终验证命令

```bash
# AICC 侧（Phase 1 hard gate）
python3 AI-Coding-Context/tools/py/summary_validator.py --dir dev_docs/_analysis --recursive --strict
python3 AI-Coding-Context/tools/py/doc_health_checker.py --full-check --doc-dir dev_docs
node AI-Coding-Context/tools/js/doc_health_checker.js --full-check --doc-dir dev_docs
python3 AI-Coding-Context/tools/py/semantic_review_checker.py --full-check --doc-dir dev_docs --repo-root .
node AI-Coding-Context/tools/js/semantic_review_checker.js --full-check --doc-dir dev_docs --repo-root .

# 仓库侧（需先 pnpm install；批次 6 前必须取得 E4 证据）
pnpm install
pnpm run verify-translation-pairing
pnpm run verify-md-links
pnpm run verify-doc-budgets
```

### 审核与确认留痕

- **方案生成完成时间**: 2026-08-18 15:16
- **等待人工审核状态**: 待审核
- **用户确认时间**: 未确认
- **进入正式生成时间**: 未开始

### 复查回写要求

当复查结论改变事实状态时，必须同步更新本文档摘要、正文、表格、待确认清单、行动计划、证据记录以及 `project_analysis_report.md` 与 `generation_progress.md`。禁止只在文末追加"复查记录"而保留正文旧结论。

---

## 🔎 Phase 1 方案复查清单

- [x] 项目类型、技术栈、依赖、测试、部署方式均有 E2/E3/E4 证据；仅 E1 证据未写成强结论
- [x] `证据与验证记录` 表包含 `证据等级` 列，关键事实均可追溯来源文件或命令
- [x] 已建立 `evidence_inventory`（F1-F22）与量化声明来源表，并写明回写规则
- [x] 已填写 `project_positioning_constraints`（1.3B），项目愿景与不可破坏约束已进入正式文档计划
- [x] 已填写 `ai_external_service_boundaries`（1.3C），DeepSeek API、pi-ai、Web 出站、OTel 上传、E2B 沙箱与"无 RAG/向量库"边界均已区分
- [x] 子文档清单覆盖项目定位触发项；不单独成文者已写明合并覆盖位置（3.4 节）
- [x] 待用户确认项仅包含代码、配置、锁文件、README、现有项目文档无法回答的问题
- [x] 每个待用户确认项均含 `当前保守结论`、`已检查证据`、`为什么代码或仓库文档无法回答`、`blocks_phase1`、`回写目标`
- [x] `CONTRIBUTING.md`、`AGENTS.md`、`docs/AGENTS.md`、pre-push skill 等治理约束已纳入建议边界，质量建议与其不冲突
- [x] 风险与注意事项均有证据来源；风险 2 明确标注为 E3（未运行时验证）
- [x] 质量保证措施与真实技术栈匹配，并写明 `node_modules` 缺失时的替代复核说明
- [x] 复查结果已同步回写三件套
- [x] 下一步动作保持为"建议通过，等待用户确认"

---

## 🚀 执行计划

> **所有批次均在用户确认后执行。** 每批结束更新 `generation_progress.md` 并请求用户确认是否继续。

### 批次 1: 主文档与架构基座（预计 4-5 小时）

1. `dev_docs/AI_Coding_Context.md` - 主文档
2. `dev_docs/architecture_overview.md`
3. `dev_docs/capability_seams.md`

**批次出口条件**: 用户确认主文档的场景导航与架构描述准确。

### 批次 2: 运行时核心（预计 5-6 小时）

4. `dev_docs/session_and_events.md`
5. `dev_docs/agent_loop_and_tools.md`
6. `dev_docs/prompt_management.md`
7. `dev_docs/model_configuration.md`

### 批次 3: 开发者路径（预计 3-4 小时）

8. `dev_docs/plugin_development_guide.md`
9. `dev_docs/monorepo_and_build.md`
10. `dev_docs/quality_gates.md`

### 批次 4: 产品面与集成（预计 3-4 小时）

11. `dev_docs/testing_guide.md`
12. `dev_docs/apps_cli_and_web.md`
13. `dev_docs/sdk_and_protocols.md`

### 批次 5: 边界与运维（预计 3-4 小时）

14. `dev_docs/security_and_sandbox.md`
15. `dev_docs/deployment_guide.md`
16. `dev_docs/cost_optimization.md`
17. `dev_docs/evaluation_metrics.md`
18. `dev_docs/troubleshooting.md`

### 批次 6: 流程目录与规则（预计 1-1.5 小时）

> **前置条件（硬性）**: 必须先解决 `dev_docs/**/README.md` 的双语配对门禁冲突（风险 2），并以 `pnpm run verify-translation-pairing` 取得 E4 证据。

19. `dev_docs/plans/{active,done,archive}/` + `dev_docs/plans/README.md`
20. `dev_docs/memos/` + `dev_docs/memos/README.md`
21. `dev_docs/knowledge/{troubleshooting,patterns,performance}/` + `dev_docs/knowledge/README.md`
22. `dev_docs/rules/combined/AI_RULES.md`

### 批次 7: 首版质量验收（预计 1.5-2 小时）

- 运行 Python/JS 两套 `doc_health_checker --full-check`
- 运行 Python/JS 两套 `semantic_review_checker --full-check`
- 生成 `dev_docs/_analysis/health_check_report.md`（含结构化 `machine_checks` 与 `accepted_issues`）
- 复跑检查，确认健康报告自身不触发模板残留、verdict 冲突或产物计数不一致
- 更新 `generation_progress.md` 的"首版质量验收记录"

---

## ✅ 审核清单（人工填写）

### 数据准确性审核

- [ ] 项目规模数据已验证（7,238 文件 / 219 包 / 501,274 行 TS）
- [ ] 目录结构描述准确
- [ ] 业务模块（包组）划分合理
- [ ] 架构特点识别准确，无臆测

### 文档规划审核

- [ ] 17 篇子文档清单合理，无冗余无遗漏
- [ ] 排除 `rag_architecture.md` / `vector_database.md` 的理由成立
- [ ] 主文档章节规划完整
- [ ] 12 个场景导航覆盖常见需求

### 风险评估

- [ ] 事实归属地重复风险的缓解措施可接受
- [ ] 双语配对门禁修复路径已选定
- [ ] 4 个待确认项已逐条答复

---

## 📝 审核意见（人工填写）

### 需要修改的部分

1. [待填写]

### 需要补充的内容

1. [待填写]

### 批准意见

- [ ] **批准，可以开始生成文档**
- [ ] **需要修改后重新提交方案**
- [ ] **拒绝，原因如下**: [说明]

---

**签名**: [待签名]
**审核日期**: [待填写]

---

## ⚠️ 风险应对预案 (Risk Mitigation Plan)

| 风险 | 可能性 | 影响 | 应对措施 |
| :--- | :----: | :--: | :------- |
| `dev_docs` 与 `docs/` 事实漂移 | 高 | 高 | 每篇标注上游事实源；生成产物只链接不复制；走路径 C 增量更新 |
| `README.md` 触发双语配对门禁 | 高 | 中 | 批次 6 前修改 `scripts/translation-pairing.ts` 并附 Agent Note |
| 文件过大无法一次读取 | 高 | 中 | 先取大纲再分段读取，优先关键函数与配置 |
| 219 包依赖关系难以理清 | 中 | 高 | 直接引用 `docs/module-graph.md` 生成产物 |
| AI token 限制导致会话中断 | 高 | 低 | 每批设检查点，`generation_progress.md` 逐产物记录 |
| 时间不足无法完成全部文档 | 中 | 中 | 优先完成 P0 七篇，P1/P2 后续补充 |
| 引用的 `docs/` 行号因上游变更而失效 | 中 | 中 | 引用锚点优先用小节标题而非行号；`verified_at` 逐篇记录 |
| 生成的中文文档与源码事实冲突 | 低 | 高 | 首版验收强制跑 `semantic_review_checker` 双实现交叉验证 |

### 应急处理流程

**遇到阻塞时**: 标记问题 → 跳过继续其他部分 → 汇总后统一询问用户。

**时间不足时**: 优先完成批次 1-3（P0 + 开发者路径）→ 批次 4-5 后期补充 → 确保已完成文档质量达标。

**质量不达标时**: 立即停止生成新文档 → 返工修复 → 重新评估剩余工作量。

---

## ✅ 质量检查清单 (Quality Checklist)

### 准确性验证

- [ ] 所有代码示例路径可追溯（`文件:行号`）
- [ ] 架构描述与 `docs/architecture.md` 一致
- [ ] 事件与工具表未手工重述生成产物
- [ ] 配置说明与 `docs/config-catalog.md` 一致
- [ ] 命令与 `package.json` scripts 一致

### 完整性验证

- [ ] 主文档包含全部 17 篇子文档链接
- [ ] 回合流程有完整 mermaid 图
- [ ] 至少 3 个真实扩展场景的 step-by-step 指南
- [ ] 1.3B 全部不可破坏约束已落入正式文档
- [ ] 1.3C 全部外部服务边界已落入 `security_and_sandbox.md`

### 可用性验证

- [ ] AI 依据主文档能在 5 分钟内定位所需信息
- [ ] 所有 markdown 链接可跳转
- [ ] mermaid 图表能正确渲染
- [ ] 每篇 frontmatter 通过 `summary_validator --strict`

### 一致性验证

- [ ] 文档间交叉引用有效
- [ ] 量化数值全仓一致（对照"量化声明来源"表）
- [ ] 与 `AGENTS.md` 治理约束无冲突

---

## 📌 备注

- 框架边界：`AI-Coding-Context/` 为 AICC 框架符号链接，已在全部扫描与统计中排除，其内容不得进入项目分析结果
- 本方案对 AICC `core/project_types/ai_llm_app.md` 的偏离已在 1.1 节显式说明并给出 E4 证据

---

## 📎 附录 A：测试目录拓扑清单（生成产物）

共 224 个测试目录，目录下文件合计 1739 个（含快照期望输出与夹具，非仅测试源文件）。

本清单用于让 `semantic_review_checker --check-test-topology` 可验证测试资产覆盖度，并作为 `testing_guide.md` 的编写依据。

**再生成命令**（清单过期时重跑并整体替换本节表格）:

```bash
python3 AI-Coding-Context/tools/py/semantic_review_checker.py --check-test-topology --doc-dir dev_docs --repo-root .
```

| 测试目录 | 目录下文件数 |
| -------- | -----------: |
| `apps/cli/tests/` | 17 |
| `apps/web/tests/` | 220 |
| `examples/acp-agent/tests/` | 395 |
| `examples/headless-agent/tests/` | 84 |
| `examples/jsonrpc-agent/tests/` | 20 |
| `native/landlock-run/test/` | 2 |
| `packages/acp/acp/tests/` | 9 |
| `packages/api/gateway/tests/` | 2 |
| `packages/api/remotes/tests/` | 2 |
| `packages/attachment/attachment-local/tests/` | 3 |
| `packages/attachment/attachment/tests/` | 1 |
| `packages/boot/app-boot/tests/` | 6 |
| `packages/boot/cmdline/tests/` | 1 |
| `packages/bundle/base/tests/` | 2 |
| `packages/bundle/headless/tests/` | 2 |
| `packages/bundle/web-app/tests/` | 3 |
| `packages/client/connection/tests/` | 11 |
| `packages/client/hmr/tests/` | 1 |
| `packages/client/locale/tests/` | 6 |
| `packages/client/modules/tests/` | 2 |
| `packages/client/runtime/tests/` | 25 |
| `packages/client/schema-form/tests/` | 2 |
| `packages/client/ui-agent-preset/tests/` | 7 |
| `packages/client/ui-attachment/tests/` | 5 |
| `packages/client/ui-commands/tests/` | 5 |
| `packages/client/ui-conversation/tests/` | 29 |
| `packages/client/ui-deliverables/tests/` | 2 |
| `packages/client/ui-directory-picker-browse/tests/` | 2 |
| `packages/client/ui-directory-picker-native/tests/` | 1 |
| `packages/client/ui-goal/tests/` | 3 |
| `packages/client/ui-input-trigger/tests/` | 5 |
| `packages/client/ui-jobs/tests/` | 2 |
| `packages/client/ui-layout/tests/` | 6 |
| `packages/client/ui-message-feedback/tests/` | 3 |
| `packages/client/ui-model-selection/tests/` | 2 |
| `packages/client/ui-permission-presets/tests/` | 3 |
| `packages/client/ui-plan/tests/` | 2 |
| `packages/client/ui-primitives/tests/` | 67 |
| `packages/client/ui-settings-general/tests/` | 7 |
| `packages/client/ui-settings-models/tests/` | 10 |
| `packages/client/ui-settings-plugin-inventory/tests/` | 3 |
| `packages/client/ui-settings-plugins/tests/` | 5 |
| `packages/client/ui-settings/tests/` | 3 |
| `packages/client/ui-sidebar/tests/` | 8 |
| `packages/client/ui-skill/tests/` | 2 |
| `packages/client/ui-slots/tests/` | 4 |
| `packages/client/ui-subagent/tests/` | 2 |
| `packages/client/ui-theme/tests/` | 8 |
| `packages/client/ui-tool/tests/` | 16 |
| `packages/client/ui-trajectory/tests/` | 8 |
| `packages/client/ui-user-questions/tests/` | 4 |
| `packages/client/ui-workflow-run/tests/` | 1 |
| `packages/client/ui-workspace/tests/` | 8 |
| `packages/client/web-react/tests/` | 7 |
| `packages/client/web/tests/` | 5 |
| `packages/code-runtime/code-runtime-worker-thread/tests/` | 6 |
| `packages/code-runtime/code-runtime/tests/` | 2 |
| `packages/compaction/command-compact/tests/` | 3 |
| `packages/compaction/compaction-basic/tests/` | 4 |
| `packages/compaction/compaction-tool-result-pruner/tests/` | 2 |
| `packages/compaction/compaction/tests/` | 3 |
| `packages/context/agent-instructions/tests/` | 2 |
| `packages/context/session-reference/tests/` | 1 |
| `packages/context/time-context/tests/` | 4 |
| `packages/context/tmux-context/tests/` | 1 |
| `packages/core/agent-default-model/tests/` | 1 |
| `packages/core/agent-loop/tests/` | 20 |
| `packages/core/agent-tool-presentation/tests/` | 1 |
| `packages/core/agent/tests/` | 6 |
| `packages/core/scope/tests/` | 3 |
| `packages/core/session/tests/` | 13 |
| `packages/core/system-prompt/tests/` | 4 |
| `packages/core/tools/tests/` | 12 |
| `packages/credentials/credentials-local/tests/` | 4 |
| `packages/credentials/credentials/tests/` | 3 |
| `packages/e2b/e2b/tests/` | 2 |
| `packages/e2b/fs-e2b/tests/` | 1 |
| `packages/e2b/subprocess-e2b/tests/` | 2 |
| `packages/examples/acp-demo/tests/` | 3 |
| `packages/examples/agent-spine-demo/tests/` | 3 |
| `packages/extensions/cordis-client-runner/tests/` | 5 |
| `packages/extensions/cordis-host-runner/tests/` | 6 |
| `packages/extensions/tool-cordis/tests/` | 1 |
| `packages/extensions/ui-cordis/tests/` | 3 |
| `packages/feedback/command-feedback/tests/` | 2 |
| `packages/feedback/message-feedback/tests/` | 4 |
| `packages/fs/fs-local/tests/` | 3 |
| `packages/fs/fs-observation-policy/tests/` | 1 |
| `packages/fs/fs-sandbox/tests/` | 2 |
| `packages/fs/fs/tests/` | 2 |
| `packages/fs/tool-fs-search/tests/` | 5 |
| `packages/fs/tool-fs/tests/` | 8 |
| `packages/fs/tool-str-replace-editor/tests/` | 1 |
| `packages/goal/command-goal/tests/` | 1 |
| `packages/goal/goal-round-driver/tests/` | 2 |
| `packages/goal/goal/tests/` | 4 |
| `packages/goal/tool-goal/tests/` | 1 |
| `packages/guard/repeat-tool-reminder/tests/` | 1 |
| `packages/guard/timeout-policy/tests/` | 1 |
| `packages/hooks/hook-protocol/tests/` | 7 |
| `packages/hooks/hooks-claude-code/tests/` | 7 |
| `packages/hooks/hooks-codex/tests/` | 6 |
| `packages/host/apiproxy/tests/` | 20 |
| `packages/host/directory-picker-auto/tests/` | 2 |
| `packages/host/directory-picker-browse/tests/` | 1 |
| `packages/host/directory-picker-native/tests/` | 6 |
| `packages/host/directory-picker/tests/` | 1 |
| `packages/host/frontend-static/tests/` | 1 |
| `packages/host/plugin-inventory/tests/` | 2 |
| `packages/host/webserver/tests/` | 1 |
| `packages/identity/anonymous-user-id/tests/` | 2 |
| `packages/interaction/commands/tests/` | 2 |
| `packages/interaction/permission-presets/tests/` | 3 |
| `packages/interaction/tool-ask-user/tests/` | 1 |
| `packages/interaction/user-approval/tests/` | 2 |
| `packages/interaction/user-questions/tests/` | 1 |
| `packages/jobs/jobs-local/tests/` | 2 |
| `packages/jobs/jobs/tests/` | 2 |
| `packages/jobs/tool-jobs/tests/` | 1 |
| `packages/llm/llm-deepseek/tests/` | 9 |
| `packages/llm/llm-pi-ai/tests/` | 13 |
| `packages/llm/llm-retry/tests/` | 5 |
| `packages/llm/llm/tests/` | 11 |
| `packages/llm/token-meter/tests/` | 3 |
| `packages/lsp/lsp-stdio/tests/` | 10 |
| `packages/lsp/lsp/tests/` | 1 |
| `packages/lsp/tool-lsp/tests/` | 4 |
| `packages/mcp/mcp-client/tests/` | 6 |
| `packages/plan/plan-mode/tests/` | 4 |
| `packages/preset/agent-presets/tests/` | 23 |
| `packages/preset/persona/tests/` | 1 |
| `packages/runtime-diagnostics/invariants/tests/` | 1 |
| `packages/sandbox/sandbox-local/tests/` | 6 |
| `packages/sandbox/sandbox-policy/tests/` | 2 |
| `packages/sandbox/sandbox-windows-acl/tests/` | 14 |
| `packages/sandbox/sandbox/tests/` | 3 |
| `packages/schedule/schedule/tests/` | 7 |
| `packages/sdk/client/tests/` | 3 |
| `packages/sdk/protocol/tests/` | 1 |
| `packages/sdk/server/tests/` | 4 |
| `packages/session-query/session-log-export/tests/` | 7 |
| `packages/session-query/session-query-sqlite/tests/` | 3 |
| `packages/session-query/session-query/tests/` | 4 |
| `packages/session-query/tool-session-query/tests/` | 2 |
| `packages/session/session-checkpoint-policy/tests/` | 3 |
| `packages/session/session-persistence-jsonl/tests/` | 4 |
| `packages/session/session-persistence-sqlite/tests/` | 1 |
| `packages/session/session-persistence/tests/` | 5 |
| `packages/session/session-projection-cache/tests/` | 1 |
| `packages/session/session-projection/tests/` | 1 |
| `packages/session/session-stats/tests/` | 2 |
| `packages/session/session-telemetry-otel/tests/` | 2 |
| `packages/session/session-telemetry/tests/` | 2 |
| `packages/session/session-title-all-prompts-llm/tests/` | 1 |
| `packages/session/session-title-first-prompt-llm/tests/` | 3 |
| `packages/session/session-title-llm/tests/` | 1 |
| `packages/session/session-title/tests/` | 7 |
| `packages/settings/settings-file/tests/` | 5 |
| `packages/settings/settings/tests/` | 4 |
| `packages/shell/bash-local/tests/` | 2 |
| `packages/shell/bash-sandbox/tests/` | 5 |
| `packages/shell/pwsh-local/tests/` | 2 |
| `packages/shell/pwsh-sandbox/tests/` | 2 |
| `packages/shell/shell-env/tests/` | 1 |
| `packages/shell/shell/tests/` | 2 |
| `packages/shell/tool-bash-persistent/tests/` | 2 |
| `packages/shell/tool-bash/tests/` | 2 |
| `packages/shell/tool-pwsh/tests/` | 3 |
| `packages/skill/skill-badge/tests/` | 1 |
| `packages/skill/skill-filesystem/tests/` | 2 |
| `packages/skill/skill/tests/` | 1 |
| `packages/skill/tool-skill/tests/` | 1 |
| `packages/spill/spill-local/tests/` | 1 |
| `packages/spill/spill-policy/tests/` | 1 |
| `packages/spill/spill/tests/` | 1 |
| `packages/storage/storage-domain/tests/` | 3 |
| `packages/storage/storage-json/tests/` | 1 |
| `packages/storage/storage-sqlite/tests/` | 2 |
| `packages/storage/storage/tests/` | 2 |
| `packages/subagent/subagent-acp/tests/` | 4 |
| `packages/subagent/subagent-claude-code/tests/` | 5 |
| `packages/subagent/subagent-codex/tests/` | 6 |
| `packages/subagent/subagent-dsh-sdk/tests/` | 2 |
| `packages/subagent/subagent-fork-in-process/tests/` | 2 |
| `packages/subagent/subagent-in-process-driver/tests/` | 7 |
| `packages/subagent/subagent-spawn-in-process/tests/` | 3 |
| `packages/subagent/subagent/tests/` | 10 |
| `packages/subagent/tool-subagent-control/tests/` | 3 |
| `packages/subagent/tool-subagent-report/tests/` | 1 |
| `packages/subagent/tool-subagent/tests/` | 3 |
| `packages/subprocess/subprocess-local/tests/` | 7 |
| `packages/subprocess/subprocess/tests/` | 1 |
| `packages/terminal/terminal-bash/tests/` | 5 |
| `packages/terminal/terminal/tests/` | 1 |
| `packages/terminal/tool-terminal/tests/` | 3 |
| `packages/test-support/acp-snapshot/tests/` | 52 |
| `packages/test-support/agent-loop-testkit/tests/` | 1 |
| `packages/test-support/client-runtime/tests/` | 4 |
| `packages/test-support/llm-mock-server/tests/` | 3 |
| `packages/test-support/llm-replay/tests/` | 1 |
| `packages/test-support/loader-smoke/tests/` | 6 |
| `packages/todo/tool-todo/tests/` | 5 |
| `packages/typert/generator/tests/` | 40 |
| `packages/typert/loader/tests/` | 1 |
| `packages/typert/protocol/tests/` | 2 |
| `packages/typert/registry/tests/` | 1 |
| `packages/util/atomic-write/tests/` | 2 |
| `packages/util/home-paths/tests/` | 1 |
| `packages/util/launch-environment/tests/` | 1 |
| `packages/util/native-command/tests/` | 1 |
| `packages/util/output-retention/tests/` | 1 |
| `packages/util/timeout/tests/` | 1 |
| `packages/web/tool-web/tests/` | 4 |
| `packages/web/web-fetch-http/tests/` | 1 |
| `packages/web/web-search-deepseek/tests/` | 4 |
| `packages/web/web-search-exa/tests/` | 2 |
| `packages/web/web-search-perplexity/tests/` | 2 |
| `packages/web/web/tests/` | 1 |
| `packages/workflow/tool-ralph/tests/` | 2 |
| `packages/workflow/tool-workflow/tests/` | 2 |
| `packages/workflow/workflow-worker-thread/tests/` | 8 |
| `packages/workflow/workflow/tests/` | 2 |
| `packages/workspace/workspace/tests/` | 2 |
| `python/sdk/tests/` | 7 |
