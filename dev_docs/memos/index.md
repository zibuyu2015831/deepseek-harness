---
title: 备忘索引
summary: 临时性发现、待查问题与上下文交接记录的存放地，以及它们向 knowledge/ 沉淀的判据。
keywords: memos | 备忘 | 待查 | 交接 | 上下文
scope: dev_docs/memos 的组织规则与索引
related_files: dev_docs/knowledge/index.md | dev_docs/plans/index.md
dependencies: dev_docs/plans/index.md
verified_at: 2026-09-06
---

# 备忘索引

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**。本目录存放**短周期、未定型**的记录，是过程产物。
>
> **与 `docs/` 冲突时一律以 `docs/` 为准。**

## 这里放什么

三类内容，共同点是**还不够格成为知识，但丢掉可惜**：

1. **待查问题** — "为什么 X 这样设计"、"Y 和 Z 看起来重复"。写下来，避免同一个疑问被反复重新发现。
2. **一次性发现** — 排查过程中撞见的、与当前任务无关的事实。例如某个文档与源码不符（这类应同时登记到根目录的 [`UPSTREAM_DOC_ISSUES.md`](../../UPSTREAM_DOC_ISSUES.md)）。
3. **上下文交接** — 一段工作中断时的现场：做到哪、下一步是什么、有哪些还没验证的假设。

## 这里不放什么

| 内容 | 应该去 |
| --- | --- |
| 已成型、可复用的结论 | [`knowledge/`](../knowledge/index.md) |
| 多步骤工作的编排 | [`plans/`](../plans/index.md) |
| 仓库改动的决策与理由 | [`.agents/notes/`](../../.agents/notes/README.md)（随 PR 提交） |
| 任何长期有效的事实陈述 | 它的上游归属地——`docs/`、源码 JSDoc 或包 README |

最后一行是硬约束：备忘可以**指向**事实，但不能**成为**事实的家。见 [`AI_RULES.md`](../rules/combined/AI_RULES.md) §6。

## 命名约定

```
YYYY-MM-DD-<kebab-case-主题>.md
```

## 沉淀判据

一条备忘满足以下任意一条时，把它提取到 [`knowledge/`](../knowledge/index.md) 并删除原备忘：

- 同一个问题在不同任务中出现了第二次；
- 结论经过了实际验证，而不只是推测；
- 它能让下一个人少走一段确定的弯路。

**备忘的默认结局是被删除**，不是被无限保留。定期清理——一份三个月前的"待查"如果还没人去查，说明它不重要。

## 列表

_（暂无。新增时登记一行：`- [标题](YYYY-MM-DD-主题.md) — 一句话`）_

## 延伸阅读

- [`AI_Coding_Context.md`](../AI_Coding_Context.md) — dev_docs 总入口
- [`knowledge/index.md`](../knowledge/index.md) — 沉淀后的去处
- [`troubleshooting.md`](../troubleshooting.md) — 已成型的排障导航
