# API Requirements Quality Checklist — OryxOS v1.0 Web Service

**Purpose**: 对 Web Service 的需求做"英语单元测试"——验证 [spec.md](../spec.md) / [plan.md](../plan.md) / [contracts/rest-api.yaml](../contracts/rest-api.yaml) / [cli-commands.md](../contracts/cli-commands.md) 在 API 维度上的需求质量
**Created**: 2026-08-07
**Feature**: [spec.md](../spec.md) / [plan.md](../plan.md) / [contracts/](../contracts/)
**Theme**: api（REST API 全集，10 个端点）
**Audience**: Reviewer（PR 视角，标准深度）
**Actor timing**: 进入 implement 之前的最后一道闸门

> **方法论**：每条 item 评估**需求本身**的清晰度、完备性、一致性、可测性，**不**评估实现是否对。每条 item 形如"需求是否...?"而不是"系统是否...?"。

---

## Requirement Completeness（需求是否齐备）

- [ ] CHK001 — Are all 10 REST endpoints from [contracts/rest-api.yaml](../contracts/rest-api.yaml) explicitly traceable to a specific FR (FR-001~049)? [Completeness, Spec §FR-036]
- [ ] CHK002 — Are request body schemas for `POST /sessions`, `POST /sessions/{id}/messages`, `POST /agents/{name}/invoke` defined for all required fields AND optional fields (including their types)? [Completeness, Spec §FR-009]
- [ ] CHK003 — Are response schemas for all success paths (200/201/204) and failure paths (400/404/408/500/503) explicitly specified for every endpoint? [Completeness, Gap]
- [ ] CHK004 — Is the `session_id` generation rule (channel + user + profile hash, see [data-model.md §4](../data-model.md)) documented as a requirement, not just an implementation detail? [Completeness, Spec §FR-009]
- [ ] CHK005 — Are requirements for the audit-query interface (FR-044 / FR-045 / SC-014) specified for both Tool and LLM call records, including the query fields exposed to external callers? [Completeness, Gap, Spec §FR-044]
- [ ] CHK006 — Are requirements for the 5 health/info endpoints (`/health`, `/info`, `/tools`, `/memory`, `/profiles`) documented as "read-only / idempotent / no-side-effect" semantics? [Completeness, Spec §FR-036]

---

## Requirement Clarity（需求是否清晰可执行）

- [ ] CHK007 — Is the semantic difference between `POST /sessions/{id}/messages` (stateful, retains history) and `POST /agents/{name}/invoke` (stateless, fresh session) explicitly documented in the spec, not just in the OpenAPI description? [Clarity, Spec §FR-009 / §FR-037]
- [ ] CHK008 — Is "请求 ID / trace ID" mentioned in ErrorResponse explicitly defined as a field (e.g., format source: request header vs server-generated UUID)? [Clarity, Spec §ErrorResponse]
- [ ] CHK009 — Is the timeout behavior for `POST /sessions/{id}/messages` (408 response, see rest-api.yaml) quantified with a specific timeout value (vs leaving "agent timeout" vague)? [Clarity, Gap, Spec §SC-003]
- [ ] CHK010 — Are the channel enum values (`cli` / `web` / `cron`) explicitly constrained in CreateSessionRequest, with a definition of what each channel means for downstream behavior (e.g., `cron` excludes user_id_required)? [Clarity, Spec §CreateSessionRequest]

---

## Requirement Consistency（需求是否前后一致）

- [ ] CHK011 — Do endpoint counts match across documents: rest-api.yaml (10) ↔ spec.md FR-036 (10) ↔ plan.md US-5 (10) ↔ cli-commands.md quickstart (10)? [Consistency, Spec §FR-036]
- [ ] CHK012 — Does the three-trigger unification requirement (CLI / HTTP / Cron all via `AgentService.process()`, FR-037) hold across all 4 triggering endpoints in rest-api.yaml? [Consistency, Spec §FR-037 / §FR-040]
- [ ] CHK013 — Is the HTTP status code semantics consistent (e.g., 404 for missing session, 408 for agent timeout, 503 for unavailable service) across rest-api.yaml / quickstart.md / plan.md? [Consistency, Spec §Error Codes]
- [ ] CHK014 — Are Provider field enum values consistent between [contracts/agent-profile-schema.json](../contracts/agent-profile-schema.json) (enum: minimax / deepseek / qwen / ...) and [spec.md §US-1 user journey](../spec.md) (`qwen → minimax`)? [Consistency, Spec §US-1]

---

## Acceptance Criteria Quality（验收标准是否可测）

- [ ] CHK015 — Are the API performance targets (SC-003 "新建 Session P99 < 200ms" + SC-004 "内部转发 < 50ms") measurable against which specific request types (POST /sessions vs POST /sessions/{id}/messages)? [Measurability, Spec §SC-003 / §SC-004]
- [ ] CHK016 — Is the audit completeness criterion (SC-013 "100% Tool/LLM 调用落审计库") verifiable end-to-end via the API surface, not just an internal state? [Measurability, Spec §SC-013 / §SC-014]
- [ ] CHK017 — Is the "3 Demo 在同实例全部跑通" criterion (SC-015) testable purely via the REST API without manual CLI invocation? [Measurability, Spec §SC-015]

---

## Scenario Coverage（场景覆盖度）

- [ ] CHK018 — Are happy-path requirements (201 on session create, 200 on message send) AND error-path requirements (400 invalid profile, 404 session missing, 408 timeout, 500 internal) both specified for every endpoint? [Coverage, Spec §FR-036]
- [ ] CHK019 — Are concurrency requirements for 100 simultaneous sessions (SC-002) addressed at the API layer, not just at the storage layer? [Coverage, Gap, Spec §SC-002]
- [ ] CHK020 — Are requirements specified for "agent not loaded yet but /agents/{name}/invoke received" race condition during service startup? [Coverage, Gap, Spec §Edge Cases]
- [ ] CHK021 — Are deprecation/versioning requirements specified for `/api/v1` prefix when v2 is introduced (presumably extension phase)? [Coverage, Gap]

---

## Edge Case Coverage（边界情况覆盖）

- [ ] CHK022 — Is the behavior specified when a session_id is requested but the underlying Agent has been deleted from `.oryxos/agents/` after creation? [Edge Case, Spec §FR-007]
- [ ] CHK023 — Are requirements specified for malformed YAML in AGENT.md (Frontmatter parse failure) surfacing as a clear 4xx/5xx response on `POST /agents/{name}/invoke`? [Edge Case, Spec §FR-005 / §FR-043]
- [ ] CHK024 — Is the behavior defined when an MCP server referenced by an Agent becomes unavailable mid-request (FR-049 calls for retry on call-time, but the HTTP response semantics on persistent unavailability are not specified)? [Edge Case, Spec §FR-049]
- [ ] CHK025 — Are requirements specified for what happens when MEMORY.md exceeds 4000 chars (FR-024 truncation) DURING an active session — does the truncated version take effect immediately for the same session? [Edge Case, Spec §FR-024]

---

## Non-Functional Requirements（非功能性需求）

- [ ] CHK026 — Are HTTPS support requirements (FR-038) specified with a minimum TLS version, allowed cipher suites, or are these left to implementation? [Non-Functional, Spec §FR-038]
- [ ] CHK027 — Are authentication/authorization requirements explicitly stated as "intranet-only / no auth" with the assumption documented (vs silent omission)? [Non-Functional, Spec §Assumption]
- [ ] CHK028 — Are rate-limiting requirements specified for external HTTP integrations (`http_get` / `http_post` builtin Tools) — at the API level or only at the sandbox level? [Non-Functional, Gap]
- [ ] CHK029 — Are observability requirements (structured JSON logs, Micrometer/Prometheus metrics) specified as part of the API surface (e.g., `/actuator` endpoints), not just internal logging? [Non-Functional, Spec §Non-Functional]

---

## Dependencies & Assumptions（依赖与假设）

- [ ] CHK030 — Is the assumption "外部系统集成 OryxOS 不需要认证（内网假设）" documented as a formal Assumptions entry in spec.md (currently only in the introductory framing)? [Assumption, Spec §Assumptions]
- [ ] CHK031 — Are the dependencies on Spring Boot 3.x + Spring MVC behavior documented for behaviors like default error response format (which can conflict with custom GlobalExceptionHandler)? [Dependency, Spec §FR-036]

---

## Traceability Summary

| Item | Spec section | Plan section | Contract |
|------|--------------|--------------|----------|
| CHK001 | FR-036 | US-5 | rest-api.yaml paths |
| CHK004 | FR-009 | data-model.md §4 | rest-api.yaml Session schema |
| CHK007 | FR-009 / FR-037 | data-model.md | rest-api.yaml Session/Agent |
| CHK011 | FR-036 | US-5 §Project Structure | rest-api.yaml |
| CHK012 | FR-037 / FR-040 | US-5 Implementation | rest-api.yaml 4 trigger endpoints |
| CHK015 | SC-003 / SC-004 | plan.md §Technical Context | rest-api.yaml perf |
| CHK019 | SC-002 | plan.md §Performance Goals | rest-api.yaml |
| CHK024 | FR-049 | research.md D7 | (no API contract) |
| CHK026 | FR-038 | research.md D10 | (no API contract) |

**Traceability coverage**: 9/31 items explicitly mapped (~29%). Below the 80% SKILL.md recommended threshold — recommend adding explicit cross-references in spec.md or rest-api.yaml for the unmapped 22 items before implement.

---

## Items Not Applicable (NA, Recorded for Completeness)

The following were considered but excluded because they don't apply to the 10 REST endpoints or are implementation details outside the scope of API requirements:

- Tool registration performance (ToolRegistry lookup latency) — internal
- ReAct loop iteration timeouts — internal control flow
- SQLite write latency for audit tables — internal storage
- Provider ChatClient connection pool — internal

---

## Sign-off

- [ ] All requirements quality gaps above are either:
  - (a) Documented in spec.md / rest-api.yaml as additions, OR
  - (b) Marked as Deferred to extension phase with rationale

---

## 引用

- [spec.md](../spec.md) — 5 US + 53 FR（FR-001~056，跳号保留扩展余地）+ 15 SC
- [plan.md](../plan.md) — Technical Context + US-5 Implementation Path
- [data-model.md](../data-model.md) — entities & relationships
- [contracts/rest-api.yaml](../contracts/rest-api.yaml) — 10 REST 端点 OpenAPI
- [contracts/cli-commands.md](../contracts/cli-commands.md) — CLI 命令契约（与 HTTP 端点对照）
- [contracts/agent-profile-schema.json](../contracts/agent-profile-schema.json) — Provider name enum