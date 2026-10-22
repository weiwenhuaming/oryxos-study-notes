# Data Model: OryxOS v1.0 — 实体、关系、约束

**Purpose**: Phase 1 数据模型——11 个实体的字段、关系、约束
**Created**: 2026-08-07
**Feature**: [spec.md](spec.md) / [plan.md](plan.md)

> **数据存储分层**：
> - **关系型（SQLite via Spring Data JPA）**：Session / Tool Invocation Record / LLM Call Record / Scheduled Task / Task Execution / Notify Channel
> - **文件（非结构化）**：Agent（目录）/ Profile（YAML frontmatter）/ Memory（MEMORY.md markdown）/ Bootstrap File / Skill（SKILL.md）

---

## 1. Agent（文件系统形态，不是数据库实体）

**表示形式**：`.oryxos/agents/<name>/` 目录

**目录内容**：
```
agents/<name>/
├── AGENT.md            # frontmatter (YAML) + 正文 (Markdown)
├── skills/<x>/         # 软连接到 .oryxos/skills/<x>/（可选）
└── scripts/*.py        # 第三方脚本（可选，受脚本白名单约束）
```

**AGENT.md frontmatter 字段**（YAML，snake_case）：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | ✓ | Agent 名（= 目录名） |
| `description` | string | ✓ | Agent 描述 |
| `identity.agent_name` | string | ✓ | Agent 名称 |
| `identity.prompt` | string | ✓ | 人格/系统提示词（或引用 SOUL.md） |
| `provider.name` | string | ✓ | Provider 名（minimax / deepseek / qwen / kimi / ...） |
| `provider.model` | string | ✓ | 模型名 |
| `provider.temperature` | float | ✗ | 温度参数（可选） |
| `tools` | list[string] | ✓ | 可用 Tool 名称列表 |
| `mcp_servers` | list[string] | ✗ | 引用的 MCP Server 列表（零代码扩展） |
| `channels` | list[object] | ✗ | Channel 配置（核心阶段通常空） |
| `bootstrap` | list[string] | ✗ | Bootstrap 文件列表 |
| `settings.max_iterations` | int | ✗ | ReAct 最大迭代次数（默认 10） |
| `settings.max_history_turns` | int | ✗ | 最大对话历史轮数（默认 20） |
| `schedules` | list[string] | ✗ | cron 表达式列表（定时触发） |

**关系**：
- 1 个 Agent 对应 1 个 Profile（AgentLoader.deriveProfile() 派生）
- 1 个 Agent 对应 1 个 Provider（按 provider.name）
- 1 个 Agent 对应 N 个 Tool（按 tools 列表 + mcp_servers）
- 1 个 Agent 对应 N 个 Schedule（按 schedules 列表）
- 1 个 Agent 对应 N 个 Session（运行时动态创建）

**约束**：
- 同一工作区下 Agent 目录名唯一
- Agent 目录名 = frontmatter.name
- 修改 frontmatter 后必须重启服务生效（constitution 原则 8）

---

## 2. Profile（运行时内存对象，不持久化）

**表示形式**：运行时内存对象，由 `AgentLoader.deriveProfile()` 从 AGENT.md 派生成

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | Agent 名 |
| `provider` | ProviderConfig | Provider 配置 |
| `tools` | list[string] | Tool 名列表（含内建 + MCP + @Tool） |
| `mcp_servers` | list[McpServerRef] | MCP server 引用 |
| `schedules` | list[CronExpr] | cron 表达式 |
| `settings` | Settings | max_iterations / max_history_turns |
| `systemPrompt` | string | identity.prompt + Bootstrap 文件正文 |
| `skills` | list[SkillRef] | 通过 skills/ 软连接解析出的 Skill 引用 |

**派生流程**：
```
AGENT.md (frontmatter + 正文)
    ↓ AgentLoader.deriveProfile()
Profile（运行时内存对象）
    ↓ AgentService.process(profile, message)
    ↓ ReActLoop.run(profile, prompt)
```

**约束**：
- Profile 是 immutable 对象（每次重启重新派生）
- Profile 派生失败时 Agent 启动失败并报告明确错误

---

## 3. Provider（运行时 Bean，配置驱动）

**存储形式**：`application.yml` 显式配置；不持久化到 SQLite

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | Provider 名（minimax / deepseek / qwen / kimi / ...） |
| `api_key` | string | 从环境变量解析（`${ENV_VAR}`） |
| `base_url` | string | OpenAI 兼容协议 base URL |
| `model` | string | 模型名 |

**关系**：
- 1 个 Provider 对应 1 个 Spring AI `ChatClient` Bean
- 1 个 Provider 对应 1 个 `ChatModel` 内部统一接口（`oryxos-provider` 模块）

**约束**：
- 切换 Provider MUST 仅修改一个配置字段（FR-011）
- Provider 配置变更 MUST 重启生效（FR-008）

---

## 4. Session（SQLite 表 + 内存缓存）

**用途**：用户和 Agent 一次对话的上下文容器

**表结构**（`sessions`）：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `session_id` | VARCHAR(64) | PK | 主键，channel + user + profile 联合生成（hash） |
| `profile_name` | VARCHAR(64) | NOT NULL, INDEX | 关联的 Profile 名 |
| `channel` | VARCHAR(32) | NOT NULL | 接入渠道（`cli` / `web` / `cron`） |
| `user_id` | VARCHAR(64) | NOT NULL | 用户标识 |
| `messages_json` | TEXT | NOT NULL | JSON 序列化的对话历史 |
| `status` | VARCHAR(16) | NOT NULL, DEFAULT 'active' | `active` / `archived` |
| `created_at` | TIMESTAMP | NOT NULL | 创建时间 |
| `last_active_at` | TIMESTAMP | NOT NULL, INDEX | 最后活跃时间（用于 24h 自动归档判断） |
| `archived_at` | TIMESTAMP | NULL | 归档时间 |

**关系**：
- N 个 Session 关联 1 个 Profile（多对一）
- 1 个 Session 关联 N 个 Tool Invocation Record（一对多）
- 1 个 Session 关联 N 个 LLM Call Record（一对多）

**约束**：
- session_id 唯一
- 24h 无活动自动归档（FR-047，澄清 Q2 答案 B）
- 归档后仍按 ID 可查（不在活跃列表）
- 跨重启可恢复（Edge Cases "跨重启 Session 恢复"）

**索引**：
- `idx_session_profile_status (profile_name, status)`
- `idx_session_last_active (last_active_at)`

---

## 5. Tool Invocation Record（SQLite 表，day-one 审计）

**用途**：每次 Tool 调用的完整审计记录（constitution 纠错清单第 8 条 + FR-044）

**表结构**（`tool_invocations`）：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | BIGINT | PK, AUTO_INCREMENT | 主键 |
| `session_id` | VARCHAR(64) | NOT NULL, INDEX | 关联 Session |
| `tool_name` | VARCHAR(128) | NOT NULL, INDEX | Tool 名称 |
| `input_json` | TEXT | NOT NULL | 调用参数（JSON 序列化） |
| `result_json` | TEXT | NOT NULL | 执行结果（JSON 序列化） |
| `success` | BOOLEAN | NOT NULL | 是否成功 |
| `error_message` | TEXT | NULL | 错误信息 |
| `rejection_reason` | VARCHAR(64) | NULL | Sandbox 拒绝原因 code（仅 `success=false` 时可能非空，见 FR-056 + T042）：`path_not_whitelisted` / `shell_metachar` / `ip_literal_denied` / `scheme_denied` / `whitelist_empty` 等 |
| `duration_ms` | BIGINT | NOT NULL | 执行耗时（毫秒） |
| `created_at` | TIMESTAMP | NOT NULL, INDEX | 调用时间 |

**关系**：
- N 条 Tool Invocation Record 关联 1 个 Session（多对一）
- 通过 tool_name 反查 `ToolRegistry` 可知来源（内建 / @Tool / MCP）

**约束**：
- FR-046：审计写入 MUST 在响应返回给调用方之前完成
- FR-056：失败的 Tool 调用也 MUST 记录（success=false + error_message 必填 + rejection_reason 在 Sandbox 拒绝场景下必填）

---

## 6. LLM Call Record（SQLite 表，day-one 审计）

**用途**：每次 LLM 调用的完整审计记录（FR-045）

**表结构**（`llm_calls`）：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | BIGINT | PK, AUTO_INCREMENT | 主键 |
| `session_id` | VARCHAR(64) | NOT NULL, INDEX | 关联 Session |
| `provider` | VARCHAR(64) | NOT NULL | Provider 名 |
| `model` | VARCHAR(128) | NOT NULL | 模型名 |
| `prompt_tokens` | INT | NOT NULL | 输入 token 数 |
| `completion_tokens` | INT | NOT NULL | 输出 token 数 |
| `total_tokens` | INT | NOT NULL | 总 token 数 |
| `duration_ms` | BIGINT | NOT NULL | 调用耗时（毫秒） |
| `created_at` | TIMESTAMP | NOT NULL, INDEX | 调用时间 |

**关系**：
- N 条 LLM Call Record 关联 1 个 Session（多对一）

**约束**：
- FR-046：审计写入 MUST 在响应返回给调用方之前完成
- token 数从 Spring AI 的 `ChatResponse.getMetadata()` 解析

---

## 7. Scheduled Task（SQLite 表）

**用途**：cron 表达式 + Agent 关联的定时任务

**表结构**（`scheduled_tasks`）：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | BIGINT | PK, AUTO_INCREMENT | 主键 |
| `profile_name` | VARCHAR(64) | NOT NULL, INDEX | 关联的 Profile 名 |
| `cron_expr` | VARCHAR(64) | NOT NULL | cron 表达式 |
| `enabled` | BOOLEAN | NOT NULL, DEFAULT true | 是否启用 |
| `created_at` | TIMESTAMP | NOT NULL | 创建时间 |

**关系**：
- N 个 Scheduled Task 关联 1 个 Profile

**约束**：
- Agent 启动时把 AGENT.md 的 schedules 字段同步到本表
- 同一 (profile_name, cron_expr) 唯一

---

## 8. Task Execution（SQLite 表）

**用途**：Scheduled Task 每次执行的记录（成功 / 失败 / 重试）

**表结构**（`task_executions`）：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | BIGINT | PK, AUTO_INCREMENT | 主键 |
| `scheduled_task_id` | BIGINT | NOT NULL, INDEX | 关联 Scheduled Task |
| `started_at` | TIMESTAMP | NOT NULL | 开始时间 |
| `finished_at` | TIMESTAMP | NULL | 结束时间 |
| `success` | BOOLEAN | NOT NULL | 是否成功 |
| `attempt` | INT | NOT NULL, DEFAULT 1 | 第几次尝试（1/2/3） |
| `error_message` | TEXT | NULL | 错误信息 |
| `notified` | BOOLEAN | NOT NULL, DEFAULT false | 是否已告警（全失败时） |

**约束**：
- 失败按 FR-048 重试（澄清 Q3 答案 B：30s/2min/8min，最多 3 次）
- 全失败后通过 NotifyChannel 告警 + notified=true

---

## 9. Notify Channel（SQLite 表）

**用途**：通知渠道配置（Slack / Webhook 等）

**表结构**（`notify_channels`）：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | BIGINT | PK, AUTO_INCREMENT | 主键 |
| `name` | VARCHAR(64) | NOT NULL, UNIQUE | 渠道名 |
| `type` | VARCHAR(32) | NOT NULL | `slack` / `webhook` / `feishu` / ...（核心阶段仅 slack / webhook） |
| `config_json` | TEXT | NOT NULL | 渠道配置（webhook URL / Slack token 等） |
| `enabled` | BOOLEAN | NOT NULL, DEFAULT true | 是否启用 |

**关系**：
- `AGENT.md` frontmatter 的 `notify_channels` 字段引用本表的 name（**不是**写在 frontmatter 里的硬编码渠道配置）

**约束**：
- 通知渠道走全局 `NotifyChannelAdapter` 注册表（constitution §6 易混概念）
- 核心阶段支持 slack / webhook；IM 渠道放扩展

---

## 10. Memory（文件形态，不是数据库实体）

**表示形式**：`.oryxos/memory/MEMORY.md` 单文件

**文件结构**（两分区）：
```markdown
# 长期记忆

## 核心记忆
- 用户偏好：只看头部客户
- 项目背景：A 公司是 2024 Q3 启动的关键客户

## 归档记忆
- 2024-09-15：Agent 询问过用户是否需要导出日报格式
- 2024-10-01：用户纠正了 timezone 设置（UTC+8）
```

**关键约束**：
- 文件总字数 ≤ 4000 字（FR-024）
- 超限时：保留 `## 核心记忆` 区全部 + 从 `## 归档记忆` 区按 LRU 整段丢弃（澄清 Q1 答案 B）
- 整段丢弃按 Markdown 一级或二级标题分段原子丢弃（避免半段残留）
- Agent 启动时整个文件作为长期上下文注入 system prompt（FR-023）

---

## 11. Bootstrap File（文件形态）

**表示形式**：`.oryxos/` 目录下三个 markdown 文件

**文件清单**：
- `AGENTS.md` — 项目级 agent 行为说明
- `SOUL.md` — agent 人格定义
- `USER.md` — 用户偏好

**加载时机**：Agent 启动时由 `ContextLoader` 加载到 system prompt

**约束**：
- 三个文件存在但不强制内容（OryxOS 不验证内容质量）
- 业务方通过 `AGENT.md` frontmatter 的 `bootstrap` 字段选择性加载

---

## 12. Skill（文件形态）

**表示形式**：`.oryxos/skills/<name>/SKILL.md` + 可选附属资源

**SKILL.md frontmatter 字段**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | ✓ | Skill 名 |
| `description` | string | ✓ | 一句话描述 |

**绑定方式**：Agent 目录下 `skills/<x>` 软连接到公共 Skill 库（**不**写在 AGENT.md frontmatter）

**加载机制**：
- 每轮 prompt 中 Skill 的元数据（name + description）出现
- Skill 正文不自动加载，按需通过 `read_file` Tool 读取（渐进式披露 L1/L2）

---

## 实体关系图

```text
Agent (filesystem)
   │ 1:1
   ▼
Profile (memory)
   │ N
   ├────────────────┐
   ▼                ▼
Session (sqlite)   Scheduled Task (sqlite)
   │ 1               │ 1
   │ N               │ N
   ▼                 ▼
Tool Invocation    Task Execution
   Record (sqlite)
   │
   │ 1
   ▼
LLM Call Record
   (sqlite)

Notify Channel (sqlite) — 全局注册表，所有 Agent 通过 name 引用
Memory (file) — 全局单文件，跨 Agent 共享
Bootstrap File (file) — 全局三个文件
Skill (file) — 公共库 + Agent 软连接
```

---

## 引用

- [spec.md](spec.md)
- [plan.md](plan.md)
- [research.md](research.md)
- [contracts/agent-profile-schema.json](contracts/agent-profile-schema.json)
- [contracts/mcp-servers-schema.json](contracts/mcp-servers-schema.json)