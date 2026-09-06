---
title: 工作计划索引
summary: dev_docs 计划目录的用途、生命周期与命名约定，以及它与仓库 Agent Note 体系的分工。
keywords: plans | active | done | archive | agent-note | 工作计划
scope: dev_docs/plans 的组织规则与索引
related_files: .agents/notes/README.md | dev_docs/rules/combined/AI_RULES.md
dependencies: .agents/notes/README.md
verified_at: 2026-09-06
---

# 工作计划索引

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**。本目录存放**本 fork 内的中文工作计划**，是过程产物，不是事实的归属地。
>
> **与 `docs/` 冲突时一律以 `docs/` 为准。**

## 与 `.agents/notes/` 的分工（先读这一条）

仓库已有一套决策记录体系：[`.agents/notes/`](../../.agents/notes/README.md)，1,700 余篇，且 [`AGENTS.md`](../../AGENTS.md#conventions) 规定**非平凡改动必须在同一 PR 内附 Agent Note**。本目录**不替代也不复制**它。

| | `.agents/notes/` | `dev_docs/plans/` |
| --- | --- | --- |
| 归属 | 上游仓库，随 PR 提交 | 本 fork 内部，通常不提交上游 |
| 时机 | 决策**已作出**，记录它和它的理由 | 决策**尚未作出**，或工作跨多步需要编排 |
| 语言 | 英文 + 中文配对 | 中文 |
| 强制性 | 非平凡改动强制 | 自愿 |

判据：**一份计划一旦落地为非平凡的仓库改动，仍然需要一篇 Agent Note。** 计划记录"打算怎么做"，Agent Note 记录"做了什么、为什么这样做"。不要把计划直接当 Agent Note 交上去——两者的读者、语言与保留期都不同。

## 生命周期

```
active/  →  done/  →  archive/
 进行中      已完成     已失效
```

- [`active/`](active/index.md) — 正在执行的计划。同一时间保持少量，多了说明没有真正在推进。
- [`done/`](done/index.md) — 已执行完毕、结论仍然成立的计划。
- [`archive/`](archive/index.md) — 被放弃、被取代，或前提已不再成立的计划。**只读**。

流转是单向的。计划完成后移入 `done/`，不要原地改状态字段——`dev_docs` 遵循当前态叙述，不写变更历史（见 [`AI_RULES.md`](../rules/combined/AI_RULES.md) §6）。

## 命名约定

```
YYYY-MM-DD-<kebab-case-主题>.md
```

日期用**创建日**，移动目录时不改名，这样文件名本身就是时间线。

## 一份计划应该包含什么

最小集合，不要模板化填空：

1. **目标** — 一句话说清做完之后什么会变得不同。
2. **约束** — 哪些 [`AI_RULES.md`](../rules/combined/AI_RULES.md) 条目适用；这次改动会碰到哪些门禁。
3. **步骤** — 可独立验证的分步，每步写明如何确认它成立。
4. **未决问题** — 需要人裁定的分叉点，以及在此之前可以先做什么。
5. **验收** — 完成的判据，包括要跑哪些检查。

如果一份计划写不出第 3 项的验证方式，说明它还不够具体，不要开始执行。

## 当前计划

_（暂无。新增计划时在此登记一行：`- [标题](active/YYYY-MM-DD-主题.md) — 一句话`）_

## 延伸阅读

- [`AI_Coding_Context.md`](../AI_Coding_Context.md) — dev_docs 总入口
- [`AI_RULES.md`](../rules/combined/AI_RULES.md) — 动手前必看的硬规则
- [`quality_gates.md`](../quality_gates.md) — 计划中"要跑哪些检查"的反查表
