# Feature Specification: OryxOS v1.0 — 企业级 Agent OS 运行时内核（核心阶段）

**Feature Branch**: `001-oryxos-v1-core`

**Created**: 2026-08-07

**Status**: Draft

**Input**: User description: "OryxOS v1.0 — 企业级 Agent OS 运行时内核（核心阶段）。详见 §OryxOS 业务背景 / §5 个 user story / §3 个验收 Demo / §关键 Entities / §非功能需求 / §明确不在范围 / §如何验收成功。"

## Overview

OryxOS 是面向严监管企业（金融、制造、政府背景 SaaS 等）的"Agent 操作系统"运行时内核。部署在企业自己的基础设施上，所有数据留在企业内部。本特性定义核心阶段（v1.0）交付的运行时内核：5 大核心能力（LLM Provider、ReAct 循环、三层记忆、Plugin Tool、Web Service）外加调度与审计两大横切能力，3 个验收 Demo 全部跑通。

**重要边界**：核心阶段交付的是 Agent OS 的"运行时内核底座"；企业级治理（多租户、SSO、完整审计 UI、Tool Policy）在扩展阶段做，不在本特性范围。

## Clarifications

### Session 2026-08-07

- Q: 当 `.oryxos/memory/MEMORY.md` 累积超过 4000 字时，系统应按什么策略截断？ → A: B — 保留"核心记忆"区全部 + 归档区按 LRU 丢弃
- Q: Session 在没有被显式调用归档 API 时，是否需要自动归档？自动归档按什么条件触发？ → A: B — 24 小时无活动自动归档（归档后 Session 不出现在活跃列表但仍可通过历史查询接口访问）
- Q: 当 AgentScheduler 定时触发的 Agent 任务因 LLM 故障 / Tool 失败 / 系统错误未能完成时，系统应按什么策略处理？ → A: B — 指数退避重试 3 次（30s / 2min / 8min），全失败再告警（与 FR-016 节奏一致）
- Q: Agent 启动时如果其引用的某个 MCP server 不可达（连接失败 / 进程未起 / 网络问题），系统应按什么策略处理？ → A: C — 启动时尝试快速连接（3s 超时）+ 不通仅记警告 + 调用时再尝试重连（平衡体验与故障隔离）

---

## User Scenarios & Testing *(mandatory)*

> **优先级与排序说明**：5 个 US 按**依赖关系**排序（同时也是商业重要性排序），从最底层的 Provider 抽象到最上层的 Web Service。P1 = 无依赖、必须最先做；P4 = 依赖全部、最末做。每个 US 是独立可演示的功能切片，依赖关系通过"上游 US 的接口必须先就绪"来满足。

### User Story 1 — LLM Provider 抽象 (Priority: P1)

**用户旅程**：
运维 Alice 在新员工入职场景下，需要把现有 Agent 切换到公司采购的便宜模型以控制成本。她打开 Agent 的 Profile 配置文件，修改"Provider"字段从"qwen"改为"minimax"（或其他任意供应商），Agent 立即开始调用新模型，无需代码改动。

**为什么是这个优先级**：所有其他能力（ReAct 循环、Memory、Tool、Web Service）都依赖 LLM 调用，Provider 抽象是地基。无 P1，P2~P4 全部无法跑通。

**独立可测**：能通过修改一个配置字段切换 Provider，且新 Provider 的 LLM 调用能正常完成一次完整对话。

**验收场景**：
1. **Given** Agent Profile 配置使用 Provider A，**When** 用户发起一次对话，**Then** 系统通过 Provider A 完成 LLM 调用并返回响应
2. **Given** Agent Profile 配置使用 Provider A，**When** 运维把 Provider 字段改为 Provider B 并重启，**Then** Agent 通过 Provider B 完成相同的对话
3. **Given** Agent 完成一次 LLM 调用，**When** 审计人员查询调用记录，**Then** 能查到调用包含 Provider 名、模型名、token 数、耗时

---

### User Story 2 — ReAct 循环 (Priority: P2)

**用户旅程**：
客服经理 Bob 让 Agent 帮他处理一个客户咨询："帮我查 A 公司上个月的销售总额并生成报告"。Agent 自主决定：先调 CRM 工具拉销售数据 → 再调文件工具生成报告 → 再调通知工具发给 Bob。Bob 没有预先编排这个流程，Agent 自己串联了 3 个 Tool 完成。

**为什么是这个优先级**：ReAct 循环把 Provider 抽象升级为"会做事的 Agent"，是产品差异化的核心。依赖 US-1（需要先有 Provider 才能调 LLM）。

**独立可测**：能完成"LLM 决定调 Tool → 执行 Tool → 看结果 → 继续或结束"的多步骤任务，达到最大迭代次数时正确终止，所有 Tool 调用可观测。

**验收场景**：
1. **Given** Agent 收到一个需要多步骤完成的任务，**When** LLM 第一次返回中包含 Tool 调用请求，**Then** 系统执行 Tool 并把结果反馈给 LLM 继续循环
2. **Given** LLM 在循环中决定调 Tool 失败，**When** 失败属于可重试错误，**Then** 系统按指数退避重试最多 3 次
3. **Given** ReAct 循环已经执行了配置允许的最大迭代次数，**When** 仍未产生最终响应，**Then** 系统强制终止并返回明确错误给用户
4. **Given** Agent 完成一次多步骤任务，**When** 审计人员查询记录，**Then** 每次 LLM 调用和每次 Tool 调用都完整可查

---

### User Story 3 — 三层记忆 (Priority: P3)

**用户旅程**：
销售 Charlie 第二次跟销售助理 Agent 对话时说"按上次的偏好给我看 A 公司"。Agent 之前在第一次对话中听到 Charlie 说"我只看头部客户，不看小客户"，已经写到了长期记忆里。这次对话开始时，系统就把这个偏好加载到 Agent 的上下文，Agent 立刻正确理解了 Charlie 的请求。

**为什么是这个优先级**：Memory 让 Agent 跨对话连续，是从"工具"升级为"助理"的关键。依赖 US-2（需要在 ReAct 循环里能调 `save_memory` / `recall_memory`）。

**独立可测**：Agent 在一次对话中通过 `save_memory` 写一个事实，下次对话开始时该事实已经出现在系统上下文中。

**验收场景**：
1. **Given** 用户在对话中调用 `save_memory("用户偏好：只看头部客户")`，**When** Agent 重启后再开新对话，**Then** 该偏好出现在 Agent 的系统上下文中
2. **Given** 长期记忆文件积累了 4000+ 字，**When** Agent 启动，**Then** 系统按规则截断文件但保留最近的事实
3. **Given** 用户在对话中调用 `recall_memory("A 公司")`，**When** 长期记忆中有相关条目，**Then** Agent 收到匹配的条目用于后续生成
4. **Given** 当前对话超过 LLM 上下文窗口，**When** 后续消息进来，**Then** 系统截断早期对话保留近期消息

---

### User Story 4 — Plugin Tool 体系 (Priority: P3)

**用户旅程**：
HR 经理 Diana 需要让 Agent 能查公司花名册。她不需要懂编程语言，只在 Agent 目录下写一份 `AGENT.md`，复用社区现成的"HRIS MCP server"（提供查花名册的能力），配置 `mcp_servers.yaml` 启用这个 server，Agent 立即能在对话中调花名册查询。整个过程没碰一行 Java 代码。

**为什么是这个优先级**：Plugin Tool 让业务方零代码扩展能力，是 OS 模式（"业务方不写 Agent 后端"）的核心。依赖 US-2（需要 ReAct 循环能 dispatch Tool）。与 US-3 并行。

**独立可测**：业务方通过写 markdown 配置文件 + 复用现成 MCP server，能让 Agent 获得新能力，且新 Tool 在 ReAct 循环中表现与内建 Tool 一致。

**验收场景**：
1. **Given** 业务方在 `AGENT.md` 中通过 `tools` 字段引用一个新 MCP server，**When** Agent 启动，**Then** 新 Tool 出现在 Agent 可用 Tool 列表
2. **Given** LLM 决定调用某个文件类 Tool（如读文件），**When** 调用参数涉及路径白名单外的目录，**Then** 系统拒绝执行并返回明确错误
3. **Given** LLM 决定调用某个 HTTP Tool，**When** 目标 URL 不在域名白名单内，**Then** 系统拒绝执行
4. **Given** 业务方用注解方式写一个 Tool Bean，**When** Agent 启动，**Then** 这个 Tool 跟内建 Tool 和 MCP Tool 在 ReAct 循环中表现一致

---

### User Story 5 — Web Service (Priority: P4)

**用户旅程**：
CRM 系统的开发者 Eve 需要在自己的 Web 应用里集成销售 Agent。她调 `POST /api/v1/agents/sales-agent/invoke` 发个 HTTP 请求，Agent 返回分析结果。她还可以用 `GET /api/v1/sessions/{id}` 查历史对话、用 `GET /api/v1/tools` 看 Agent 当前可用能力。整个集成过程没启动 OryxOS 的 CLI，也没装额外 SDK，纯 HTTP 调用。

**为什么是这个优先级**：Web Service 是企业把 OryxOS 嵌入已有业务系统的唯一通道，是"OS 模式"商业落地的关键。依赖全部前 4 个 US。

**独立可测**：外部系统通过 10 个 REST 端点完整调用 OryxOS 全部 5 大能力（覆盖会话、Agent、Profile、Memory、Tool、系统状态 6 类操作）。

**验收场景**：
1. **Given** 外部系统调 `POST /api/v1/sessions`，**When** 请求合法，**Then** 系统返回新创建的 Session ID
2. **Given** 外部系统调 `POST /api/v1/sessions/{id}/messages` 发消息，**When** Agent 处理完，**Then** 系统返回 Agent 的最终响应
3. **Given** 外部系统调 `POST /api/v1/agents/{name}/invoke`（无状态调用），**When** Agent 处理完，**Then** 系统返回 Agent 的响应（不创建 Session）
4. **Given** 运维调 `GET /api/v1/health`，**When** 系统健康，**Then** 返回 200 和健康状态

---

### Edge Cases

- **Provider 故障**：核心阶段 LLM Provider 不可达时直接报错给 Agent（**不**自动 failover，扩展阶段做）
- **Tool 调用超时**：单次 Tool 调用超过配置的超时阈值时被中断并返回超时错误
- **Shell 命令白名单外**：执行危险命令（如 `rm -rf /`）被拒绝
- **路径穿越**：文件类 Tool 接收 `../../../etc/passwd` 等穿越路径被路径白名单拒绝
- **跨重启 Session 恢复**：OryxOS 重启后能恢复未结束的 Session
- **Session 自动归档**：24 小时无活动自动从 `active` 切 `archived`，对应 FR-047；归档不是删除，按 ID 仍可查
- **定时任务失败重试**：定时任务失败按 FR-048 自动重试 3 次（30s/2min/8min），全失败再告警；本次重试不阻塞下一次周期触发
- **长期记忆文件超限**：超过 4000 字时按 FR-024 的分区+LRU 策略截断（保留核心记忆区全部，超出部分从归档区按整段 LRU 丢弃；核心阶段不做语义压缩，扩展阶段再做）
- **多 Agent 同名冲突**：不允许同名 Agent 目录共存
- **并发 Session 写同一 Agent**：100 个 Session 同时写同一 Agent 的状态不能出现数据损坏
- **MCP server 不可达**：调用的 MCP server 连接失败时返回明确错误给 Agent，不让 ReAct 循环卡死
- **MCP server 启动时不可达**：Agent 启动时尝试 3s 超时快速连接，不通仅记警告不阻塞 Agent 启动（对应 FR-049）；调用 MCP Tool 时再尝试重连；这是与上一条"调用时不可达"互补的"启动时"语义

---

## Requirements *(mandatory)*

### Functional Requirements

#### 工作区与初始化

- **FR-001**: 系统 MUST 提供一条命令初始化 `.oryxos/` 工作区目录
- **FR-002**: 初始化命令 MUST 幂等：已存在的目录和文件不覆盖
- **FR-003**: 初始化后 MUST 自动创建 3 个 Bootstrap 文件（AGENTS.md、SOUL.md、USER.md）

#### Agent 与 Profile

- **FR-004**: 一个 Agent MUST 由 `.oryxos/agents/<name>/` 目录定义
- **FR-005**: Agent 的 frontmatter MUST 包含：name、description、provider、tools、settings 字段
- **FR-006**: 系统 MUST 支持多个 Agent 在同一实例并存（≥10 个）
- **FR-007**: 系统 MUST 提供列出 / 查看 / 删除 Agent 的命令
- **FR-008**: 修改 Profile MUST 无需改代码（仅修改配置文件 + 重启）

#### LLM Provider（US-1）

- **FR-009**: 系统 MUST 通过统一内部接口路由 LLM 调用，Agent 不感知具体 Provider
- **FR-010**: 系统 MUST 支持 OpenAI 兼容协议的所有 Provider（包括自部署推理服务）
- **FR-011**: 切换 Provider MUST 仅修改一个配置字段，不改代码（"一个配置字段"**特指** `AGENT.md` frontmatter 的 `provider.name` 字段；改完后 Agent 必须重启才生效，参见 FR-008；API key / base-url 等基础设施配置仍走 `application.yml` 的 `oryxos.providers.<name>.*` 块，**不**算"切换字段"）
- **FR-012**: 每次 LLM 调用 MUST 持久化记录：Provider 名、模型名、token 数（输入/输出/总）、耗时

#### ReAct 循环（US-2）

- **FR-013**: Agent 收到任务后 MUST 启动 Reason+Act 迭代循环
- **FR-014**: 循环 MUST 在 LLM 给出无 Tool 调用的响应时结束
- **FR-015**: 循环 MUST 在达到最大迭代次数（可配置）时强制终止
- **FR-016**: Tool 调用失败且属于可重试错误时 MUST 按指数退避重试，最多 3 次（"指数退避"的具体节奏**由实现层决定**；**注意**：`AgentScheduler` 定时任务场景下的重试节奏**与 FR-048 一致**——30s / 2min / 8min——因为该场景是系统级而不是单次 Tool 调用级，重试节奏已在 FR-048 锁定；FR-016 适用于"ReAct 循环中单次 Tool 调用失败立即重试"的场景）
- **FR-017**: 每次 LLM 调用和 Tool 调用 MUST 记录结构化日志

#### Memory（US-3）

- **FR-018**: 会话记忆 MUST 包含当前对话的完整历史
- **FR-019**: 长期记忆 MUST 存储为单一文件（`.oryxos/memory/MEMORY.md`）
- **FR-020**: 长期记忆 MUST 跨所有对话保留
- **FR-021**: 系统 MUST 提供 `save_memory` 内建 Tool 追加内容到长期记忆
- **FR-022**: 系统 MUST 提供 `recall_memory` 内建 Tool 按关键词检索长期记忆
- **FR-023**: Agent 启动时 MUST 把长期记忆文件加载到系统提示词
- **FR-024**: 长期记忆文件超过 4000 字时 MUST 按如下策略截断：保留 `## 核心记忆` 区全部内容；若仍超 4000 字，从 `## 归档记忆` 区按 LRU（最旧优先）丢弃；丢弃时整段（按 Markdown 一级或二级标题分段）原子丢弃，避免半段残留在文件里
- **FR-025**: 上下文超过 LLM 窗口时 MUST 截断早期对话

#### Tool 体系（US-4）

- **FR-026**: 系统 MUST 提供内建 Tool：read_file、write_file、list_dir、shell、http_get、http_post、save_memory、recall_memory、notify_slack、notify_webhook（与 [research.md §D11](../research.md) "8 + 2（通知）" 一致；通知 Tool 调用走全局 `NotifyChannelAdapter` 注册表，**不**经 Sandbox 白名单）
- **FR-027**: 内建 Tool MUST 受应用层白名单约束（文件路径、Shell 命令、HTTP 域名）
- **FR-028**: 内建 Tool MUST 支持单次调用超时
- **FR-029**: 业务方 MUST 能通过写 `AGENT.md` + 复用现成 MCP server 零代码扩展 Tool
- **FR-030**: 业务方 MUST 能通过写自定义 MCP server 扩展 Tool
- **FR-031**: 业务方 MUST 能通过写被标注的 Bean 扩展 Tool
- **FR-032**: 三档来源 MUST 在 ReAct 循环中表现一致

#### Channel 与 CLI

- **FR-033**: 系统 MUST 提供 CLI Channel 支持多轮对话
- **FR-034**: CLI MUST 支持 `--message "xxx"` 参数发单条消息后退出
- **FR-035**: CLI MUST 支持 `--profile <name>` 参数指定 Agent

#### Web Service（US-5）

- **FR-036**: 系统 MUST 暴露以下 10 个 REST 端点：
  - `POST /api/v1/sessions`
  - `POST /api/v1/sessions/{id}/messages`
  - `GET /api/v1/sessions/{id}`
  - `DELETE /api/v1/sessions/{id}`
  - `POST /api/v1/agents/{name}/invoke`
  - `GET /api/v1/profiles`
  - `GET /api/v1/memory`
  - `GET /api/v1/tools`
  - `GET /api/v1/health`
  - `GET /api/v1/info`
- **FR-037**: CLI、定时、HTTP 三种调用入口 MUST 走同一条内部执行路径
- **FR-038**: API MUST 支持 HTTPS

#### 调度

- **FR-039**: 系统 MUST 支持按 cron 表达式定时触发 Agent
- **FR-040**: 定时触发 MUST 走与 CLI / HTTP 相同的内部执行路径

#### 配置与密钥

- **FR-041**: 敏感配置 MUST 支持通过环境变量注入
- **FR-042**: Profile 文件 MUST 支持 `${ENV_VAR}` 占位引用
- **FR-043**: 配置加载时 MUST 做基础校验（必填项、格式）

#### 审计（横切硬约束）

- **FR-044**: 每次 Tool 调用 MUST 持久化调用记录，包含：会话 ID、Tool 名、输入、结果、成功/失败、错误信息、耗时
- **FR-045**: 每次 LLM 调用 MUST 持久化调用记录，包含：会话 ID、Provider、模型、token 数（输入/输出/总）、耗时
- **FR-046**: 审计写入 MUST 在响应返回给调用方之前完成
- **FR-047**: Session 在 24 小时内无任何消息活动 MUST 自动从 `active` 状态切换为 `archived` 状态；归档后的 Session MUST 不出现在活跃 Session 列表接口中，但仍 MUST 可通过 `GET /api/v1/sessions/{id}` 按 ID 查询到
- **FR-048**: 定时触发的 Agent 任务在 LLM 故障 / Tool 失败 / 系统错误导致未完成时 MUST 按 30s / 2min / 8min 的指数退避节奏自动重试最多 3 次；3 次重试仍失败 MUST 通过 NotifyChannel 告警并写入审计；重试不阻塞下一次周期触发（错过本次周期不影响下一次按计划触发）
- **FR-049**: Agent 启动时 MUST 对其引用的每个 MCP server 尝试一次快速连接（3 秒超时）；连接失败 MUST 仅记警告而不阻塞 Agent 启动；MCP Tool 被调用时 MUST 再尝试重连（重连节奏由实现层决定，但单次重连超时 MUST ≤ 5 秒）

#### API 完备性（apichecklist 补强）

- **FR-050**: 系统 MUST 暴露审计查询接口，允许业务方按会话 ID、时间范围、调用类型（Tool / LLM）查询历史调用记录；查询响应 MUST 包含调用时间、调用方、Tool 名 / Provider 名、输入摘要（脱敏后）、结果摘要、token 数（如适用）、耗时
- **FR-051**: Agent 同步调用接口（`POST /api/v1/sessions/{id}/messages` 与 `POST /api/v1/agents/{name}/invoke`）的 408 响应 MUST 在 **60 秒** 内返回（业务方调用方可配置 override，默认 60s）；超过此时限的 Agent 处理 MUST 在响应中携带已完成的中间步骤（如已落审计的 Tool 调用列表）
- **FR-052**: 服务启动期间（Agent 还未完成加载）收到的 `POST /api/v1/agents/{name}/invoke` MUST 返回 **503 Service Unavailable** 并带 `Retry-After: 5` 头；不允许返回 404（404 仅用于 Agent 目录不存在）
- **FR-053**: 调用 MCP Tool 时若 MCP server **持续不可达**（已按 FR-049 重试 1 次仍失败），`POST /api/v1/sessions/{id}/messages` MUST 返回 **502 Bad Gateway** 且响应中必须包含 `error.code = "mcp_unavailable"`；调用方可根据此 code 判断是否放弃
- **FR-054**: 长期记忆文件按 FR-024 截断后，正在使用该记忆的活跃 Session MUST **不感知** 即时截断——本轮 prompt 已注入的 memory context 保持不变；下一轮对话才使用截断后的版本（避免 mid-session context 不一致）
- **FR-055**: Web Service MUST 假设运行在内网环境，**不** 实现任何认证 / 授权机制；所有 10 个 REST 端点对任何能访问服务的客户端开放（详见 Assumptions 段）
- **FR-056**: 应用层白名单校验 MUST 遵循 **fail-closed** 语义：任何无法精确判断为允许的输入（例如路径不在白名单、命令未匹配任何白名单条目、HTTP 域名解析异常、配置为空且无默认值）一律拒绝执行并返回明确错误；**不**允许 fail-open（"无法判断就放行"）或静默警告后放行

### Key Entities

- **Agent（Agent 目录）**：业务 Agent，由 `AGENT.md`（frontmatter + 正文）加可选的 `skills/` 软连接和 `scripts/` 定义。**一个目录 = 一个完整 Agent**。
- **Profile**：从 Agent frontmatter 派生的运行时配置对象，包含绑定的 Provider、模型、Tool 列表、Channel 绑定、定时规则。
- **Provider**：LLM 服务抽象，每个 Agent 配置一个 Provider，支持 OpenAI 兼容协议的所有实现。
- **Session**：会话容器，由 channel + user + agent 联合标识，持久化消息历史。
- **Memory**：长期记忆文件（`.oryxos/memory/MEMORY.md`），跨会话保存事实，两分区结构（核心记忆、归档记忆）。
- **Tool**：可调用的能力，统一接口，与来源无关（内建 / 注解 / MCP）。
- **Skill**：公共可复用知识模块，Agent 通过本地软连接引用；每轮 prompt 中出现元数据，正文按需读取。
- **Channel**：消息入口（核心阶段仅 CLI；IM 频道延后）。
- **Bootstrap 文件**：启动时加载到系统提示词的 markdown 文件（AGENTS.md / SOUL.md / USER.md）。
- **Tool Invocation Record**：每次 Tool 调用的完整审计记录（输入、结果、成功/失败、耗时）。
- **LLM Call Record**：每次 LLM 调用的完整审计记录（Provider、模型、token、耗时）。

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

#### 容量与性能

- **SC-001**: 单实例 MUST 稳定运行 ≥10 个 Agent 无降级
- **SC-002**: 单实例 MUST 稳定运行 ≥100 个并发 Session 无降级
- **SC-003**: 新建 Session 的 P99 延迟 MUST < 200ms
- **SC-004**: 平台自身内部转发开销 MUST < 50ms（**不含** LLM 调用本身的延迟）

#### 可靠性

- **SC-005**: 已注册的 Agent 配置和已持久化的 Session 数据 MUST 保证不丢
- **SC-006**: Tool 调用失败且属于可重试错误时 MUST 在 3 次重试内成功（成功率 ≥99%；分母限定为 **US-2 Demo 一批 Tool 调用**：可重试错误场景下的内建 Tool 调用样本，不含 MCP Tool 调用、不含 Provider 故障）

#### 可运维性

- **SC-007**: 新运维 MUST 能在 30 分钟内从零完成单节点部署（仅依赖提供的文档）
- **SC-008**: 配置变更 MUST 在重启后生效
- **SC-009**: 部署 MUST 支持物理机、虚拟机、Docker、K8s

#### 兼容性

- **SC-010**: 系统 MUST 在 Linux 主流发行版上运行（Ubuntu 22.04+、CentOS 8+、Debian 11+、Alibaba Cloud Linux 3、Rocky Linux）

#### 安全与合规

- **SC-011**: 敏感配置 MUST 不以明文存储
- **SC-012**: Tool 调用 MUST 100% 通过应用层白名单校验
- **SC-013**: 每次 Tool 调用和 LLM 调用 MUST 100% 落审计库（在响应返回前）

#### 业务可观测

- **SC-014**: 业务方 MUST 能通过审计接口查询每一次 Tool / LLM 调用的完整输入输出
- **SC-015**: 3 个验收 Demo MUST 在同一实例上全部跑通（每日天气、每日科技日报、每日 GitHub 日报）

---

## Assumptions

- 假设企业已有 K8s 集群或物理机作为部署目标
- 假设 LLM Provider 都支持 OpenAI 兼容协议（核心阶段已覆盖主流国内大模型）
- 假设业务方愿意写 markdown 配置文件（不是纯 GUI 拖拽）
- 假设单 Agent 实例的并发量在核心阶段不超过 100 Session
- 假设企业网络环境允许调用外部 LLM API（不允许的企业用自部署推理服务，FR-010 已覆盖）
- 假设 Bootstrap 文件内容由用户自行编辑，OryxOS 不验证其内容质量
- 假设单实例数据库性能满足核心阶段的存储需求（扩展阶段评估是否上分布式）
- 假设 Session 数据在单机内存储足够（跨节点 Session 同步在扩展阶段）
- **Web Service 运行在内网环境，不实现任何认证 / 授权机制**（FR-055）；如部署到公网或半信任网络，需在网关层或反向代理层补 OAuth / API Key 等认证；OryxOS 核心阶段不内置认证

---

## 验收 Demo 矩阵（v1 硬要求：3 个全部跑通）

| Demo | 验证能力 | 场景 | 验收标准 |
|------|---------|------|---------|
| Demo 一：每日天气 | US-1、US-2、US-4（内建 HTTP Tool）、定时触发 | 每天早上固定时间，Agent 自动查天气、生成穿搭建议，推送到聊天频道。**全程无人工触发**。 | 到点自动跑完整 ReAct 循环；查天气和推送各一次 HTTP 调用，都过域名白名单且都写入审计记录；`GET /api/v1/sessions/{id}` 能查到这次自动触发的完整对话记录 |
| Demo 二：每日科技日报 | US-3、US-4（第一档零代码 + 第二档 MCP）、定时触发 | 每天固定时间，Agent 汇总当日科技新闻并推送日报。**业务方不写后端代码**，只写 `AGENT.md`、公共 Skill 绑定、`mcp_servers.yaml`。Agent 应用用户之前提过的关注方向。 | 业务方全程不写代码；prompt 只出现 Skill 元数据，模型按需读取正文并完成日报 |
| Demo 三：每日 GitHub 日报 | US-3（长期记忆）、US-4（第一档 + 捆绑脚本）、定时触发 | Agent 运行捆绑的脚本产出 GitHub 趋势日报，根据用户记忆中的偏好做个性化。 | 脚本执行受路径 / Shell 白名单约束；输出按用户偏好裁剪 |

三个 Demo 跑在同一个 Agent 实例上。三个都是"钟推"（定时触发），但每个 Agent 也必须能"人推"（CLI / HTTP）调用，并且走与定时触发同一条内部执行路径。

---

## Out of Scope（非目标，本特性明确不做）

- 多租户、SSO、RBAC（扩展阶段做）
- HTTP 流式响应（SSE / WebSocket）
- IM 频道（企业微信、飞书、钉钉、Slack）—— 核心阶段仅 CLI
- 向量语义检索（长期记忆仅关键词匹配）
- 容器级沙箱（Docker / K8s pod 隔离）
- **SSRF 防护**：核心阶段假设 Web Service 部署在内网环境（FR-055），不对 HTTP Tool 的出站请求做"私有 IP / 云元数据端点 / loopback / 169.254.169.254 等 RFC1918 段"的特殊拒绝；扩展阶段做 SSRF 防护
- **HTTPS / TLS 支持（FR-038）**：核心阶段 Web Service 只提供 HTTP（明文）端点；TLS 终止假设由企业侧反向代理（Nginx / Envoy / K8s Ingress）承担。扩展阶段补 `server.ssl.*` 自动配置 + 证书管理（cert-manager / Vault 集成）。
- Provider failover、hedge racing、自动路由
- 跨区域集群、移动管理台、语音输入
- Web 仪表板 UI、Prometheus metrics 端点（扩展阶段做）

---

## 下一步

本 spec 完成后，准备进入 `/speckit.plan` 阶段，把 9 模块设计落到 `plan.md`。