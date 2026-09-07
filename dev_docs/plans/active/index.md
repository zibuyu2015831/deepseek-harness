---
title: 进行中的计划
summary: 正在执行的工作计划列表与进入/离开本目录的判据。
keywords: plans | active | 进行中
scope: dev_docs/plans/active 的索引
related_files: dev_docs/plans/index.md
dependencies: dev_docs/plans/index.md
verified_at: 2026-09-06
---

# 进行中的计划

> 目录规则见 [`plans/index.md`](../index.md)。

**进入判据**：目标、约束、可验证的分步都已写清，且已经开始执行。

**离开判据**：全部步骤通过各自的验证 → 移入 [`done/`](../done/index.md)；前提不再成立或被取代 → 移入 [`archive/`](../archive/index.md)。

同一时间保持少量条目。堆积说明计划被创建但没有被推进，这时应该归档而不是继续累加。

## 列表

- [阅读能力架构设计](2026-09-06-reading-capability-architecture.md) — 把 Web 阅读做成 `ctx.documents` 能力接缝；`document ↔ session` 用会话投影 + 可重建索引两套机制；助读卡片一张卡就是一条会话、可追问、卡片间并发，阅读界面内自建右栏承载会话列表与完整转录；UI 落 `shell.overlay`；六包划分与 M1–M8
