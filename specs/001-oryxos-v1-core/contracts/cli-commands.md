# CLI Commands Contract — OryxOS v1.0

**Purpose**: Phase 1 接口契约——`oryxos` 12 个 CLI 子命令的接口定义
**Created**: 2026-08-07
**Feature**: [spec.md](../spec.md) / [plan.md](../plan.md)

> **实现框架**：Picocli（[research.md D9](../research.md) 决策）
> **入口约定**：所有命令通过 `java -jar oryxos.jar <command> [subcommand] [args]` 或安装后的 `oryxos <command>` 调用
> **三入口统一**：CLI / Web / Cron 三种触发源都走同一个 `AgentService.process(profile, message)`（FR-037）

---

## 顶层命令树

```
oryxos
├── init                          # 初始化工作区（生成 .oryxos/ 目录骨架）
├── chat <agent>                  # CLI Channel：跟 Agent 多轮对话
├── serve                         # 启动 Web Service（端口 8080）
├── gateway                       # 启动 AgentScheduler（cron 触发器）
├── profile                       # 子命令组：管理 Agent Profile
│   ├── list
│   ├── show <name>
│   ├── create <name>
│   └── delete <name>
├── provider                      # 子命令组：管理 LLM Provider 配置
│   ├── list
│   └── test <name>
├── tool                          # 子命令组：查看 Tool 注册表
│   └── list
├── session                       # 子命令组：管理 Session
│   ├── list
│   ├── show <id>
│   └── archive <id>
├── memory                        # 子命令组：管理长期记忆
│   ├── show
│   ├── add <text>
│   └── clear
├── status                        # 系统运行状态
└── version                       # 版本信息
```

> **12 个子命令统计**：init / chat / serve / gateway（4 个顶层）+ profile 4 + provider 2 + tool 1 + session 3 + memory 3 + status / version（2 个）= **21 个命令节点**，但顶层命令归类为 12 类。

---

## 命令详细规范

### 1. `oryxos init`

**用途**：在当前工作目录生成 `.oryxos/` 工作区骨架。

**参数**：

| 参数 | 必填 | 说明 |
|------|------|------|
| `--force` | ✗ | 强制覆盖已存在的 `.oryxos/` 目录 |

**生成内容**：
- `.oryxos/AGENTS.md`、`SOUL.md`、`USER.md`（空模板）
- `.oryxos/agents/example/AGENT.md`（示例 Agent）
- `.oryxos/skills/.gitkeep`
- `.oryxos/memory/` 目录（含空 `MEMORY.md`）
- `.oryxos/mcp_servers.yaml`（空模板）

**示例**：

```bash
oryxos init
# ✅ Created .oryxos/ structure at /home/user/projects/my-agent
# ✅ Sample agent: example
# ✅ Run 'oryxos chat example' to try

oryxos init --force
# ⚠ Overwriting existing .oryxos/ structure
# ✅ Created .oryxos/ structure
```

---

### 2. `oryxos chat <agent>`

**用途**：启动 CLI Channel，跟指定 Agent 多轮对话（对应 FR-006、FR-007）。

**参数**：

| 参数 | 必填 | 说明 |
|------|------|------|
| `<agent>` | ✓ | Agent 名（= `.oryxos/agents/<agent>/`） |
| `--user-id <id>` | ✗ | 用户标识（默认 `cli-user`，影响 Session ID 生成） |
| `--resume <sessionId>` | ✗ | 恢复已有 Session |

**交互行为**：
- 提示符：`[agent-name] > `
- 输入消息 → 调 AgentService.process(profile, message) → 输出 Agent 响应
- 内置命令：
  - `/exit` 或 `/quit` — 退出
  - `/memory` — 查看当前长期记忆
  - `/clear` — 归档当前 Session 并开新会话

**示例**：

```bash
oryxos chat daily-weather
# [daily-weather] > 杭州今天天气怎么样？
# （Agent 调用 http_get Tool 查询 → 返回响应）
# [daily-weather] > /exit

oryxos chat daily-weather --resume abc123def
# [daily-weather] > （恢复 Session abc123def）
```

---

### 3. `oryxos serve`

**用途**：启动 Web Service（[contracts/rest-api.yaml](rest-api.yaml) 的 10 个 REST 端点）。

**参数**：

| 参数 | 必填 | 说明 |
|------|------|------|
| `--port <port>` | ✗ | HTTP 端口（默认 `8080`） |
| `--host <host>` | ✗ | 绑定地址（默认 `0.0.0.0`） |

**启动行为**：
- 加载 `.oryxos/` 工作区
- 初始化 `AgentRegistry`（扫描 agents/ 目录）
- 初始化 `ProviderRegistry`（按 application.yml 显式映射）
- 启动 Spring MVC + virtual thread
- 启动 `AgentScheduler`（如果存在 cron 配置）
- 启动后台归档任务（24h Session 自动归档，FR-047）

**示例**：

```bash
oryxos serve
# ✅ OryxOS 1.0.0 started
# ✅ Listening on http://0.0.0.0:8080
# ✅ Loaded 3 agents, 2 providers

oryxos serve --port 9090
```

---

### 4. `oryxos gateway`

**用途**：仅启动 AgentScheduler（cron 触发器），不暴露 Web API。供纯调度场景使用（如部署独立调度 worker）。

**参数**：无

**启动行为**：同 `serve` 的初始化流程，但跳过 Web 启动。

**示例**：

```bash
oryxos gateway
# ✅ OryxOS Gateway started
# ✅ Scheduler online with 3 cron tasks
# ⏰ Next run: daily-weather @ 2026-08-08 08:00:00
```

---

### 5. `oryxos profile`（子命令组）

#### 5.1 `oryxos profile list`

列出所有 Profile（活跃 Agent）。

```bash
oryxos profile list
# NAME              PROVIDER    MODEL         TOOLS           SCHEDULES
# daily-weather     minimax     MiniMax-M3    4               1
# tech-daily        minimax     MiniMax-M3    6               1
```

#### 5.2 `oryxos profile show <name>`

显示 Profile 完整详情（含派生后的 system prompt 预览）。

```bash
oryxos profile show daily-weather
# name: daily-weather
# description: 每日天气推送
# provider: minimax / MiniMax-M3
# tools: [http_get, notify_slack]
# schedules: ["0 8 * * *"]
# notify_channels: [ops-slack]
# system_prompt_preview: "你是 WeatherBot，每天早上 8 点..."（前 100 字）
```

#### 5.3 `oryxos profile create <name>`

交互式创建 Profile（生成 AGENT.md 模板）。

```bash
oryxos profile create my-agent
# ? Description: <input>
# ? Provider (minimax/...): <select>
# ? Model: <input>
# ? Tools (comma-separated): <input>
# ✅ Created .oryxos/agents/my-agent/AGENT.md
```

#### 5.4 `oryxos profile delete <name>`

删除 Profile（仅删除目录，不影响 Session 历史）。

```bash
oryxos profile delete my-agent
# ⚠ Delete .oryxos/agents/my-agent/?
# y/N: y
# ✅ Deleted
```

---

### 6. `oryxos provider`（子命令组）

#### 6.1 `oryxos provider list`

列出已配置的 Provider。

```bash
oryxos provider list
# NAME        BASE URL                                  MODEL
# minimax     https://api.minimaxi.com/anthropic        MiniMax-M3
```

#### 6.2 `oryxos provider test <name>`

测试 Provider 连通性（发一个最小 prompt，验证可用）。

```bash
oryxos provider test minimax
# 🔌 Testing minimax...
# ✅ OK (response: "你好" in 412ms, 8 tokens)
```

---

### 7. `oryxos tool list`

列出所有已注册 Tool（含三档来源标记）。

```bash
oryxos tool list
# NAME                SOURCE       DESCRIPTION
# read_file           builtin      Read file content
# write_file          builtin      Write content to file
# http_get            builtin      HTTP GET request
# notify_slack        builtin      Send Slack message
# my_custom_tool      annotation   Custom @Tool annotated method
# github_create_issue mcp          Create GitHub issue via MCP
```

---

### 8. `oryxos session`（子命令组）

#### 8.1 `oryxos session list`

列出活跃 Session（不显示已归档）。

```bash
oryxos session list
# SESSION ID         PROFILE         CHANNEL    USER          LAST ACTIVE
# a1b2c3...          daily-weather   cli        cli-user      2 min ago
# d4e5f6...          tech-daily      web        alice         1 hour ago
```

#### 8.2 `oryxos session show <id>`

显示 Session 详情（含完整对话历史）。

#### 8.3 `oryxos session archive <id>`

手动归档 Session。

```bash
oryxos session archive a1b2c3
# ✅ Archived session a1b2c3
```

---

### 9. `oryxos memory`（子命令组）

#### 9.1 `oryxos memory show`

显示长期记忆内容（两分区）。

```bash
oryxos memory show
# ## 核心记忆
# - 用户偏好：只看头部客户
# - 项目背景：A 公司是 2024 Q3 启动的关键客户
#
# ## 归档记忆
# - 2024-09-15：Agent 询问过用户是否需要导出日报格式
# - 2024-10-01：用户纠正了 timezone 设置（UTC+8）
#
# Total: 247/4000 chars
```

#### 9.2 `oryxos memory add <text>`

手动写入长期记忆（追加到核心记忆区）。

```bash
oryxos memory add "用户偏好简洁回复"
# ✅ Added to core memory
```

#### 9.3 `oryxos memory clear`

清空长期记忆（需 `--confirm` 标志）。

```bash
oryxos memory clear --confirm
# ⚠ This will delete all memory. Continue? (--confirm required)
# ✅ Cleared
```

---

### 10. `oryxos status`

显示系统运行状态。

```bash
oryxos status
# OryxOS 1.0.0
# ✅ Status: running
# Agents: 3
# Providers: 1 (minimax)
# Tools: 14 (10 builtin, 2 annotation, 2 mcp)
# Active Sessions: 7
# Uptime: 2 hours 14 minutes
# Memory: 247/4000 chars
```

---

### 11. `oryxos version`

显示版本信息。

```bash
oryxos version
# OryxOS 1.0.0
# Java: 21.0.2
# Spring Boot: 3.3.4
# Build: 2026-08-07
```

---

## 全局约定

### 退出码

| 码 | 含义 |
|----|------|
| 0 | 成功 |
| 1 | 通用错误 |
| 2 | 参数错误 |
| 3 | 工作区未初始化（缺 `.oryxos/`） |
| 4 | Agent 不存在 |
| 5 | Provider 不可用 |
| 6 | Tool 不存在 |

### 错误输出格式

所有错误以 JSON Lines 格式输出到 stderr（便于脚本集成）：

```
{"level":"ERROR","code":"agent_not_found","message":"Agent 'foo' not found","request_id":"abc123"}
```

### 帮助

每个命令支持 `--help`，自动生成详细说明。

```bash
oryxos chat --help
# Usage: oryxos chat [-hr] [--user-id=<userId>] [--resume=<sessionId>] <agent>
# Start interactive CLI chat with the specified Agent.
#       <agent>                Agent name
#   -r, --resume=<sessionId>   Resume an existing session
#       --user-id=<userId>     User identifier
```

---

## 引用

- [spec.md](../spec.md) FR-006、FR-007、FR-037
- [plan.md](../plan.md)
- [research.md](../research.md) D9（Picocli 选型）
- [contracts/rest-api.yaml](rest-api.yaml)（Web Service 同源）