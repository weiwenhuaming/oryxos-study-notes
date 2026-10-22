# Quickstart: OryxOS v1.0 — 3 个验收 Demo 验证步骤

**Purpose**: Phase 1 Demo 验证手册——核心阶段必须跑通的 3 个 Demo 完整步骤
**Created**: 2026-08-07
**Feature**: [spec.md](spec.md) / [plan.md](plan.md)

> **Demo 是验收门槛**（constitution §3 原则 7："每个 user story 完成后有可演示 Demo"）。
> 3 个 Demo 全部跑通才能进入扩展阶段。

---

## Demo 总览

| Demo | 演示形态 | 验证能力 | 对应 US |
|------|---------|---------|---------|
| **Demo 一：每日天气** | 光杆 `AGENT.md`，每天 8 点自动跑 | LLM + ReAct + 内建 HTTP Tool + Notify + AgentScheduler | US-1 + US-2 + US-5 |
| **Demo 二：每日科技日报** | `AGENT.md` + `skills/` 软连接 + MCP | 三层记忆 + Plugin Tool 第二档 MCP + Skill 渐进披露 | US-3 + US-4 + US-5 |
| **Demo 三：每日 GitHub 日报** | `AGENT.md` + `scripts/github_trending.py` | shell 跑脚本 + Memory 偏好 + 定时 + Notify | US-3 + US-4 + US-5 |

---

## Demo 一：每日天气推送（Hello World of OryxOS）

### 演示目标

一个**没有任何 Skill、没有 MCP、没有任何脚本**的"光杆"Agent，每天早上 8 点自动查询指定城市天气并通过 Slack 通知给用户。

**核心能力验证**：
- ✅ LLM Provider 切换（US-1）
- ✅ ReAct 循环自主调用 Tool（US-2）
- ✅ 内建 HTTP Tool + Notify Tool（US-2 / US-4）
- ✅ AgentScheduler cron 触发（US-2 / US-5）
- ✅ Web Service 对外提供查询（US-5）

### 前置准备

#### 1. 环境变量

```bash
export MINIMAX_API_KEY="eyJhbGciOi..."        # MiniMax（用户唯一持有的 key）
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."
```

#### 2. application.yml（简化版）

```yaml
spring:
  ai:
    chat:
      client:
        enabled: true
    tool:
      execution:
        enabled: false    # ⚠️ 禁用 Spring AI 自动 tool 执行
    model:
      tool-calling:
        auto: false

oryxos:
  providers:
    minimax:
      api-key: ${MINIMAX_API_KEY}
      base-url: https://api.minimaxi.com/anthropic
      model: MiniMax-M3
  web:
    port: 8080
  notify:
    channels:
      - name: ops-slack
        type: webhook
        url: ${SLACK_WEBHOOK_URL}
```

### 步骤 1：初始化工作区

```bash
mkdir weather-demo && cd weather-demo
oryxos init
```

生成结构：
```
.oryxos/
├── AGENTS.md
├── SOUL.md
├── USER.md
├── agents/example/AGENT.md      # 示例 Agent（可删除）
├── memory/MEMORY.md
└── mcp_servers.yaml
```

### 步骤 2：创建 daily-weather Agent

```bash
oryxos profile create daily-weather
# 交互填入：
#   Description: 每日天气推送
#   Provider: minimax
#   Model: MiniMax-M3
#   Tools: http_get, notify_slack
```

或手动编辑 `.oryxos/agents/daily-weather/AGENT.md`（注意：v1.0-core Profile frontmatter 不包含 `identity.*` / `notify_channels` 字段——`name` 用顶层字段、`prompt` 直接写 body、通知渠道走全局 SQLite `notify_channels` 表，不写在 AGENT.md）：

```markdown
---
name: daily-weather
description: 每日天气推送
provider:
  name: minimax
  model: MiniMax-M3
  temperature: 0.7
tools:
  - http_get
  - notify_slack
settings:
  max_iterations: 5
  max_history_turns: 10
schedules:
  - "0 8 * * *"
---

# 你是 WeatherBot

你的任务是：根据用户提供（或默认）的城市，调用 http_get 工具查询天气，
并通过 notify_slack 工具把结果推送到 Slack。
风格简洁，给一句话温度 + 简短建议。

# 任务指令

1. 询问用户城市（如果未指定）
2. 调用 http_get 查询天气（使用天气 API，如 Open-Meteo）
3. 把结果用一句话总结
4. 调用 notify_slack 推送到 ops-slack 渠道（渠道名通过 SQLite notify_channels 表配置，不写在 AGENT.md）
```

### 步骤 3：CLI 试跑

```bash
oryxos chat daily-weather
# [daily-weather] > 杭州今天天气怎么样？
# （Agent 思考 → 调 http_get → 拿到数据 → 调 notify_slack → 回答用户）
# [WeatherBot] 已查询杭州天气：晴 26°C，微风。建议穿短袖。
# [daily-weather] > /exit
```

**预期**：Agent 自主完成 http_get + notify_slack 两次 Tool 调用；Slack 收到推送；tool_invocations 表写入 2 条记录。

### 步骤 4：Web Service 验证

```bash
oryxos serve
# ✅ Started on http://0.0.0.0:8080
```

测试 12 个端点（v1.0-core 实际暴露 12 个 REST 端点，含 AuditController for FR-050）：

```bash
# 1. 健康检查
curl http://localhost:8080/api/v1/health
# {"status":"UP",...}

# 2. 创建 Session
curl -X POST http://localhost:8080/api/v1/sessions \
  -H "Content-Type: application/json" \
  -d '{"profile":"daily-weather","channel":"web","user_id":"alice"}'
# {"session_id":"a1b2c3...",...}

# 3. 发消息
curl -X POST http://localhost:8080/api/v1/sessions/a1b2c3.../messages \
  -H "Content-Type: application/json" \
  -d '{"content":"上海今天天气怎么样？"}'
# {"content":"已查询上海天气...","tool_calls":[...],"iterations":2}

# 4. 列出 Profile
curl http://localhost:8080/api/v1/profiles
# [{"name":"daily-weather",...}]

# 5. 列出 Tool
curl http://localhost:8080/api/v1/tools
# [{"name":"http_get","source":"builtin",...},{"name":"notify_slack","source":"builtin",...}]
```

### 步骤 5：Scheduler 验证

```bash
# 修改 schedules 到下一分钟
vim .oryxos/agents/daily-weather/AGENT.md
# schedules: ["<next-minute> * * * *"]  # 改成当前分钟 + 1

oryxos gateway &
# ⏰ Scheduler online with 1 cron task
# ⏰ Next run: daily-weather @ 2026-08-07 14:32:00

# 等到时间，Agent 自动触发，Slack 收到推送
# 检查 scheduled_tasks + task_executions 表
sqlite3 .oryxos/oryxos.db "SELECT * FROM task_executions ORDER BY id DESC LIMIT 1"
# 1|<scheduled_task_id>|2026-08-07 14:32:00|2026-08-07 14:32:05|1|1||0
```

### 验收 checklist

- [x] Agent 通过 CLI 多轮对话调用 http_get + notify_slack Tool
- [x] 同一 Agent 通过 REST API 调用，结果一致
- [x] cron 表达式触发自动执行，无需人工干预
- [x] 每次 Tool 调用在 tool_invocations 表留下记录
- [x] 切换 Provider（minimax → 其它）只改一个字段，重启生效（FR-011）

### 涉及验收场景

- **FR-001**：Profile YAML 定义加载生效
- **FR-002**：LLM 调用成功
- **FR-006**：CLI Channel 交互
- **FR-013~017**：ReAct 循环 + Tool 调用
- **FR-014**：cron 触发
- **FR-021**：Tool 白名单校验
- **FR-034**：notify_channels 引用全局配置
- **FR-036**：Web API 端点
- **FR-044**：tool_invocations 落库

---

## Demo 二：每日科技日报（MCP + Skill 渐进披露）

### 演示目标

Agent 通过 **MCP server** 拉取 Hacker News / 36Kr 的当天科技要闻，结合公共 Skill（"科技日报模板"），生成本地摘要并推送给用户。

**核心能力验证**：
- ✅ Plugin Tool 第二档：MCP（US-4）
- ✅ Skill 渐进式披露 L1/L2（US-4）
- ✅ 三层记忆：记住用户偏好（如"只关心 AI 新闻"）（US-3）
- ✅ 复杂多 Tool 协作（US-2）
- ✅ Web Service + 定时（US-5）

### 前置准备

#### 1. MCP server 安装

使用社区的 Hacker News MCP server（stdio 模式）：

```bash
npm install -g @modelcontextprotocol/server-hacker-news
```

#### 2. mcp_servers.yaml

`.oryxos/mcp_servers.yaml`：

```yaml
servers:
  - name: hacker-news-mcp
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-hacker-news"]
    timeout_ms: 3000
    enabled: true
```

#### 3. 公共 Skill

`.oryxos/skills/tech-daily-template/SKILL.md`：

```markdown
---
name: tech-daily-template
description: 科技日报标准格式（每条新闻 1 行：标题 + 来源 + 1 句话总结）
---

# 科技日报模板

输出格式：
- **[标题]**（来源）
  一句话总结。

筛选标准：
- 优先 AI / 大模型 / 开发者工具
- 每条不超过 80 字
- 总数控制在 10 条以内
```

### 步骤 1：创建 tech-daily Agent

`.oryxos/agents/tech-daily/AGENT.md`：

```markdown
---
name: tech-daily
description: 每日科技日报
provider:
  name: minimax
  model: MiniMax-M3
  temperature: 0.5
tools:
  - notify_slack
  - save_memory
  - recall_memory
mcp_servers:
  - hacker-news-mcp
settings:
  max_iterations: 15
  max_history_turns: 5
schedules:
  - "0 9 * * 1-5"
---

# 你是 TechDailyBot

你每天早上 9 点为用户整理科技要闻。
先调用 hacker_news 工具获取当天 Top 30 故事，
再按 tech-daily-template Skill 的格式筛选并摘要，
最后通过 notify_slack 推送到用户。

# 任务指令

1. 用 `recall_memory` 工具检查用户偏好（如 "只看 AI 新闻"）
2. 调用 hacker_news MCP 工具获取 Top 30 当天故事
3. 按用户偏好过滤（如有）+ Skill 模板格式筛选 10 条
4. 用 `save_memory` 工具记录今天的判断标准（用于明天参考）
5. 调用 notify_slack 推送日报

# Bootstrap

USER.md / SOUL.md / AGENTS.md 通过 ContextLoader 自动加载到 system prompt 前缀，
不写在 AGENT.md frontmatter。
```

#### 1.1 软连接公共 Skill

```bash
mkdir -p .oryxos/agents/tech-daily/skills
ln -s ../../../skills/tech-daily-template .oryxos/agents/tech-daily/skills/tech-daily-template
ls -la .oryxos/agents/tech-daily/skills/
# tech-daily-template -> ../../../skills/tech-daily-template
```

### 步骤 2：CLI 试跑（验证 MCP + Skill）

```bash
oryxos chat tech-daily
# [tech-daily] > 今天的科技新闻有哪些？
# （Agent 调 recall_memory → 拿到偏好 → 调 hacker_news MCP 工具 → 拉 SKILL.md 内容 → 摘要 → 调 notify_slack）
# [TechDailyBot] 今日科技日报：
# - **OpenAI 发布 GPT-5**（Hacker News）
#   多模态能力大幅提升...
# - **Anthropic 推出新工具**（Hacker News）
#   ...

# [tech-daily] > 我只想看 AI 新闻
# [TechDailyBot] 已记住。明天起只推送 AI 相关。
# → save_memory("用户偏好：只看 AI 新闻")

# [tech-daily] > /exit
```

### 步骤 3：验证记忆跨 Session

```bash
# 新开 Session
oryxos chat tech-daily
# [tech-daily] > 今天有什么 AI 新闻？
# （新一轮对话，system prompt 已经注入"用户偏好：只看 AI 新闻"）
# [TechDailyBot]（只筛选 AI 相关条目）
```

### 步骤 4：Skill 渐进披露验证

观察日志（`.oryxos/logs/oryxos.log`）：

```json
{"ts":"...","level":"INFO","event":"skill_metadata_injected","agent":"tech-daily","skills":["tech-daily-template"]}
{"ts":"...","level":"INFO","event":"skill_body_loaded","agent":"tech-daily","skill":"tech-daily-template","trigger":"read_file"}
```

**预期**：Skill 元数据（name + description）在 Agent 启动时即注入 system prompt；正文（SKILL.md body）只在 Agent 调 `read_file` 工具时才加载。

### 步骤 5：MCP 启动行为验证

如果 MCP server 启动失败（故意改错命令）：

```yaml
# mcp_servers.yaml
servers:
  - name: hacker-news-mcp
    command: nonexistent-command
    timeout_ms: 3000
```

```bash
oryxos serve
# ⚠ Failed to connect to MCP server 'hacker-news-mcp' (timeout 3000ms): spawn nonexistent-command ENOENT
# ⚠ Agent 'tech-daily' references unavailable MCP server 'hacker-news-mcp', affected tools will be unavailable at call time
# ✅ Server continues startup
```

**预期**：3s 超时后仅警告，不阻塞启动；Agent 启动；调用失败时 Tool 不可用。

### 验收 checklist

- [x] MCP server 启动并暴露 Tool 到 ToolRegistry
- [x] Agent 通过 MCP Tool 拉取真实数据
- [x] Skill 元数据在启动时注入，正文按需加载
- [x] save_memory / recall_memory 跨 Session 保留偏好
- [x] MEMORY.md 超 4000 字按 FR-024 策略截断
- [x] 24h 无活动 Session 自动归档（FR-047）
- [x] MCP 启动失败不阻塞系统启动（FR-049）

### 涉及验收场景

- **FR-018~020**：三层记忆 + MEMORY.md 文件
- **FR-024**：超限截断策略
- **FR-029**：Skill 渐进披露
- **FR-030**：Plugin Tool 第二档 MCP
- **FR-032**：MCP 工具透明性（ReAct 不感知来源）
- **FR-049**：MCP 启动行为
- **FR-047**：Session 24h 自动归档

---

## Demo 三：每日 GitHub 日报（脚本白名单 + Memory 偏好）

### 演示目标

Agent 调用**已审核的第三方 Python 脚本**（`github_trending.py`）获取 GitHub Trending 项目，结合 Memory 里的用户偏好（语言/领域）筛选后推送。

**核心能力验证**：
- ✅ 内建 `shell` Tool 跑第三方脚本（US-2）
- ✅ Sandbox 白名单校验（US-4）
- ✅ Memory 偏好影响决策（US-3）
- ✅ 完整 cron + Notify 流程（US-2 / US-5）

### 前置准备

#### 1. 第三方脚本（信任边界扩大，需 Sandbox 校验）

`.oryxos/agents/github-daily/scripts/github_trending.py`：

```python
#!/usr/bin/env python3
"""GitHub Trending fetcher."""
import json
import sys
import urllib.request


def fetch(language: str = "") -> list[dict]:
    url = f"https://github.com/trending/{language}"
    # 实际实现：解析 HTML 返回 [{name, description, stars}, ...]
    return [{"name": "owner/repo", "description": "...", "stars": 1234}]


if __name__ == "__main__":
    lang = sys.argv[1] if len(sys.argv) > 1 else ""
    print(json.dumps(fetch(lang), ensure_ascii=False))
```

#### 2. application.yml 沙箱白名单

```yaml
oryxos:
  sandbox:
    shell:
      whitelist: [python, ls, cat, echo, grep, awk, sed, find]
      script_paths:
        - ${ORYXOS_HOME}/agents/*/scripts/*.py
```

### 步骤 1：创建 github-daily Agent

`.oryxos/agents/github-daily/AGENT.md`：

```markdown
---
name: github-daily
description: 每日 GitHub Trending 日报
provider:
  name: minimax
  model: MiniMax-M3
  temperature: 0.5
tools:
  - shell
  - notify_slack
  - save_memory
  - recall_memory
settings:
  max_iterations: 8
  max_history_turns: 20
schedules:
  - "0 10 * * *"
---

# 你是 GitHubDailyBot

每天早上 10 点向用户推送 GitHub Trending Top 10 项目。

# 任务指令

1. `recall_memory("语言偏好")` 拿到用户关注的语言（默认 Python）
2. `shell python scripts/github_trending.py <语言>` 拿到 Trending 列表
3. 按 stars 排序，筛选 Top 10
4. 用 `save_memory` 记录今天推荐过哪些（明天去重）
5. `notify_slack` 推送（渠道走 SQLite notify_channels 表）
```

### 步骤 2：试跑 + Sandbox 验证

```bash
oryxos chat github-daily
# [github-daily] > 跑一下今天的 Trending

# Agent 第一次尝试（合法）：
# [GitHubDailyBot] 我会调用 shell 工具执行脚本...
# → shell("python scripts/github_trending.py python")
# → Sandbox 校验：路径在 .oryxos/agents/*/scripts/ 白名单内 → 放行
# → 脚本执行成功，返回 JSON

# Agent 尝试调用未授权命令（演示拦截）：
# （手动修改 Profile 加危险命令，或在对话中诱导）
# → shell("rm -rf /")
# → Sandbox 拒绝：'rm' 不在白名单
# → Agent 收到错误："Tool execution denied: command 'rm' not in whitelist"
```

### 验收 checklist

- [x] shell Tool 调用白名单内的脚本成功
- [x] shell Tool 调用白名单外的命令被拒绝
- [x] 脚本路径在白名单前缀内放行
- [x] 跨 Session 保留语言偏好（"我只看 Python"）
- [x] cron 每日 10 点自动跑
- [x] 全流程无 LLM 误调 Tool 之外的系统命令

### 涉及验收场景

- **FR-013~017**：ReAct + Tool 调用
- **FR-020**：Memory 偏好影响决策
- **FR-021**：Sandbox 白名单（路径 + 命令）
- **FR-027**：脚本白名单
- **FR-028**：安全失败语义（拒绝时返回明确错误）
- **SC-012**：Tool 调用 100% 白名单校验

---

## Demo 跑通判定（发版门槛）

3 个 Demo 全部满足对应 checklist 后，核心阶段验收完成；可以：

1. 运行 `/speckit.analyze` 检查 spec 一致性（**不能省**）
2. git commit 标记 `feat: complete OryxOS v1.0 core phase`
3. 进入扩展阶段规划（治理层 / 向量检索 / 多实例集群等）

---

## 引用

- [spec.md](spec.md) — FR-001~056 + SC-001~015
- [plan.md](plan.md)
- [data-model.md](data-model.md) — 11 个实体
- [research.md](research.md) — 15 个技术决策
- [contracts/rest-api.yaml](contracts/rest-api.yaml)
- [contracts/agent-profile-schema.json](contracts/agent-profile-schema.json)
- [contracts/cli-commands.md](contracts/cli-commands.md)
- [contracts/mcp-servers-schema.json](contracts/mcp-servers-schema.json)
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — constitution v1.0.0