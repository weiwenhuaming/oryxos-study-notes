# Research: OryxOS v1.0 关键技术决策记录

**Purpose**: Phase 0 设计期调研产出——记录关键技术决策的 Decision / Rationale / Alternatives considered
**Created**: 2026-08-07
**Feature**: [spec.md](spec.md) / [plan.md](plan.md)

> 形式约束（按 SKILL.md）：每条决策含 `Decision`（选了什么）、`Rationale`（为什么）、`Alternatives considered`（其他选项及放弃原因）。

---

## D1. JDK 版本与并发模型

**Decision**: **JDK 21 + 同步阻塞 + virtual thread（不上响应式）**

**Rationale**:
- constitution §3 原则 1 锁定 JDK 21（virtual thread 是 first-class 需求）
- 同步阻塞 + virtual thread 模式让代码可读性接近传统 Servlet，对 Java 团队门槛最低
- virtual thread（Project Loom）让"每个请求一个线程"模型能撑住 100 并发 Session（SC-002）而无需反应式改造

**Alternatives considered**:
- **JDK 17**: 放弃。virtual thread 不可用，回归到"线程池 + 阻塞队列"的资源受限模式
- **JDK 21 + Reactor/WebFlux**: 放弃。Reactive 编程范式门槛高；Java 生态对企业团队的熟悉度低；constitution §3 原则 1 明确"同步阻塞"
- **JDK 21 + 响应式 R2DBC（取代 JDBC）**: 放弃。R2DBC 生态成熟度不够；审计查询 / Session 持久化用同步 JDBC + SQLite 更简单

---

## D2. Spring AI 用法边界

**Decision**: **只用 Spring AI 的 Provider 抽象 + 协议转换 + `@Tool` schema 生成；**禁用**自动 tool 执行**

**Rationale**:
- constitution §3 原则 4 明文规定（"最易写错的一条"）
- 自动 tool 执行会让 Spring AI 自己调一次 Tool，再让我们的 `ReActLoop` 调一次，结果是 tool 被调两次且审计落库漏一
- 业务正确性 vs 框架便利性：**业务正确性优先**——这是 day-one 审计可追溯的硬约束

**具体禁用配置**（`application.yml`）:

```yaml
spring:
  ai:
    # 启用：ChatClient / ChatModel / @Tool schema 生成
    # 禁用：tool execution auto-config
    chat:
      client:
        enabled: true
    # 关键：以下两个必须显式关闭
    tool:
      execution:
        enabled: false   # 禁用自动 tool 执行
    model:
      tool-calling:
        auto: false       # 禁用模型自动触发 tool 调用
```

**Alternatives considered**:
- **完全不用 Spring AI**: 放弃。每个 LLM Provider 都要自己实现 OpenAI 兼容协议适配器 + 重试 / 限流 / token 计数，工作量翻 3 倍
- **用 Spring AI Agent 抽象**: 放弃。constitution §3 原则 3 明文"自实现 ReAct"；Agent 抽象无法做到 FR-013~017 的可定制（人推/钟推复用、循环中断、审计埋点）
- **Spring AI 完整启用**: 放弃。constitution §3 原则 4 明文禁止

---

## D3. Maven 模块结构（9 模块固定）

**Decision**: **9 个 Maven 模块固定不变**（constitution §4 + TechnicalSolution §10）

**Rationale**:
- constitution §3 原则 1 锁定 9 模块
- 模块边界跟依赖图严格对齐，避免"循环依赖"或"业务模块依赖核心模块"
- `oryxos-tool` 三合一（内建 + MCP + Sandbox）避免旧设计的"builtin / skill / mcp 三模块"拆分——后者导致 Tool 接口碎片化

**Alternatives considered**:
- **拆分 Tool 为 builtin / skill / mcp 三模块**: 放弃（constitution §3 原则 4 + TechnicalSolution §10 明确"三合一"）
- **抽离 common 模块**: 放弃。跨模块共享的接口放 `oryxos-core`（已经做了），不需要单独 common 模块
- **拆出 security 模块**: 放弃。Sandbox 校验放在 `WhitelistSandbox` 实现类里，归 `oryxos-tool`；单独 security 模块会让依赖图复杂

---

## D4. ReAct 循环自实现位置与依赖

**Decision**: **`ReActLoop` 位于 `oryxos-core` 模块**，自实现约数十行 Java；**不**依赖 Spring AI Agent 抽象

**Rationale**:
- constitution §3 原则 3 明文规定
- `ReActLoop` 是"上接 AgentService、下接 ToolExecutor"的纯业务循环，**不**需要 Spring AI 任何抽象
- 核心算法（FR-013~017）：
  ```java
  while (iteration < maxIterations) {
      LlmResponse resp = providerService.call(profile, prompt);
      if (!resp.hasToolCalls()) return resp;
      for (ToolCall call : resp.toolCalls()) {
          ToolResult result = toolExecutor.execute(profile, call);
          // 追加到对话历史
      }
      iteration++;
  }
  ```

**Alternatives considered**:
- **用 Spring AI AgentExecutor**: 放弃（constitution §3 原则 3 + 无法做到自定义审计埋点）
- **放 `oryxos-tool` 模块**: 放弃。ReAct 循环是核心控制流，不是 Tool 实现；放错地方会让 `oryxos-core` 依赖 `oryxos-tool` 形成循环
- **引入 LangChain4j AgentExecutor**: 放弃。多一层抽象反而难定制；多一个依赖

---

## D5. Provider 显式 name → ChatModel 映射

**Decision**: **`ProviderRegistry` 维护显式 map**：`provider name (minimax/qwen/...) → ChatModel Bean`；**不**用类路径扫描

**Rationale**:
- constitution §3 纠错清单第 5 条明文规定
- 显式映射让"添加一个 Provider = 改一行配置"成为可能（FR-011），业务方不必重新编译
- 类路径扫描会扫到所有 `@Component` Bean，启动慢且不稳定

**实现示例**（`application.yml`）:
```yaml
oryxos:
  providers:
    minimax:
      api-key: ${MINIMAX_API_KEY}
      base-url: https://api.minimaxi.com/anthropic
      model: MiniMax-M3
    qwen:
      api-key: ${QWEN_API_KEY}
      base-url: https://dashscope.aliyuncs.com/compatible-mode
      model: qwen-plus
```

**Alternatives considered**:
- **类路径扫描 `@Component`**: 放弃（constitution 纠错清单第 5 条）
- **ServiceLoader SPI**: 放弃。需要业务方写 `META-INF/services` 文件，门槛高；显式 YAML 配置更直观
- **运行时反射调用**: 放弃。反射 + 私有构造器路径不可控，安全风险

---

## D6. MemoryService 三层统一门面

**Decision**: **`MemoryService` 是统一门面**，对外只暴露 `add()` / `recall()` / `getContext()` 三个方法，内部委托给会话层 + 长期层

**Rationale**:
- constitution §3 纠错清单第 7 条明文规定（不能跟 Session 合并）
- "门面"模式让 `ReActLoop` 只跟 `MemoryService` 交互，未来切换底层实现（向量库 / 分布式）不影响上游
- 长期记忆用 `.oryxos/memory/MEMORY.md` 文件（FR-019/020）；超 4000 字按 FR-024 策略截断（澄清 Q1 答案 B）

**关键接口**:
```java
public interface MemoryService {
    void add(String content);            // 写入长期记忆（追加到核心记忆区）
    List<String> recall(String query);   // 按关键词检索（核心记忆区优先 + 归档区 fallback）
    String getContextForPrompt();        // 启动时注入 system prompt（按分区+LRU 截断后）
}
```

**Alternatives considered**:
- **MemoryService 直接继承 SessionManager**: 放弃（constitution 纠错清单第 7 条；违反单一职责）
- **Memory 直接是 JPA Entity**: 放弃。长期记忆是文件，不是结构化数据
- **Memory 走向量库**: 放弃。constitution §3 原则 6 明确"向量检索放扩展阶段"；澄清 Q1 答案 B 是关键词匹配 + 分区 LRU

---

## D7. MCP 集成策略

**Decision**: **MCP Java SDK + stdio 优先；SSE 放扩展阶段**

**Rationale**:
- MCP 生态在 Java 侧成熟度不如 Python，stdio 模式（本地进程通信）实现简单、延迟低、调试容易
- SSE 需要 HTTP 双向流，企业 K8s 部署的 Ingress / 网关经常不支持，扩展阶段再补
- Agent 启动时对每个 MCP server 尝试 3s 超时快速连接（FR-049，澄清 Q4 答案 C）

**Alternatives considered**:
- **MCP SSE 优先**: 放弃。Java 生态成熟度不够；扩展阶段再补
- **自实现 MCP 协议**: 放弃。spec-Kit 时间约束 + MCP Java SDK 已覆盖核心功能
- **不用 MCP**: 放弃。Plugin Tool 第二档依赖 MCP（FR-030），业务方轻代码扩展路径必须有

---

## D8. 持久化层选型

**Decision**: **SQLite + Spring Data JPA**（关系数据） + **`.oryxos/memory/MEMORY.md` 文件**（长期记忆）

**Rationale**:
- constitution §3 原则 6 锁定
- SQLite 嵌入式 C 实现、单文件部署、零运维，对核心阶段单实例部署最合适
- JPA 让实体声明式映射，开发效率高
- 审计表（`tool_invocations` / `llm_calls`）day-one 落库（FR-044/045/046）

**SQLite 库表清单**（详见 [data-model.md](data-model.md)）:
- `sessions` — 会话元数据 + 对话历史
- `tool_invocations` — Tool 调用审计（day-one）
- `llm_calls` — LLM 调用审计（day-one）
- `scheduled_tasks` — cron 触发任务
- `task_executions` — 任务执行结果（成功/失败/重试）
- `notify_channels` — 通知渠道配置

**Alternatives considered**:
- **H2 内存数据库**: 放弃。重启数据丢失；不满足 SC-005"持久化 Session 数据保证不丢"
- **PostgreSQL**: 放弃。核心阶段单实例部署，PG 运维复杂度超出范围
- **纯文件存储（JSON）**: 放弃。并发 Session 写入性能差；缺索引；不利于 SC-002 "100 并发无降级"
- **MyBatis**: 放弃。JPA 对声明式实体映射更友好；MyBatis 需要手写 SQL，开发效率低

---

## D9. CLI 框架

**Decision**: **Picocli**

**Rationale**:
- 注解驱动，代码紧凑
- 内置子命令支持（`oryxos profile create/list/show/delete`）
- 自动 `--help` 生成 + 错误提示友好
- 跟 Spring Boot 集成有现成模式（`PicocliSpringFactory`）

**Alternatives considered**:
- **Spring Shell**: 放弃。对 12 个子命令 + 嵌套子命令组的场景，Picocli 注解比 Spring Shell 命令类更简洁
- **JCommander**: 放弃。社区活跃度下降；新项目不建议用

---

## D10. Web 框架

**Decision**: **Spring MVC（同步阻塞）+ virtual thread**

**Rationale**:
- constitution §3 原则 1 锁定（"不上 WebFlux"）
- 同步阻塞代码对企业团队门槛最低
- Spring MVC + JDK 21 virtual thread 在 Spring Boot 3.2+ 已 GA，性能可与响应式媲美
- 10 个 REST 端点用 6 个 `@RestController` 自然分组（FR-036）

**Alternatives considered**:
- **Spring WebFlux / Reactor**: 放弃（constitution §3 原则 1；企业团队门槛高）
- **JAX-RS / Resteasy**: 放弃。Spring 生态一致性好，JAX-RS 增加认知负担

---

## D11. 内建 Tool 集合

**Decision**: **8 个内建 Tool**（read_file / write_file / list_dir / shell / http_get / http_post / save_memory / recall_memory）+ 通知 Tool（notify_slack / notify_webhook）

**Rationale**:
- 文件 / Shell / HTTP 三个覆盖"操作系统调用"基本盘
- save_memory / recall_memory 来自 US-3，**必须**是内建 Tool（业务方无法用零代码扩展自己的 Memory Tool）
- 通知 Tool 来自 TechnicalSolution §6.8，Demo 一"每日天气"用 Slack / Webhook 推送需要

**Alternatives considered**:
- **只 4 个 Tool（read_file / shell / http_get / save_memory）**: 放弃。write_file / list_dir / http_post / recall_memory 在多个 Demo 中需要
- **更多 Tool（如 git / docker / database）**: 放弃。核心阶段 4 周时间约束；超范围 Tool 放扩展阶段

---

## D12. Sandbox 安全策略

**Decision**: **应用层白名单校验**（`WhitelistSandbox` 实现）

**Rationale**:
- 核心阶段时间约束，应用层白名单实现成本最低
- Java SecurityManager 在 JDK 17 起已废弃，JDK 21 不可用
- 容器级沙箱（Docker / K8s pod）放扩展阶段

**三类白名单**:
- **路径白名单**：`read_file` / `write_file` / `list_dir` 接收的路径必须以白名单前缀开头（默认 `.oryxos/` + 显式配置目录）
- **Shell 命令白名单**：`shell` Tool 执行的命令必须在白名单内（默认 `ls / cat / grep / awk / sed / find / echo` 等安全命令）
- **HTTP 域名白名单**：`http_get` / `http_post` 目标 URL 的域名必须在白名单内（默认配置 + 业务 Profile 可覆盖）

**Alternatives considered**:
- **Java SecurityManager**: 放弃（JDK 21 不可用）
- **Docker 容器沙箱**: 放弃（扩展阶段）
- **WebAssembly 沙箱**: 放弃（生态成熟度不够；扩展阶段）

---

## D13. 调度策略

**Decision**: **`AgentScheduler` 用 cron 表达式触发**；失败按 FR-048 重试（澄清 Q3 答案 B：30s/2min/8min 指数退避 3 次）

**Rationale**:
- 3 个 Demo 全部是定时触发（"每日 X"），必须有 cron 能力
- cron 表达式业务方熟悉度高，比"每 N 分钟"灵活
- 失败重试不阻塞下一次周期（FR-048）——错过本次周期不影响下一次按计划触发

**Alternatives considered**:
- **Quartz Scheduler**: 放弃。功能强大但复杂度高；核心阶段用 JDK 自带 `ScheduledExecutorService` + cron 解析（cron-utils）足够
- **每 N 分钟触发**: 放弃。cron 灵活性更高
- **失败立即重试 + 阻塞下次周期**: 放弃。"早 8 点日报"任务失败不应阻塞"早 9 点"其他任务

---

## D14. Profile / AGENT.md 配置格式

**Decision**: **YAML frontmatter + Markdown 正文**；YAML 用 SnakeYAML 解析

**Rationale**:
- AGENT.md 是 markdown 文件，frontmatter 是 YAML 是社区事实标准（GitHub Issues / Hugo / Jekyll 等都用）
- 业务方熟悉度高；写作工具支持好（VSCode 插件、Typora 等）
- 字段名 snake_case（constitution §10 规范）

**Alternatives considered**:
- **纯 JSON**: 放弃。JSON 不支持注释；业务方写 profile 时注释很重要
- **TOML**: 放弃。Java 生态对 TOML 支持弱
- **HOCON**: 放弃。Spring 支持但社区熟悉度低

---

## D15. 配置与密钥加载

**Decision**: **`${ENV_VAR}` 占位引用** + 环境变量注入；**禁止**明文 key 写在配置中

**Rationale**:
- constitution §3 原则 6 + SC-011 "敏感配置不以明文存储"
- `${ENV_VAR}` 在 YAML 解析阶段替换，业务方 Profile 里只写引用
- 扩展阶段对接 KMS / Vault，核心阶段环境变量足够

**Alternatives considered**:
- **加密配置文件（jasypt）**: 放弃。密钥仍然要存在某处；增加部署复杂度；扩展阶段再做
- **从 Vault 读取**: 放弃。核心阶段不引入额外基础设施依赖
- **业务方明文写在 Profile**: 放弃（SC-011 禁止）

---

## 决策总览

| # | 主题 | 决策 |
|---|------|------|
| D1 | JDK + 并发 | JDK 21 + 同步阻塞 + virtual thread |
| D2 | Spring AI 边界 | Provider + @Tool schema；**禁用**自动 tool 执行 |
| D3 | 模块结构 | 9 模块固定 |
| D4 | ReAct 实现 | `oryxos-core` 自实现；不依赖 Spring AI Agent |
| D5 | Provider 映射 | 显式 `name → ChatModel` 映射；不类路径扫描 |
| D6 | Memory 架构 | `MemoryService` 门面 + 三层委托 |
| D7 | MCP | Java SDK + stdio；SSE 放扩展 |
| D8 | 持久化 | SQLite + JPA + MEMORY.md 文件 |
| D9 | CLI | Picocli |
| D10 | Web | Spring MVC + virtual thread |
| D11 | 内建 Tool | 8 + 2（通知） |
| D12 | Sandbox | 应用层白名单 |
| D13 | 调度 | cron + 失败重试不阻塞下次周期 |
| D14 | 配置格式 | YAML frontmatter + Markdown |
| D15 | 密钥 | 环境变量 + `${ENV_VAR}` 占位 |

---

## 引用

- [spec.md](spec.md)
- [plan.md](plan.md)
- [data-model.md](data-model.md)
- [CLAUDE.md](../../CLAUDE.md)（项目宪法 + 索引）
- [docs/TechnicalSolution.md](../../docs/TechnicalSolution.md)
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md)