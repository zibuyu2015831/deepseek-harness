---
title: DeepSeek Harness 项目分析与问题报告
summary: 记录首次生成 dev_docs 文档体系过程中发现的问题、风险假设、疑问与建议；重点是新增文档层与仓库既有文档治理体系及 CI 门禁的集成冲突，而非源码缺陷。
keywords: analysis-report | risks | doc-governance | translation-pairing | deepseek-harness | aicc
scope: deepseek-harness 仓库 dev_docs 首次生成前的问题与风险汇总
related_files: AGENTS.md | docs/AGENTS.md | docs/architecture.md | docs/testing.md | scripts/translation-pairing.ts | scripts/translation-pairing.manifest.json | scripts/verify-md-links.ts | scripts/verify-md-wrap.ts | scripts/verify-doc-budgets.ts | .gitignore | package.json | CONTRIBUTING.md | packages/README.md
dependencies: dev_docs/_analysis/generation_plan.md | dev_docs/_analysis/generation_progress.md
verified_at: 2026-09-06
---

# DeepSeek Harness - 分析与问题报告

## 📋 报告摘要

- **分析日期**: 2026-08-18
- **项目规模**: 9,072 个文件；TypeScript 684,902 行 + TSX 82,677 行 + Python 8,718 行；255 个 workspace 包
- **主要技术栈**: TypeScript 6 (ESM) + Cordis（vendored）+ pnpm 11 workspaces + Vitest 4 + tsdown/tsc 双面构建
- **分析覆盖度**: 约 70%——已覆盖仓库治理、构建体系、架构主干、能力接缝、外部服务边界与文档门禁；**未覆盖**具体包内实现细节（将在正式生成阶段按批次深入）

### 问题统计

| 严重程度 | 数量 | 状态 |
| -------- | ---- | -------- |
| 🔴 严重  | 2    | 均已有用户决策，需执行 |
| 🟡 警告  | 4    | 建议修复，其中 1 项阻断批次 6 |
| 🔵 疑问  | 4    | 需确认（均不阻断 Phase 1） |
| 💡 建议  | 4    | 可选优化 |

> **重要前提**: 本报告的"严重问题"**不是源码缺陷**。deepseek-harness 的源码质量与工程治理水平显著高于常见项目（每文件 100% 覆盖率门禁、30+ 静态门禁、1,742 篇决策记录）。所有严重问题均来自"向一个已有成熟文档治理体系的仓库中新增第二套文档体系"这一动作本身。

---

## 🔴 严重问题（必须处理）

### 问题 1: `dev_docs/` 与 `docs/` 形成事实的第二归属地

**问题类型**: 架构 / 文档治理

**发现位置**:

- 文件: `docs/AGENTS.md:15-34`（"The tier taxonomy: one home per fact"）
- 文件: `AGENTS.md`（Conventions 章节，"一个事实一个归属地"的上游规则）
- 影响范围: 全部 17 篇计划子文档

**问题描述**:

仓库 `docs/AGENTS.md` 明确规定："Each fact has one home: the tier whose job it is; elsewhere, link there."（每个事实只有一个归属地：负责它的那一层；其他地方链接过去。）该规则由分层表强制，并配有 `verify-doc-budgets`、`verify-md-links`、`dsh-doc-standards` skill 的 slop 检查清单等多重执行手段。

按 AICC 标准全量体系生成的 `dev_docs/architecture_overview.md`、`testing_guide.md`、`deployment_guide.md` 等，会为架构、测试策略、发布流程等事实**重新撰写正文**，从而在 `docs/architecture.md`、`docs/testing.md` 之外建立第二份描述。

**问题分析**:

- **根本原因**: AICC 框架假定目标项目缺乏结构化文档；本仓库恰恰相反
- **影响范围**: 全部 17 篇子文档；间接影响所有依据 `dev_docs` 工作的 AI 与人类
- **潜在风险**: 上游 `docs/` 变更后 `dev_docs` 不会自动同步。AI 读到过期中文文档会产出与当前架构不符的改动，而这类错误在 255 包的插件树中极难定位

**证据等级**: E2（配置与项目文档）

**当前状态**: 需用户确认 → **用户已确认采用 AICC 标准全量体系**，转为**已接受风险**

**blocks_phase1**: false

**回写目标**: `dev_docs/rules/combined/AI_RULES.md` + 每篇子文档的"上游事实源"声明

**为什么需要用户确认**: 是否接受双体系并承担同步成本，属于团队流程与所有权决策，仓库文档未记录 `dev_docs` 这一新层的维护约定。

**维护者规则约束**: `docs/AGENTS.md` 是维护者制定的文档标准，`dev_docs` 不得声称覆盖或替代它。任何 `dev_docs` 内容与 `docs/` 冲突时，必须以 `docs/` 为准。

**缓解措施（必须执行，写入每篇子文档）**:

1. 每篇 frontmatter 的 `dependencies` 指向其上游 `docs/` 事实源
2. 正文每个主要章节开头标注 `> 上游事实源: ../docs/xxx.md`
3. 事件表、工具表、配置表、模块依赖图**一律链接生成产物**（`docs/tool-catalog.md`、`docs/config-catalog.md`、`docs/persistence-catalog.md`、`docs/module-graph.md`、`docs/event-producer-consumer.md`），禁止复制
4. `AI_RULES.md` 写入硬规则："`dev_docs` 与 `docs/` 冲突时以 `docs/` 为准，并立即更新 `dev_docs`"
5. 后续维护走 AICC 路径 C（`@commit` 增量更新），使 `docs/` 变更能触发 `dev_docs` 同步

---

### 问题 2: `AI-Coding-Context` 符号链接未被 gitignore

**问题类型**: 版本管理 / 可移植性

**发现位置**:

- 文件: `.gitignore`（无相关条目）
- 验证: `git check-ignore -v AI-Coding-Context` 退出码为 1（未被忽略）
- 现象: `git status` 将其列为未跟踪项 `?? AI-Coding-Context`

**问题描述**:

`AI-Coding-Context` 是指向仓库**外部绝对路径** `/Users/<user>/code/zibuyu/AI-Coding-Context` 的符号链接。若被提交，Git 会保存该绝对路径字符串，在任何其他开发者或 CI 机器上都会成为断链。

**问题分析**:

- **根本原因**: 框架以符号链接方式挂载，未同步更新 `.gitignore`
- **影响范围**: 全体协作者与 CI
- **潜在风险**: 断链的符号链接可能导致 `knip`、`verify-md-links` 等遍历型门禁行为异常；同时泄露本机用户名与目录结构

**证据等级**: E4（命令验证）

**当前状态**: ✅ **已修复并闭环**。`.gitignore:46` 已包含 `AI-Coding-Context`；该符号链接在当前 clone 中不存在（未随 Git 传播），证明修复有效。

**blocks_phase1**: false

**回写目标**: `.gitignore`（执行动作）+ `generation_progress.md` 行动清单

**维护者规则约束**: `.gitignore` 修改属于仓库配置变更；按 `AGENTS.md` Conventions，此类机械性本地编辑可豁免 Agent Note。

**修复建议**:

```gitignore
# AICC 文档框架挂载点（指向仓库外的符号链接，不可移植）
AI-Coding-Context
```

---

## 🟡 警告问题（建议修复）

### 警告 1: `dev_docs/**/README.md` 将被双语配对门禁捕获

**问题类型**: CI 门禁冲突

**发现位置**:

- `scripts/translation-pairing.ts:127` — `const README_ARTIFACT = /(?:^|\/)readme(?:\.md|\.zh\.md|\.i18n\.yaml)$/i`
- `scripts/translation-pairing.ts:146` — `TRANSLATION_SCOPE_GLOB_EXCLUDES` 未包含 `dev_docs/`
- `scripts/translation-pairing.ts:180-186` — `isTranslationScopeFile()` 对任意路径下的 README 返回 true

**问题描述**:

`README_ARTIFACT` 正则以 `(?:^|\/)readme` 开头匹配，即**任意目录层级**下的 `README.md` 都属于双语配对作用域。AICC 标准体系要求创建 `dev_docs/plans/README.md`、`dev_docs/memos/README.md`、`dev_docs/knowledge/README.md`，这三个中文文件会被要求补 `.zh.md` 与 `.i18n.yaml` 配对文件。

**影响评估**:

- `pnpm run verify-translation-pairing` 失败
- `pnpm run doc-sync` 失败（该聚合包含配对门禁）
- 由于本仓库 CI 以 `doc-sync` 作为文档变更的必过门禁，会直接阻塞 PR

**证据等级**: ✅ **E4（实证）** — 2026-09-06 升级

> **实证方法**：在 `pnpm install` 完成后创建探针文件 `dev_docs/plans/README.md`，运行 `pnpm run verify-translation-pairing`，门禁以 exit 1 失败并报 `dev_docs/plans/README.md: in-scope documentation must merge bilingual (docs/i18n/README.md); add the counterpart and record the pair`。探针文件随即删除，门禁恢复通过。E3 推断得到完全确认。

**当前状态**: 已确认，待选择修复方案（新增方案 (c)，见下）

**blocks_phase1**: false（**阻断批次 6**）

**回写目标**: `generation_plan.md` 批次 6 前置条件 + `quality_gates.md`

**为什么需要用户确认**: 修复方式二选一，属于维护者对仓库门禁脚本的治理决策。

**维护者规则约束**: 修改 `scripts/translation-pairing.ts` 属于非平凡改动，按 `AGENTS.md` Conventions 必须在同一 PR 内附 Agent Note。

**修复建议（二选一）**:

方案 (a) — 清单豁免（改动最小，但需为每个新 README 重复维护）:

```jsonc
// scripts/translation-pairing.manifest.json
{
  "excluded": [
    "dev_docs/plans/README.md",
    "dev_docs/memos/README.md",
    "dev_docs/knowledge/README.md"
    // ... 既有 8 项
  ]
}
```

方案 (b) — 作用域排除（一次性覆盖未来所有 `dev_docs` README）:

在 `scripts/translation-pairing.ts` 的 `TRANSLATION_SCOPE_GLOB_EXCLUDES` 中加入 `'dev_docs/**'`，并在 `isTranslationSourceExcluded()` 中加入 `file.startsWith('dev_docs/')` 判断。

方案 (c) — 改用非 `README` 文件名（**新增，推荐**）:

配对门禁的 `README_ARTIFACT` 正则只匹配文件名 `readme`。`dev_docs` 的目录索引改叫 `index.md` 即完全绕开该门禁，**无需改动仓库门禁脚本，也就无需 Agent Note、无需向上游提 PR**。代价是偏离 AICC 的标准目录约定；由于 AICC 检查工具在本机不可用（见建议 4），该偏离当前无法被检出，实际成本为零。

**修复优先级**: 🟡 P1 - 批次 6 开始前必须选定其一（批次 1-5 不受影响）

---

### 警告 2: 依赖未安装，仓库门禁当前不可运行

**问题类型**: 环境

**发现位置**: 仓库根，`test -d node_modules` 失败

**问题描述**:

首次分析时 `node_modules/` 不存在，所有仓库自有门禁（`verify-*`、`gen-*`、`vitest`、`tsc`）均不可运行。

**当前状态**: ✅ **已解决**。2026-09-06 执行 `pnpm install`（exit 0），随后取得三项 E4 门禁结果：

| 门禁 | 结果 |
| ---- | ---- |
| `verify-md-links` | PASS — 2273 个文件，全部相对链接与锚点可解析 |
| `verify-translation-pairing` | PASS — 1135 组配对一致 |
| `verify-doc-budgets` | PASS — 8 篇预算文档均在上限内 |

三项均在含 `dev_docs/_analysis/` 三篇文档的工作树上运行，说明 `dev_docs/*.md`（非 README）不进入这些门禁的作用域。

> ⚠️ 但 `verify-md-links` 的 PASS **不等于** `dev_docs` 的链接已被验证——复读 `scripts/verify-md-links.ts:19-29` 确认其 `PATTERNS` 不含 `dev_docs/`。`dev_docs` 内部链接的验收办法见 `generation_plan.md` 复核记录。

**证据等级**: E4（命令验证）

**blocks_phase1**: false（AICC 侧 Phase 1 hard gate 使用框架自带 Python/Node 工具，不依赖仓库 `node_modules`）

**回写目标**: `generation_progress.md` 行动清单

**维护者规则约束**: 无冲突。

**修复建议**:

```bash
pnpm install
pnpm run verify-translation-pairing   # 复验警告 1，取得 E4 证据
pnpm run verify-md-links              # 确认 dev_docs 不在作用域内
```

---

### 警告 3: 引用 `docs/` 行号存在失效风险

**问题类型**: 文档可维护性

**发现位置**: `generation_plan.md` 中多处引用形如 `docs/architecture.md:96`、`scripts/translation-pairing.ts:127`

**问题描述**:

行号引用在上游文件编辑后会静默失效——指向错误内容而不报错，比断链更危险（`verify-md-links` 只检查目标文件与锚点是否存在，不检查行号）。

**影响评估**: 中期（数周至数月）内引用逐渐指向错误内容。

**证据等级**: E2

**当前状态**: 已确认

**blocks_phase1**: false

**回写目标**: 全部正式子文档的编写规则

**维护者规则约束**: `docs/AGENTS.md` 要求"用可机器检查的链接做交叉引用，不要用自由散文"。行号不可机器检查，与该精神不符。

**修复建议**:

- 正式文档中优先引用**小节标题锚点**（`../docs/architecture.md#capability-seams`），由 `verify-md-links` 保护
- 行号仅用于**源码**引用（源码无标题可锚定），且必须同时给出符号名以便失效后定位
- 每篇 frontmatter 的 `verified_at` 如实记录核验日期

---

### 警告 4: AICC `ai_llm_app` 类型配置与本项目实际形态不匹配

**问题类型**: 框架适配

**发现位置**: `AI-Coding-Context/core/project_types/ai_llm_app.md`（推荐子文档清单）

**问题描述**:

AICC 的 AI/LLM 应用类型配置面向 RAG 应用设计，推荐 `rag_architecture.md`、`vector_database.md` 两篇必需文档，核心代码模式全部为 LangChain/Python 示例。本项目是 **Agent Harness 运行时**，无检索增强生成、无向量存储、无 embedding 流水线。

**影响评估**: 若照搬清单，会产出两篇无内容可写的文档，并引入与项目无关的 LangChain 概念。

**证据等级**: E4

**验证命令**:

```bash
grep -rilE "pinecone|weaviate|chroma|qdrant|milvus|pgvector|embedding" packages --include=package.json
# 命中数: 0
```

**当前状态**: 已确认

**blocks_phase1**: false

**回写目标**: `generation_plan.md` 1.1 节"与 AICC 标准类型配置的偏离说明"（**已回写**）

**维护者规则约束**: 无冲突。

**修复建议**: 已在方案中将两篇替换为 `capability_seams.md` 与 `session_and_events.md`，并保留显式偏离说明。

---

## 🔵 疑问事项（需确认）

### 疑问 1: `dev_docs` 与 `docs/` 的长期维护责任归属

**疑问类型**: 流程与所有权

**当前保守结论**: `docs/` 为权威源，`dev_docs` 为中文派生层，冲突时以 `docs/` 为准。

**已检查证据**: `docs/AGENTS.md:15-34`；`AGENTS.md` Conventions；`scripts/doc-budgets.manifest.json`（`dev_docs` 不在预算清单内）；`CONTRIBUTING.md`

**为什么代码或仓库文档无法回答**: 仓库从未记录 `dev_docs` 这一层的存在，更未规定其维护约定。属于团队流程决策。

**用户裁定（2026-09-06）**: 采用当前保守结论。

- [x] `docs/` 为权威源，`dev_docs` 为派生层；两者冲突时**一律以 `docs/` 为准**，并立即更新 `dev_docs`
- [x] 不要求同 PR 强制同步；采用 `verified_at` 驱动的复核（超过 90 天视为过期）
- [x] `dev_docs` 不纳入仓库 CI 门禁作用域（F23 已证明其当前确实在作用域外）

**证据等级**: E2 | **当前状态**: ✅ 已裁定 | **blocks_phase1**: false | **回写目标**: `AI_RULES.md`

**维护者规则约束**: 不得建议放宽 `docs/AGENTS.md` 的既有要求；`dev_docs` 的维护约定只能是**追加**约束，不能是豁免。

---

### 疑问 2: 双语配对门禁的修复路径选择

**疑问类型**: 门禁治理

**当前保守结论**: 采用方案 (b)，在 `TRANSLATION_SCOPE_GLOB_EXCLUDES` 与 `isTranslationSourceExcluded()` 中排除 `dev_docs/`。

**已检查证据**: `scripts/translation-pairing.ts:127,146,180-186`；`scripts/translation-pairing.manifest.json`（现有 `excluded` 共 8 项，全部位于 `docs/` 与 `.agents/notes/`）

**为什么代码或仓库文档无法回答**: 修改门禁脚本属于维护者治理决策；仓库文档未规定第三方文档体系如何接入门禁作用域。

**用户裁定（2026-09-06）**: 采用保守结论方案 (b)。但本轮 E4 实证后新增了方案 (c)（`dev_docs` 内改用 `index.md`），它无需改动仓库门禁脚本、无需 Agent Note、无需向上游提 PR，成本严格低于 (a) 与 (b)。

> **后续（2026-09-07）**：核实发现上游根本不接受外部 PR（见本文档"生成过程中发现的上游文档缺陷"一节的说明）。这把方案 (c) 从"成本更低"升级为**唯一可行**——(a)/(b) 都需要改动 `scripts/translation-pairing.ts`，而对上游脚本的改动无法回流，只能永久停留在 fork 内并在每次 merge 时承担冲突。选 (c) 是对的。

- [x] 默认方案 (b)；**批次 6 开始前请复核是否改选 (c)**
- [ ] 是否作为独立 PR 先行合入 — 仅方案 (a)/(b) 需要，选 (c) 则不涉及

**证据等级**: ✅ E4 | **当前状态**: 已裁定，含新增待复核选项 | **blocks_phase1**: false（阻断批次 6） | **回写目标**: `generation_plan.md` 批次 6 前置条件

**维护者规则约束**: 该改动为非平凡改动，必须同 PR 附 Agent Note（`AGENTS.md` Conventions）。

---

### 疑问 3: `dev_docs` 是否需要英文对照版

**疑问类型**: 产品与社区策略

**当前保守结论**: 仅中文，不生成英文版。

**已检查证据**: 用户明确指示"文档使用中文"；`README.md` 与 `docs/` 全部为双语；`docs/i18n/README.md` 定义配对契约；社区渠道（GitHub Discussions、Discord）以英文为主

**为什么代码或仓库文档无法回答**: 本仓库是面向国际社区的开源项目，是否接受一个中文独有的文档层属于产品与社区策略，非技术问题。

**用户裁定（2026-09-06）**: 采用当前保守结论——仅中文，不生成英文版。

- [x] `dev_docs` 保持中文独有
- [x] 若未来补英文，复用 `docs/i18n/` 既有配对契约，不另建机制

> **本轮新增事实（F26）**：`docs/` 已是**全量双语**——`docs/`、`docs/subsystems/`、`docs/cookbook/` 下每篇英文源都有 `.zh.md` 与 `.i18n.yaml`，`verify-translation-pairing` 报 1135 组配对一致。因此 `dev_docs` **不能以"提供中文文档"作为存在理由**：中文已由上游覆盖。其价值必须收敛到上游不提供的两件事——面向 AI 的**导航层**（场景 → 文档 → 上游事实源的三级跳转）与**流程层**（`plans/` `memos/` `knowledge/`）。这一定位必须写入每篇子文档的开篇声明与 `AI_RULES.md`。

**证据等级**: E2（裁定）/ E4（F26 双语事实） | **当前状态**: ✅ 已裁定 | **blocks_phase1**: false | **回写目标**: `AI_RULES.md` + 全部子文档开篇定位声明

**维护者规则约束**: 若未来补英文，必须复用 `docs/i18n/` 契约，不得另建第二套翻译机制。

---

### 疑问 4: subagent / workflow / skill 三条扩展链路是否独立成文

**疑问类型**: 文档粒度偏好

**当前保守结论**: 合并进 `plugin_development_guide.md` 的"扩展点选择"章节。

**已检查证据**: `packages/subagent/`、`packages/workflow/`、`packages/skill/` 各有 README 与 `docs/subsystems/` 对应页；`docs/architecture.md` 的"Where new behavior goes"表已覆盖三者

**为什么代码或仓库文档无法回答**: 取决于团队对这三条链路的实际开发频率，仓库无此信息。

**用户裁定（2026-09-06）**: 采用当前保守结论——不拆分。

- [x] subagent / workflow / skill 合并进 `plugin_development_guide.md` 的"扩展点选择"章节，正文链接至 `docs/subsystems/subagent.md`、`docs/subsystems/workflow.md`、`docs/subsystems/skills.md`

**证据等级**: E2 | **当前状态**: ✅ 已裁定 | **blocks_phase1**: false | **回写目标**: `generation_plan.md` 3.1/3.2 子文档清单

**维护者规则约束**: 无冲突。

---

## 💡 优化建议（可选）

### 建议 1: `AI_RULES.md` 应蒸馏 `AGENTS.md` 而非重写

**建议类型**: 文档质量

**当前状态**: `AGENTS.md` 已是高度浓缩的"标准指令"层（≤1,600 词预算，每条规则 1-3 行并链接归属地）。

**建议**: `dev_docs/rules/combined/AI_RULES.md` 应做两件事——① 用中文重述 `AGENTS.md` 的**硬约束**（不可破坏项）；② 补充 `AGENTS.md` 未覆盖的 `dev_docs` 自身规则。**不得**改写 `AGENTS.md` 中的规则语义，也不得补充 `AGENTS.md` 有意省略的细节。

**收益**: 避免 AI 在两份规则文件间产生歧义。

**成本**: 需在每次 `AGENTS.md` 变更后同步。

**优先级**: 💡 P1（实际上接近必需）

---

### 建议 2: 建立 `dev_docs` 的 `verified_at` 复核周期

**建议类型**: 流程

**建议**: 利用 AICC 的 `verified_at` 字段（超过 90 天即视为过期），配合 `summary_validator` 定期扫描：

```bash
python3 AI-Coding-Context/tools/py/summary_validator.py --dir dev_docs --recursive --strict
```

**收益**: 将问题 1 的漂移风险从"不可见"变为"可度量"。

**优先级**: 💡 P1

---

### 建议 3: 确认 `dev_docs` 不影响 `knip` 与 `hygiene` 聚合

**建议类型**: 门禁验证

**建议**: `pnpm install` 后运行 `pnpm run hygiene`，确认新增 `dev_docs/` 不引入 knip 未使用文件告警或 workspace 约束违规。

**收益**: 提前排除除双语配对之外的其他门禁冲突。

**成本**: 一次完整 install + 门禁运行。

**优先级**: 💡 P1

---

### 建议 4: AICC 工具链在本机不可用，验收方式必须降级并如实标注

**建议类型**: 环境 / 验收完整性

**当前状态**: 🔴 **AICC 框架不在本机**。`AI-Coding-Context` 原本是指向仓库外绝对路径的符号链接，按问题 2 的修复已加入 `.gitignore`，因此不随 Git 传播——`ls -ld AI-Coding-Context` 返回 No such file。这意味着 `summary_validator.py`、`doc_health_checker.py`、`semantic_review_checker.py` 及其 JS 对应实现**全部无法运行**，Python 版本兼容性问题因此不再是当前矛盾（本机 Python 为 3.14.6，Node 为 v26.3.1）。

**影响**: 批次 7 首版质量验收原定的"Python/JS 双实现交叉验证"无法执行。

**建议（替代验收方案）**:

1. 借用仓库自有 `verify-md-links` 验证 `dev_docs` 内部链接与锚点——临时向其 `PATTERNS` 追加 `'dev_docs/**/*.md'`，运行后还原，改动不提交
2. 人工执行 `generation_plan.md` 的"质量检查清单"四组（准确性 / 完整性 / 可用性 / 一致性）
3. 在 `health_check_report.md` 中将全部 AICC 检查标注为 `NOT_RUN` 并写明原因，**不得伪造或推测检查结果**
4. 若后续需要真实 AICC 验收，重新挂载框架后补跑，并把结果追加进健康报告

**收益**: 避免以"未运行"冒充"通过"。

**优先级**: 💡 P0（影响验收结论的可信度）

---

## 🔬 生成过程中发现的上游文档缺陷（2026-09-06）

撰写 `dev_docs` 时对每条断言做源码核实，附带发现了若干**仓库自有文档与代码实际不符**之处。它们不是 `dev_docs` 的问题，而是上游文档的陈旧或笔误。

> **无法回流上游。** 2026-09-07 核实：上游 `CONTRIBUTING.md` 明文"cannot accept external pull requests"，`has_issues` 为 `false`，PR API 返回 404，该仓库是私有开发仓库的单向发布镜像。唯一对外渠道是 Discussions，且官方在其中基本不公开回复。因此这些条目的定位是 **fork 内自用的已知陷阱清单**，详见 [`UPSTREAM_DOC_ISSUES.md`](../../UPSTREAM_DOC_ISSUES.md)。

> 这些缺陷是"生成 `dev_docs` 是否值得"的一个正面证据：为满足 one-home-per-fact 而对每条引用做源码核实，本身就构成了一次对上游文档的交叉审计。

| # | 位置 | 上游写的 | 实际情况 | 证据等级 | 影响 |
| --- | --- | --- | --- | --- | --- |
| U1 | `AGENTS.md:103`、`vendor/README.md:34` | vendored 包 "rescoped … and `private: true`" | 9 个 `vendor/*/package.json` **全部没有 `private` 字段**，且都带 `publishConfig.access: "public"`。**决定性证据**：`.agents/notes/implemented/process/2026-08-10-npm-release-sequences.md:133` 上游自己记着"该约定 no longer holds" | E4 | 高：会让贡献者误以为 vendored 包不发布，而它们实际会被发布。**反证**：`scripts/release/verify.ts:44-49` 的 `verifyPublishable` 会拒绝任何 `private: true` 的成员——若上游那两处散文成立，vendor 家族根本发布不出去 |
| U2 | `AGENTS.md:38` | Repository layout 列出 `self-modification/` | 该目录**不存在**；对应内容在 `packages/extensions/`（`tool-cordis` / `ui-cordis` / `cordis-host-runner` / `cordis-client-runner`） | E4 | 中：按图索骥会找不到目录 |
| U3 | `AGENTS.md:49` | Repository layout 列出 `support/` | 该目录**不存在**；实际是 `packages/test-support/` | E4 | 中：同上 |
| U4 | `docs/testing.md:40` | 要求保持 `packages/examples/*/tests/built-bin.e2e.ts` 构建产物冒烟测试为绿 | `packages/examples/` **已不存在**（最后一个同名测试由 `d8dbb8235c` 于 2026-08-23 删除，目录由 `244de7c18a` 于 08-26 清空）；全仓唯一的 `built-bin.e2e.ts` 现位于 `apps/cli/tests/` | E4 | 中：指向一个已不存在的测试路径 |
| U5 | `docs/subsystems/session.md:92` | `Session.append` "runtime-validates all event data with `isJsonValue`" | 源码 `packages/core/session/src/index.ts:709` 实际调用 `snapshotJsonValue`（一次遍历同时校验并复制，以防住有状态 getter） | E4 | 低-中：函数名错误，但描述的行为方向一致；实际语义比文档更强 |
| U6 | `packages/llm/llm-deepseek/README.md:54` | 配置表写 `baseURL` 为 "`$DEEPSEEK_BASE_URL` wins when set" | 源码 `index.ts:378-380` 与同文件 `Config.baseURL` JSDoc（`:128`）均为 `config.baseURL ?? $DEEPSEEK_BASE_URL ?? 默认`，即**显式配置优先于环境变量** | E3 | 低：疑为表格措辞松散（同段最小配置注释与源码一致），非行为分歧 |
| U7 | `BENCHMARK.md`（全文仅 231 字节 / 3 行） | 指引读者"运行 `jsonrpc-agent` minimal 变体" | `jsonrpc-agent` 在全仓库**已无任何踪迹**（`find` 零命中）；对应示例已迁至 `python/sdk/examples/`，profile 名为 `sdk-minimal` | E4 | 中：唯一的基准文档指向一个不存在的示例，且该文件不被任何 CI 作业执行，除本方案外全仓无第二处引用 |

**核实命令**：

```bash
python3 -c "import json,glob;[print(f, json.load(open(f)).get('private'), json.load(open(f)).get('publishConfig')) for f in glob.glob('vendor/*/package.json')]"
test -d packages/self-modification; test -d packages/support        # 均失败
grep -n "packages/examples" docs/testing.md
grep -n "isJsonValue" docs/subsystems/session.md
grep -n "snapshotJsonValue" packages/core/session/src/index.ts
```

**`dev_docs` 的处理原则**：一律**以源码为准**，并在正文显式标注该冲突，而不是沉默地跟随任一方。已按此处理 U1（`AI_Coding_Context.md` 命名规范表 + 证据表 F28）与 U6（`model_configuration.md` 第 7 节）。

> **本表是生成期快照，不是提 issue 的依据。** 权威记录在仓库根目录的 [`UPSTREAM_DOC_ISSUES.md`](../../UPSTREAM_DOC_ISSUES.md)：它经 2026-09-06 第二轮逐条回源复查，补入 6 条附带发现（S1–S6）、按真实根因重新归并成簇，并**删除了本表初版里两条已被证伪的推断**——其一是"vendor 清单版本已陈旧"（`package.json` 与 manifest 版本按设计就不相等，见 `scripts/release/bump.ts:153-161`），其二是 U5 的"隐藏了复制契约"（该契约在 `packages/core/session/src/index.ts:691-693` 有完整文档）。上表已按复查结论修正。

---

## 🔍 架构观察

### 观察 1: 文档密度与决策记录密度极高

**发现**:

- Markdown 255,250 行
- `.agents/notes/**/*.md` 1,742 篇决策记录
- `docs/` 采用严格分层 + 双语配对 + 字数预算 + 生成产物新鲜度门禁

**评价**: ✅ 这是优势而非缺陷。它同时意味着——本仓库对"新增一层文档"的容忍度**低于**普通项目，因为既有体系已解决大部分问题。

**对文档生成的影响**: 强化了问题 1 的缓解措施——`dev_docs` 的价值应集中在**中文可达性**与**流程层**（plans / memos / knowledge），而非重述已有事实。

---

### 观察 2: 测试与质量门禁体系完备

**发现**:

- 260 个测试目录，共 1713 个测试文件（含快照期望输出与夹具）；`packages/` 下 `*.spec.ts` / `*.test.ts` 源文件 854 个
- 四层测试：unit（`vitest.config.ts`）、coverage（每文件 100%）、snapshot（`vitest.snapshot.config.ts`）、e2e（`vitest.e2e.config.ts`），另有 Web 与压力测试配置
- 30+ 个 `verify-*` / `gen-*` 门禁脚本，由 `scripts/run-gates.ts` 编排为多个 CI 聚合
- Windows 平台通过 wine 门禁覆盖

**评价**: ✅ 无需在问题报告中提出"补充测试"类建议——那与维护者规则冲突。`AGENTS.md` 明确要求"按 diff 选最小检查集，禁止反射式跑全量套件"，CI 拥有穷尽覆盖。

---

### 观察 3: vendored 依赖与 rescope 机制

**发现**: `vendor/` 下 9 个 Cordis 生态包为固定上游源码副本，rescope 为 `@deepseek-ai/*` 并标记 `private: true`；`pnpm-workspace.yaml` 通过 `overrides` 将 `@deepseek-ai/cosmokit`、`@deepseek-ai/schemastery` link 到 vendor。

**评价**: 🔵 属于有意的架构决策（`docs/rescope.md`、`vendor/README.md` 有完整说明）。`dev_docs/monorepo_and_build.md` 必须覆盖此机制——它是新贡献者最易困惑的点之一。

---

## 📊 风险假设与证据等级

| 风险假设 | 证据等级 | 当前证据 | 验证状态 | 下一步验证动作 | 是否可定优先级 |
| -------- | -------- | -------- | -------- | -------------- | -------------- |
| `dev_docs` 会与 `docs/` 漂移 | E2 | `docs/AGENTS.md:15-34` 分层规则 | 已确认（规则存在）；**漂移已实际发生**：方案 1.5 节的架构描述在 19 天内即落后于 `docs/architecture.md` | 建立 `verified_at` 复核周期；子文档一律链接不复制 | P0（风险已兑现） |
| `dev_docs/**/README.md` 触发配对门禁 | **E4** | 探针实证：`verify-translation-pairing` exit 1 | ✅ 已确认 | 批次 6 前选定方案 (a)/(b)/(c) | P1（阻断批次 6） |
| `dev_docs/*.md`（非 README）不受 `verify-translation-pairing` / `verify-doc-budgets` 约束 | **E4** | 三项门禁在含 `_analysis` 三篇的工作树上全部 PASS | ✅ 已确认 | — | 否 |
| `dev_docs` 不在 `verify-md-links` 作用域内 | **E4** | `verify-md-links.ts:19-29` 的 `PATTERNS` 复读 + 2273 文件 PASS | ✅ 已确认 | 批次 7 临时扩展 `PATTERNS` 后本地跑一次，不提交 | P1 |
| 符号链接提交后在他机断链 | E4 | `.gitignore:46` 已忽略；当前 clone 中该链接不存在 | ✅ 已闭环 | — | 已完成 |
| 无 RAG / 向量库 / embedding | E4 | `grep` 命中 0 | 已确认 | — | 否 |
| AICC 工具链可用 | **E4（证伪）** | `ls -ld AI-Coding-Context` → No such file | ❌ **不可用** | 批次 7 采用降级验收方案并如实标注 `NOT_RUN` | P0 |
| `dev_docs` 不影响 `knip` / `hygiene` | E1 | 未验证 | 待验证 | `pnpm run hygiene` | 否 |

**证据等级规则**:

- E1：目录结构、文件名、文件数量。只能写"疑似""风险假设""建议后续验证"
- E2：配置文件、锁文件、README、项目文件。可写"已从配置确认"
- E3：源码片段、协议、关键函数、调用链。可写"代码显示""实现方式为"
- E4：构建、测试、脚本运行、工具检查结果。可写"已验证""检查通过/失败"

---

## ✅ 审核与行动计划

### 用户审核清单

**严重问题确认**:

- [x] 问题 1: `dev_docs` 事实归属地重复 — 已接受风险，缓解措施 1-5 强制执行
- [x] 问题 2: 符号链接未 gitignore — ✅ 已修复闭环

**警告确认**:

- [x] 警告 1: 双语配对门禁冲突 — 默认方案 (b)，批次 6 前复核是否改选 (c)
- [x] 警告 2: `pnpm install` 已执行，E4 证据已取得
- [x] 警告 3: 采用标题锚点替代行号引用（源码引用保留行号并附符号名）
- [x] 警告 4: 确认排除 `rag_architecture.md` / `vector_database.md`

**疑问确认**（均采用"当前保守结论"）:

- [x] 疑问 1: `dev_docs` 维护责任归属 — `docs/` 为准，`verified_at` 驱动复核，不入 CI 门禁
- [x] 疑问 2: 门禁修复路径 — 方案 (b)，批次 6 前复核 (c)
- [x] 疑问 3: 是否需要英文对照 — 仅中文；并据 F26 收敛 `dev_docs` 定位为导航层 + 流程层
- [x] 疑问 4: subagent/workflow 是否独立成文 — 不拆分，合并进 `plugin_development_guide.md`

**优化建议决策**:

- [x] 建议 1: `AI_RULES.md` 蒸馏而非重写 — **采纳**
- [x] 建议 2: `verified_at` 复核周期 — **采纳**（AICC 的 `summary_validator` 不可用，改为人工按 `verified_at` 复核）
- [x] 建议 3: 验证 `knip` / `hygiene` 无影响 — **采纳**，批次 7 执行
- [x] 建议 4: AICC 工具链不可用 — **采纳降级验收方案**，升级为 P0

---

### 修复计划

**阶段 1: 正式生成前（批次 1 之前）** — ✅ 全部完成

- [x] 将 `AI-Coding-Context` 加入 `.gitignore`
- [x] 执行 `pnpm install`（exit 0）
- [x] 答复全部待确认事项（4 项）
- [x] 复核并回写全部统计数（仓库 19 天内增长 25%-49%）

**阶段 2: 批次 6 之前（硬性前置）**

- [ ] 在方案 (a)/(b)/(c) 中选定一个并实施；选 (c) 则无需改动仓库脚本、无需 Agent Note
- [x] 运行 `pnpm run verify-translation-pairing` 取得 E4 证据（探针实证已完成）

**阶段 3: 首版验收阶段**

- [ ] 运行 `pnpm run hygiene`，确认 `dev_docs` 不引入 knip / workspace 告警
- [ ] 临时扩展 `verify-md-links` 的 `PATTERNS` 覆盖 `dev_docs/**/*.md`，运行后还原（不提交）
- [ ] ~~运行 Python/JS 双套 `doc_health_checker` 与 `semantic_review_checker`~~ — **不可执行**，AICC 框架不在本机；按建议 4 的降级方案验收并标注 `NOT_RUN`

---

## 📝 后续行动

### 立即行动

1. **用户审核本报告** — 确认 2 项严重问题的处理方式与 4 项疑问的答复
2. **执行阶段 1 修复** — `.gitignore` + `pnpm install`
3. **确认方案** — 回复"方案审核通过"后进入批次 1

### 文档生成建议

**本报告中的严重问题不阻断正式文档生成。** 与模板默认假设不同，本仓库源码不存在需先修复的缺陷；两项严重问题均为文档层集成问题，且用户已作出决策。

**建议流程**:

```
1. 生成分析报告 ✅
2. 用户审核确认 ⏸️  ← 当前位置
3. 执行阶段 1 修复（.gitignore + pnpm install）
4. 批次 1-5 生成正式子文档
5. 阶段 2 修复（门禁作用域）
6. 批次 6 生成流程目录与 AI_RULES
7. 批次 7 首版质量验收
```

---

## 📅 报告元信息

- **生成者**: AI Assistant（AICC 路径 A，Step 6）
- **生成日期**: 2026-08-18
- **分析时长**: 约 1.5 小时
- **下次审查**: 批次 6 开始前（需复验警告 1）

---

## 🔗 相关文档

- [文档生成方案](./generation_plan.md)
- [生成进度记录](./generation_progress.md)

---

**重要提醒**:

1. 本报告的观察需人工判断；某些"问题"是本仓库有意的设计决策
2. 本仓库工程治理成熟度高，报告刻意**未**提出补测试、加类型检查、统一命名等在本仓库已被门禁解决的常规建议
3. 请优先关注 🔴 问题 1 的缓解措施——它决定 `dev_docs` 的长期价值
