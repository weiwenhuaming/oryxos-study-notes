# Spec-Kit Analyze Report — OryxOS v1.0

**Branch**: `001-oryxos-v1-core` | **Date**: 2026-08-07（**Updated 2026-08-08**: 4 MEDIUM + 5 LOW 已修） | **Scope**: spec.md ↔ plan.md ↔ tasks.md ↔ constitution.md

> **2026-08-08 update**: 4 项 MEDIUM（F3 / C3 / C4 / E1+E2+E3）+ 5 项 LOW（F1 / F6 / C1 / C2 / A1-保留） 已在新会话中修复完成。详见下方"Disposition Log"段。

---

## TL;DR for Next Session

- **Critical Issues**: 0 — Constitution 7 条原则 + 8 条纠错清单 + Spring AI 红线全部对齐 ✅
- **Coverage**: 49 / 56 FR 显式映射 task (~87.5%)，高于 SKILL.md 80% 阈值
- **Total Findings**: 11 (3 MEDIUM + 5 LOW + 3 informational)
- **Suggested MVP fix scope**: 4 项 MEDIUM（详见"Next Actions"表），约 15 min 工作量

---

## Finding Table

| ID | Category | Severity | Location(s) | Summary | Recommendation |
|----|----------|----------|-------------|---------|----------------|
| **C1** | Underspec | MEDIUM | spec.md §SC-001 / SC-002 | "无降级" 缺度量基线（P99 退化？吞吐量？） | 加 SC 子项量化，或在 plan.md §Performance Goals 加"降级定义"段 |
| **C2** | Underspec | MEDIUM | spec.md §SC-006 | "Tool 调用成功率 ≥99%" 缺分母（系统全局？特定错误类？Demo 跑批？） | SC-006 加 "以 US-2 Demo 一批 Tool 调用为分母"的限定 |
| **C3** | Ambiguity | MEDIUM | spec.md §FR-011 / research.md §D5 / tasks.md §T035 | "切换 Provider 仅修改一个配置字段"——是 profile 里的 `provider.name`，还是 `application.yml` 里的 `oryxos.providers.<name>.api-key`？T035 verify 模糊 | FR-011 加注："特指 profile.provider.name 字段；API key 等基础设施配置改 application.yml" |
| **C4** | Inconsistency | MEDIUM | spec.md §FR-016 / §FR-048 / §US-2 验收 #2 | "指数退避最多 3 次" vs "30s / 2min / 8min"——FR-016 是 Tool 调用重试（节奏未定义），FR-048 是 scheduled task 重试（30s/2min/8min）。US-2 验收 #2 没指明走哪个节奏 | FR-016 加注："节奏由实现层决定；scheduled task 场景下与 FR-048 一致（30s / 2min / 8min）" |
| **F1** | Inconsistency | LOW | research.md §D5 | research.md D5 example 仍写 `deepseek`，未跟随 plan human-review 锁定的 `minimax` 统一 | research.md D5 example 改 minimax |
| **F3** | Inconsistency | MEDIUM | spec.md §FR-026 / research.md §D11 / plan.md §US-4 | FR-026 列 8 个内建 Tool；research D11 + plan US-4 说"8 + 2 通知 Tool (notify_slack/notify_webhook)"；FR-026 缺通知 Tool | FR-026 追加 "notify_slack / notify_webhook"，与 research.md D11 对齐 |
| **F6** | Inconsistency | LOW | spec.md §FR-049 | "MCP 重连节奏由实现层决定" + "单次重连超时 ≤ 5 秒"——tasks.md 中无对应 task 落地"重连实现" | tasks.md 加 T0xx MCP call-time retry 实现 task（或在 US-4 T046 注释里补） |
| **A1** | Duplication | LOW | spec.md §FR-024 + §Edge Cases "长期记忆文件超限" | Edge Cases 几乎是 FR-024 的改写 | 保留（Edge Cases 是验收可读性需要） |
| **D-1** | Constitution | ✅ PASS | — | 7 条原则 + 8 条纠错清单全部对齐（plan.md §Constitution Check 表格 + application.yaml Spring AI 红线 + research D1-D15） | 无 |
| **E1** | Coverage | LOW | spec.md §FR-047 | 24h Session 自动归档——plan.md 没明确哪个 task 实现；tasks.md 中未找到 T0xx 显式覆盖 | tasks.md 加 T0xx 或在 T015 SessionRepository 加注释 |
| **E2** | Coverage | LOW | spec.md §FR-048 | 30s/2min/8min retry——plan.md US-5 提及 cron，但 tasks.md 中"重试节奏" task 不可见 | tasks.md 加 T0xx 重试节奏实现 task |
| **E3** | Coverage | LOW | spec.md §FR-050 / §FR-051 / §FR-052 / §FR-053 | api.md 补的 6 项 FR-050~055 在 tasks.md 中**没有对应 task**（FR-054/055 是行为约束，无 task 也行；FR-050~053 是 API 行为应有 task） | tasks.md US-5 段加 T0xx-T0xx 4 个 task |

---

## Coverage Summary Table (FR → Task)

| FR | Task | Notes |
|----|------|-------|
| FR-001~008 (init / agent) | T023 / T028 | ✅ |
| FR-009~012 (Provider 抽象) | T029~T034 | ✅ |
| FR-013~017 (ReAct 循环) | T040~T044 (Phase 4) | ✅ |
| FR-018~025 (Memory) | T050~T053 (Phase 5) | ✅ |
| FR-026~032 (Tool 体系) | T036~T050 (Phase 4) | ✅ |
| FR-033~035 (CLI Channel) | T051~T053 | ✅ |
| FR-036~038 (10 REST 端点) | T054~T068 (Phase 6) | ✅ |
| FR-039~040 (Cron) | T069~T072 (Phase 6) | ✅ |
| FR-041~043 (配置) | T027 / T032 | ✅ |
| FR-044~046 (审计 day-one) | T016 / T017 / T031 + ReAct 实现 | ✅ |
| **FR-047** (24h 自动归档) | **❌ 未找到** | **E1** |
| **FR-048** (重试节奏 30s/2min/8min) | **❌ 未找到** | **E2** |
| **FR-049** (MCP 3s 启动探针) | T046 (mcp/) | ✅ |
| FR-050~053 (API 完整性) | **❌ 未找到** | **E3** |
| FR-054~055 (mid-session 截断 / 无认证) | 行为约束，无须独立 task | ✅ |
| FR-056 (fail-closed) | T041/T042 inline | ✅ |

**Coverage**: 49 / 56 FR 显式映射 task (~87.5%)。4 项缺 task (FR-047 / FR-048 / FR-050~053)。

---

## Constitution Alignment Issues

**None** — 7 条原则 + 8 条纠错清单 + Spring AI 红线 (`spring.ai.tool.execution.enabled=false` + `spring.ai.model.tool-calling.auto=false`) 全部对齐 ✅

具体验证点：

| Principle / Checklist | Validation |
|----------------------|------------|
| I. JDK 21 + 9 modules | pom.xml release=21 + 9 modules 在 parent pom §modules |
| II. Five Core Capabilities | spec §5 US + 3 Demos |
| III. Self-Implemented ReAct | tasks T040-T044 在 oryxos-core，无 Spring AI Agent 依赖 |
| IV. Spring AI Only Half | application.yaml §Spring AI boundary 段 + research.md D2 |
| V. Three-Tier Plugin Tool | tasks T036-T050 cover builtin + @Tool + MCP |
| VI. SQLite + MEMORY.md + Day-One Auditing | data-model.md 6 entities + tasks T015-T017 + FR-044/045/046 |
| VII. Per-US Demo Gate | spec §Demo 矩阵 + tasks 各 US 末尾 Verify |
| Deviation #1-8 (8 条纠错清单) | plan.md §Constitution Check 表 + research D4/D5/D6 + application.yaml |

---

## Unmapped Tasks

未发现：所有 task 都能映射回至少 1 个 FR / Demo 步骤 / constitution 原则。

---

## Metrics

- **Total Requirements**: 56 FR + 15 SC = 71
- **Total Tasks**: 92 (T001-T092)
- **Coverage %** (requirements with ≥1 task): **87.5%** (49/56 FR) — 高于 SKILL.md 80% 阈值
- **Ambiguity Count**: 3 (C1-C3, 1 MEDIUM + 2 MEDIUM)
- **Duplication Count**: 1 (A1, LOW)
- **Inconsistency Count**: 4 (F1 LOW, F3 MEDIUM, F6 LOW)
- **Underspec Count**: 2 (C1-C2, MEDIUM)
- **Critical Issues Count**: **0**

---

## Next Actions（4 项 MEDIUM + 5 项 LOW，按用户"宪法红线"风格分类）

### 🔴 MEDIUM — 建议立即修（spec 一致性 / 测试可执行性）

| # | ID | File | Change |
|---|------|------|--------|
| 1 | **F3** | [spec.md](../spec.md) FR-026 | 追加 `notify_slack / notify_webhook` 到内建 Tool 列表（与 research D11 + plan US-4 对齐） |
| 2 | **C3** | [spec.md](../spec.md) FR-011 | 加注："特指 profile.provider.name 字段；API key 等基础设施配置改 application.yml" |
| 3 | **C4** | [spec.md](../spec.md) FR-016 | 加注："节奏由实现层决定；scheduled task 场景下与 FR-048 一致（30s / 2min / 8min）" |
| 4 | **E1+E2+E3** | [tasks.md](../tasks.md) | 新增 5-6 个 task：T0xx Session 24h 自动归档 + T0xx 调度重试节奏 + T0xx~T0xx 审计查询 + Agent invoke 503/408/502 |

### 🟡 LOW — 可延后到 US-1 实施时顺手做

| # | ID | 建议动作 |
|---|------|---------|
| 5 | F1 | research.md D5 example `deepseek` → `minimax` |
| 6 | F6 | tasks.md T046 注释里补 MCP call-time retry 节奏 |
| 7 | A1 | 保留（Edge Cases 是验收可读性） |
| 8 | C1 | plan.md §Performance Goals 加"降级定义"段 |
| 9 | C2 | SC-006 加"以 US-2 Demo 一批 Tool 调用为分母" |

---

## 引用

- [spec.md](../spec.md) — 5 US + 56 FR + 15 SC
- [plan.md](../plan.md) — 9 模块 + Constitution Check + Performance Goals
- [tasks.md](../tasks.md) — 92 tasks across 8 phases
- [constitution.md](../../../.specify/memory/constitution.md) — 7 原则 + 8 偏离纠错清单
- [research.md](../research.md) — D1-D15 决策
- [data-model.md](../data-model.md) — 11 entities
- [checklists/api.md](./api.md) — 31 items
- [checklists/security.md](./security.md) — 30 items (含 FR-056 来源)

---

## 重建指南（新会话开头）

1. 读 [CLAUDE.md](../../../../CLAUDE.md) — 项目宪法 + 索引
2. 读本文件 — 知道当前在 implement 前的状态
3. 用户已经选择"修 4 项 MEDIUM 后再进 implement"——按 Next Actions 表执行
4. 修完后进 Phase 2 (T012-T028 Foundational) → US-1 → ...

---

## Disposition Log（2026-08-08 修复记录）

| ID | File | 修复方式 | 验证 |
|----|------|---------|------|
| **F3** | spec.md FR-026 | 追加 `notify_slack / notify_webhook`，引用 research.md §D11 | ✅ |
| **C3** | spec.md FR-011 | 加注：特指 `profile.provider.name` 字段；API key 走 application.yml | ✅ |
| **C4** | spec.md FR-016 | 加注：节奏实现层决定；scheduled task 场景与 FR-048 一致 | ✅ |
| **E1** | tasks.md T059 | 加注释：FR-047 24h 自动归档 + 按 ID 仍可查 | ✅ |
| **E2** | tasks.md T048 | 加注释：FR-048 + 告警 + `notified=true` + 不阻塞下一次 cron | ✅ |
| **E3** | tasks.md 新增 T082a/b/c/d | AuditController / 60s 超时 / 503 Retry-After / 502 mcp_unavailable | ✅ |
| **F1** | research.md §D5 | deepseek → minimax（决策行 + YAML 示例同步） | ✅ |
| **F6** | tasks.md T063 | 加注释：call-time 重连节奏 + ≤ 5s 超时 + FR-053 触发 502 | ✅ |
| **C1** | plan.md §Performance Goals | 新增"无降级量化定义" 5 项基线段 | ✅ |
| **C2** | spec.md SC-006 | 加注：分母限定为 US-2 Demo 一批 Tool 调用 | ✅ |
| **A1** | — | 保留（Edge Cases 验收可读性需要） | ✅ |

**状态**：4 项 MEDIUM + 5 项 LOW 已全部修复（2026-08-08）；spec.md / plan.md / tasks.md / research.md / data-model.md / checklists/ 之间一致性已对齐。可以进 Phase 2 (T012-T028 Foundational) → US-1 → US-2 → US-3+US-4 → US-5 → Phase 8 polish → tag v1.0.0-core。
