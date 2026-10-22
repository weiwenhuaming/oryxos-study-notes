# Implementation Plan: OryxOS v1.0 — 企业级 Agent OS 运行时内核（核心阶段）

**Branch**: `001-oryxos-v1-core` | **Date**: 2026-08-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-oryxos-v1-core/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

---

## Summary

实现 OryxOS 1.0 核心阶段交付的运行时内核：**9 个 Maven 模块**（oryxos-core / -provider / -memory / -tool / -channel-cli / -web / -storage / -cli / -boot），覆盖 5 大能力（Provider 抽象、ReAct 循环、三层记忆、Plugin Tool、Web Service）+ 定时调度 + 审计横切，**3 个验收 Demo（每日天气、每日科技日报、每日 GitHub 日报）全部跑通**。遵循 constitution v1.0.0 的 7 条原则和 8 条偏离纠错清单；模块拆分与依赖方向在 constitution §4 已锁定，**不可在 plan 阶段变更**。

---

## Technical Context

**Language/Version**: Java 21（virtual thread 是 first-class 需求；不接受 JDK 17/11）
**Build Tool**: Maven 多模块（9 模块固定）
**Application Framework**: Spring Boot 3.x（同步阻塞 + virtual thread，不上响应式）
**LLM Integration**: Spring AI + Spring AI Alibaba（**只用 Provider 抽象 + `@Tool` schema 生成；禁用自动 tool 执行**）
**ReAct Loop**: 自实现（约数十行 Java，位于 `oryxos-core`，**不**依赖 Spring AI Agent 抽象）
**MCP Integration**: MCP Java SDK（stdio 优先；SSE 放扩展阶段）— **Phase 6 实施时实际改用手写 stdio JSON-RPC,不引 SDK**(生态不熟 + 零依赖;ADR-001 留痕)
**Storage**: SQLite + Spring Data JPA（关系数据） + `.oryxos/memory/MEMORY.md` 文件（长期记忆）
**HTTP Server**: Spring MVC（**不**上 WebFlux / Reactor）
**CLI Framework**: Picocli
**YAML Parsing**: SnakeYAML（Profile / AGENT.md frontmatter）
**Logging**: Logback + SLF4J（结构化 JSON）
**Testing**: JUnit 5 + Mockito
**Target Platform**: Linux 服务器 / 物理机 / Docker / K8s（Ubuntu 22.04+ / CentOS 8+ / Debian 11+ / Alibaba Cloud Linux 3 / Rocky Linux）
**Project Type**: 单体应用（一个 fat JAR）
**Performance Goals**:
- 单实例 ≥10 个 Agent + ≥100 并发 Session 无降级（SC-001 / SC-002）
- 新建 Session P99 < 200ms（SC-003）
- 内部转发开销 < 50ms（SC-004）

**"无降级" 量化定义**（补 C1，避免 SC-001/SC-002 验收歧义）：
- "无降级" = **5 项可度量基线同时满足**：
  1. P99 Session 创建延迟退化 ≤ 20%（基线 < 200ms，负载后 < 240ms）
  2. P99 `/sessions/{id}/messages` Agent 处理延迟退化 ≤ 30%（基线为单 Session 串行下的处理时间）
  3. 错误率增量 ≤ 0.5%（5xx + 业务可重试错误的总和）
  4. CPU 占用稳态 ≤ 70%（10 核基线）
  5. SQLite `tool_invocations` / `llm_calls` 写入延迟 P99 ≤ 50ms
- 任意 1 项不满足即视为"已降级"，需在 PR review 时说明原因
**Constraints**:
- day-one 审计落库：tool_invocations / llm_calls 在响应返回前写库（FR-044 / FR-045 / FR-046）
- Tool 白名单：路径 / 命令 / 域名，100% 校验（FR-027 / SC-012）
- 配置无明文 key（FR-011 / FR-041 / SC-011）
**Scale/Scope**: 单实例 ~10 Agent / 100 Session；多实例集群放扩展阶段

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

按 `.specify/memory/constitution.md` v1.0.0 的 7 条原则 + 8 条偏离纠错清单逐项评估：

| # | 原则 | 当前状态 | 备注 |
|---|------|---------|------|
| I | JDK 21 + Spring Boot 3.x Monolith + 9 Maven Modules | ✅ Pass | Technical Context 已声明 JDK 21；9 模块在 §Project Structure 锁定 |
| II | Five Core Capabilities First | ✅ Pass | 5 US 范围与 spec §User Scenarios 一致；治理层明确放扩展 |
| III | Self-Implemented ReAct Loop | ✅ Pass | `ReActLoop` 落在 `oryxos-core`，自实现；不引入 Spring AI Agent 抽象 |
| IV | Spring AI Used Only Half（NO Auto Tool Execution） | ✅ Pass | 仅用 Provider + `@Tool` schema；自动 tool 执行**禁用**（详见 research.md "Spring AI 边界配置"决策） |
| V | Three-Tier Plugin Tool Adoption | ✅ Pass | 内建 Tool + `@Tool` 注解 + MCP 全部包装为 `OryxTool`，由 `ToolRegistry` 统一 |
| VI | SQLite + MEMORY.md File Storage（Day-One Auditing） | ✅ Pass | data-model.md 显式列出 Tool Invocation Record / LLM Call Record 实体，day-one 落库 |
| VII | Per-User-Story Demo Gate | ✅ Pass | 3 个 Demo 在 quickstart.md 完整定义；每个 US 末尾跑通对应 Demo |

**8 条偏离纠错清单对应处理**：

| # | 偏离点 | 处理位置 |
|---|--------|---------|
| 1 | JDK 21 特性 | Technical Context 显式声明 |
| 2 | Spring AI Agent 抽象 | research.md "ReAct 循环位置"决策 |
| 3 | Spring AI 自动 tool 执行 | research.md "Spring AI 边界配置"决策 |
| 4 | Tool 拆成多个模块 | 9 模块结构 oryxos-tool 合并 |
| 5 | Provider 类型扫描 | research.md "Provider 显式映射"决策 |
| 6 | AgentLoader / AGENT.md 归 Tool 模块 | 9 模块结构中 AgentLoader / ContextLoader 在 oryxos-core |
| 7 | Memory 跟 Session 合并 | research.md "MemoryService 三层门面"决策 |
| 8 | 审计表未写入 | data-model.md 显式两个审计实体 + FR-044/045/046 |

**结论**：全部 7 条原则 + 8 条纠错清单 Pre-design 评估**通过**，无 unjustified violation。

---

## Project Structure

### Documentation (this feature)

```text
specs/001-oryxos-v1-core/
├── plan.md              # 本文件
├── research.md          # Phase 0：关键技术决策记录
├── data-model.md        # Phase 1：实体字段 / 关系 / 约束
├── quickstart.md        # Phase 1：3 个 Demo 验证步骤
├── contracts/           # Phase 1：接口契约
│   ├── rest-api.yaml          # 10 个 REST 端点 OpenAPI
│   ├── agent-profile-schema.json  # AGENT.md frontmatter JSON Schema
│   ├── cli-commands.md        # 12 个 CLI 命令契约
│   └── mcp-servers-schema.json    # mcp_servers.yaml JSON Schema
├── checklists/
│   └── requirements.md        # spec 质量校验
├── spec.md             # 已存在
└── tasks.md             # Phase 2 由 /speckit.tasks 生成（**非本 plan 产出**）
```

### Source Code (repository root)

9 个 Maven 模块（**固定不变**——constitution §4 + TechnicalSolution §10）：

```text
oryxos-os/
├── pom.xml                          # 父 POM（dependencyManagement + pluginManagement）
├── oryxos-core/                     # 核心抽象 + 跨模块接口
│   ├── pom.xml
│   └── src/main/java/com/oryxos/core/
│       ├── OryxTool.java            # Tool 抽象接口（被 oryxos-tool 实现）
│       ├── Session.java             # 会话模型
│       ├── Profile.java             # 派生配置对象
│       ├── ContextLoader.java       # 加载 Bootstrap + AGENT.md 正文
│       ├── AgentLoader.java         # 扫描 .oryxos/agents/ + deriveProfile
│       ├── ReActLoop.java           # 自实现 ReAct 循环（核心）
│       ├── PromptBuilder.java       # 组装 system + history + tools
│       ├── ToolExecutor.java        # Tool 调度器（ReAct 循环调用）
│       ├── AgentService.java        # 统一入口（CLI/HTTP/Cron 三入口走同一条路径）
│       └── AgentScheduler.java      # cron 触发器（第三触发源）
├── oryxos-provider/                 # Provider 抽象 + 显式映射
│   ├── pom.xml
│   └── src/main/java/com/oryxos/provider/
│       ├── ProviderService.java
│       ├── ChatModel.java           # 内部统一接口（包装 Spring AI ChatClient）
│       └── ProviderRegistry.java    # 显式 provider name → ChatModel Bean 映射
├── oryxos-memory/                   # MemoryService 三层门面
│   ├── pom.xml
│   └── src/main/java/com/oryxos/memory/
│       ├── MemoryService.java       # 统一门面
│       ├── LongTermMemory.java      # MEMORY.md 文件后端
│       └── MemoryTools.java         # save_memory / recall_memory
├── oryxos-tool/                     # 三合一：内建 Tool + MCP + Sandbox + Notify
│   ├── pom.xml
│   └── src/main/java/com/oryxos/tool/
│       ├── builtin/                 # 内建 Tool（File / Shell / Http / Notify / Memory）
│       │   ├── ReadFileTool.java
│       │   ├── WriteFileTool.java
│       │   ├── ListDirTool.java
│       │   ├── ShellTool.java
│       │   ├── HttpTool.java
│       │   └── NotifyTools.java     # Slack / Webhook
│       ├── mcp/                     # MCP Client
│       │   ├── McpClientService.java
│       │   └── McpToolAdapter.java
│       ├── ToolRegistry.java        # 三档 Tool 统一注册表
│       ├── Sandbox.java             # 沙箱接口
│       └── WhitelistSandbox.java    # 应用层白名单实现
├── oryxos-channel-cli/              # CLI Channel
│   ├── pom.xml
│   └── src/main/java/com/oryxos/channel/cli/
│       └── CliChannel.java
├── oryxos-web/                      # Web Service（10 个 REST 端点）
│   ├── pom.xml
│   └── src/main/java/com/oryxos/web/
│       ├── WebServer.java
│       ├── controller/              # 6 个 ApiController
│       │   ├── SessionController.java
│       │   ├── AgentController.java
│       │   ├── ProfileController.java
│       │   ├── MemoryController.java
│       │   ├── ToolController.java
│       │   └── SystemController.java
│       ├── GlobalExceptionHandler.java
│       └── openapi/                 # OpenAPI 文档
├── oryxos-storage/                  # SQLite 持久化层
│   ├── pom.xml
│   └── src/main/java/com/oryxos/storage/
│       ├── entity/                  # JPA 实体
│       ├── SessionRepository.java
│       ├── ToolInvocationRepository.java
│       ├── LlmCallRepository.java
│       └── ScheduledTaskStore.java
├── oryxos-cli/                      # Picocli 主入口 + 12 个子命令
│   ├── pom.xml
│   └── src/main/java/com/oryxos/cli/
│       ├── InitCommand.java
│       ├── ChatCommand.java
│       ├── ServeCommand.java
│       ├── GatewayCommand.java
│       ├── ProfileCommand.java      # 子命令组
│       ├── ProviderCommand.java
│       ├── ToolCommand.java
│       ├── SessionCommand.java
│       ├── StatusCommand.java
│       └── ConfigLoader.java
└── oryxos-boot/                     # Spring Boot 启动模块
    ├── pom.xml
    └── src/main/java/com/oryxos/boot/
        └── OryxOsApplication.java   # @SpringBootApplication 主类
```

### 模块依赖图（constitution §4 模块铁律）

```text
                        ┌──────────────┐
                        │ oryxos-boot  │  ← 启动入口（聚合所有模块）
                        └──────┬───────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
      ┌──────────┐       ┌──────────┐       ┌──────────┐
      │ oryxos-  │       │ oryxos-  │       │ oryxos-  │
      │   cli    │       │   web    │       │ channel- │
      │(Picocli) │       │(Spring   │       │   cli    │
      │          │       │  MVC)    │       │          │
      └─────┬────┘       └─────┬────┘       └────┬─────┘
            │                  │                  │
            └──────────────────┼──────────────────┘
                               ▼
                        ┌──────────────┐
                        │ oryxos-core  │  ← 抽象 + 跨模块接口（不依赖任何业务模块）
                        └──────┬───────┘
                               │
        ┌──────────┬───────────┼───────────┬──────────┐
        ▼          ▼           ▼           ▼          ▼
   ┌────────┐ ┌────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐
   │provider│ │ memory │ │   tool   │ │storage │ │  (扩展)  │
   └────────┘ └────────┘ └──────────┘ └────────┘ └──────────┘

依赖铁律：
- oryxos-core 不依赖任何业务模块
- oryxos-tool 是合并模块（不拆为 builtin/skill/mcp）
- oryxos-channel-cli 底层调 oryxos-web 的 Agent 接口（不重复实现 Agent 逻辑）
- 所有 Channel 走同一份 AgentService.process()（FR-037）
```

**Structure Decision**: 采用 Maven 9 模块结构；`oryxos-boot` 作为启动聚合模块；模块依赖严格按 constitution §4 锁定。

---

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| （无） | — | — |

**说明**：7 条原则 + 8 条纠错清单全部通过，**无 violation 需要 justification**。plan 不引入任何新模块、不修改 constitution、不拆分 `oryxos-tool`。

---

## 5 US 实现路径概要（按依赖顺序，非重要性）

### US-1（Priority: P1）— Provider 抽象

**涉及模块**：`oryxos-core`（`Profile` / `PromptBuilder`） + `oryxos-provider` + `oryxos-boot`（主类 + 自动配置）

**关键实现点**：
- `ProviderService` 通过 `ProviderRegistry` 按 profile.provider.name 显式查表拿到对应 `ChatModel` Bean（不类路径扫描）
- 每次调用 `ChatModel.call()` 前/后写入 `llm_calls` 表（FR-012 / FR-045）
- `application.yml` 里 Spring AI 自动配置仅启用 ChatClient / ChatModel；tool execution auto-config **禁用**

**验收**：修改 profile 的 `provider.name` 字段 + 重启即可切换 LLM Provider。

---

### US-2（Priority: P2）— ReAct 循环

**涉及模块**：`oryxos-core`（`ReActLoop` + `ToolExecutor`） + `oryxos-tool`（内建 HTTP Tool） + `oryxos-channel-cli`（CLI Channel） + `oryxos-cli`（`chat` / `serve` / `gateway` 3 个入口）

**关键实现点**：
- `ReActLoop`：循环 `while (iteration < max_iterations)`，每次调 LLM 拿到响应 → 若含 tool_calls 则 `ToolExecutor.execute()` → 把 tool 结果追加到对话历史 → 继续
- `ToolExecutor`：从 `ToolRegistry` 按 tool 名查找 + `WhitelistSandbox` 校验 + 调用
- 三种入口（`cli chat` / `serve` HTTP / `gateway` cron）都走同一个 `AgentService.process(profile, message)`
- 每次 Tool 调用写入 `tool_invocations` 表（FR-044）

**验收**：CLI 多轮对话中 Agent 能自主调用 HTTP Tool 完成天气查询；Demo 一前置。

---

### US-3（Priority: P3）— 三层记忆

**涉及模块**：`oryxos-memory`（`MemoryService` / `LongTermMemory` / `MemoryTools`） + `oryxos-core`（`PromptBuilder` 注入长期记忆）

**关键实现点**：
- `MemoryService` 是统一门面，对外只暴露 `add()` / `recall()` / `getContext()` 三个方法，内部委托给 `SessionManager`（会话层）+ `LongTermMemory`（长期层）
- `LongTermMemory` 后端：`.oryxos/memory/MEMORY.md` 文件，两分区（`## 核心记忆` / `## 归档记忆`）
- 超 4000 字时按 FR-024 策略截断：核心记忆区全部保留 + 归档区按 LRU 整段丢弃
- `save_memory` / `recall_memory` 是 `MemoryTools` 提供的内建 Tool（在 `oryxos-tool` 模块注册）
- 24h 无活动 Session 自动归档（FR-047）

**验收**：Agent 在第一次对话中 `save_memory("偏好")`，第二次对话开始时该偏好已经在 system prompt 里；Demo 二前置。

---

### US-4（Priority: P3）— Plugin Tool 体系

**涉及模块**：`oryxos-tool`（`ToolRegistry` + `McpClientService` + `McpToolAdapter` + 三档 Tool 统一注册）

**关键实现点**：
- 三档来源（内建 / `@Tool` 注解 / MCP）都包装成 `OryxTool`，由 `ToolRegistry` 统一注册；`ReActLoop` 不感知来源
- MCP 集成用 MCP Java SDK，stdio 优先（SSE 放扩展）— 实际 Phase 6 实施时改用手写 stdio JSON-RPC,SnakeYAML 解析 yaml 不引 jackson-dataformat-yaml(ADR-001)
- Agent 启动时对每个 MCP server 尝试 3s 超时快速连接（FR-049），不通仅警告不阻塞
- `WhitelistSandbox`：文件路径前缀匹配 / Shell 命令白名单 / HTTP 域名白名单
- 8 个内建 Tool：read_file / write_file / list_dir / shell / http_get / http_post / save_memory / recall_memory

**验收**：业务方写 `AGENT.md` 引用 MCP server，零代码扩展 Tool；Demo 二前置。

---

### US-5（Priority: P4）— Web Service

**涉及模块**：`oryxos-web`（10 个 REST 端点） + `oryxos-storage`（审计查询接口） + `oryxos-cli`（`serve` 子命令） + `oryxos-core`（`AgentService` 复用）

**关键实现点**：
- 6 个 `ApiController`（Session / Agent / Profile / Memory / Tool / System）
- `GlobalExceptionHandler` 统一错误响应（结构化 JSON）
- OpenAPI 文档生成在 `oryxos-web` 的 `openapi/` 目录
- 所有端点最终调 `AgentService`（与 CLI/Cron 同一条路径）

**验收**：3 个 Demo 全部跑通（同实例）；10 个 REST 端点全部端到端可调。

---

## 下一步

本 plan 完成后，需人工 review 锁定（CLAUDE.md §9 红线），然后进入 `/speckit.tasks` 阶段拆分 tasks。

Phase 0 / Phase 1 产出物（`research.md` / `data-model.md` / `contracts/*` / `quickstart.md`）随 plan.md 一同生成。