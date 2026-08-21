# 销售询盘助手 MVP 设计方案

## 1. 文档目的

本文档定义 Java Agent 治理与执行平台的首个架构验证 MVP。

MVP 不建设完整 Agent 管理平台，而是通过一个固定的“销售询盘助手”验证以下核心闭环：

```text
Spring Boot
  → Pi RPC
  → Model
  → Tool Proxy
  → Java Tool Gateway
  → Execution Event / Trace
```

本文档是项目总纲 `doc/project/project-design.md` 的收敛版本。当两者在 MVP 范围上存在差异时，以本文档为准。

## 2. MVP 目标

固定一个销售询盘助手 Agent，通过两个本地模拟 Tool 查询客户和询盘信息，在单个 Platform Session 中支持多轮 Execution，并通过 HTTP API、SSE 和 MySQL 验证执行与观测闭环。

MVP 完成定义：

```text
固定 Agent
  + 单 Session 多轮 Execution
  + Pi RPC
  + 两个 Java Tool
  + Tool Proxy Extension
  + SSE
  + MySQL Trace
  + Abort / Timeout
  = MVP 完成
```

## 3. 项目边界

### 3.1 MVP 范围内

- 一个固定的销售询盘助手 Agent
- 固定 System Prompt
- 固定模型 `zai-coding-cn/glm-5.3`
- 复用 Pi 环境中已有的模型认证
- 一个 Platform Session 中的多轮 Execution
- 每个 Platform Session 对应一个 Pi RPC 子进程
- `AgentRuntime` SPI 及 Pi 实现
- `get_customer_info` Java 模拟 Tool
- `get_inquiry_info` Java 模拟 Tool
- Pi Tool Proxy Extension
- Spring Boot Internal Tool Gateway
- Execution 创建、执行、完成、失败、超时和中止
- Pi 原始事件到平台事件的转换
- SSE 实时事件与历史事件回放
- MySQL Session、Execution 和 Event 持久化
- Swagger 或 Postman 演示

### 3.2 MVP 范围外

- Agent、AgentVersion、Runtime、Tool 的 CRUD 管理
- Agent 版本发布和回滚
- Web 前端
- 登录、RBAC、租户和数据权限
- 多 Agent 和多 Runtime
- 请求级模型选择和模型密钥管理
- 多个 Platform Session 并发运行
- 同一个 Session 内的并发 Execution
- Runtime 进程池和独立 Runtime Worker
- 应用重启后恢复 Pi 对话上下文
- Tool 分组、动态选择和语义检索
- 真实 CRM、询盘系统或知识库集成
- MQ、分布式部署和复杂调度
- 自动评估和 Runtime 对比

## 4. 关键设计原则

### 4.1 Platform owns Agent

Spring Boot 持有固定 Agent 的 System Prompt、模型标识、允许的 Tool 和执行策略。Pi 不定义业务 Agent。

### 4.2 Runtime owns Agent Loop

Pi 负责模型交互、Agent Loop、上下文维护和 Tool Call 调度。Spring Boot 不控制 Think、Act 和 Observe 等内部步骤。

### 4.3 Tool belongs to Platform

两个业务 Tool 由 Java 实现。Pi Extension 仅注册 Tool Schema 并转发调用，不包含业务数据或业务判断。

### 4.4 Runtime-specific objects stay outside domain

Spring Boot 领域和应用层只依赖 `AgentRuntime` 抽象。Pi 的事件、协议和进程对象只存在于 Pi Runtime Adapter 内部。

### 4.5 MVP keeps one explicit execution path

MVP 不为尚未接入的 Runtime、动态 Tool 或管理后台提前建设复杂扩展机制。保留必要接口边界，实际实现保持单一路径。

## 5. 总体架构

```text
Swagger / Postman
       │
       │ HTTP + SSE
       ▼
┌─────────────────────────────────────┐
│ Spring Boot MVP                     │
│                                     │
│ API                                 │
│  ├─ Session API                     │
│  └─ Execution API / SSE             │
│                                     │
│ Application                         │
│  ├─ SessionService                  │
│  └─ ExecutionService                │
│                                     │
│ Runtime                             │
│  ├─ AgentRuntime SPI                │
│  ├─ RuntimeRegistry                 │
│  └─ PiAgentRuntime                  │
│                                     │
│ Tool                                │
│  ├─ ToolRegistry                    │
│  ├─ get_customer_info               │
│  ├─ get_inquiry_info                │
│  └─ Internal Tool Gateway           │
│                                     │
│ Persistence                         │
│  └─ Session / Execution / Event     │
└───────────────┬─────────────────────┘
                │ stdin/stdout JSONL
                ▼
       ┌───────────────────┐
       │ Pi RPC Process    │
       │  ├─ Agent Loop    │
       │  ├─ Context       │
       │  └─ Tool Proxy    │
       └─────────┬─────────┘
                 │ HTTP
                 ▼
        Internal Tool Gateway
```

### 5.1 组件职责

| 组件 | 职责 | 不负责 |
|---|---|---|
| Session API | 创建和关闭 Platform Session | 管理 Pi 内部上下文 |
| Execution API | 提交消息、查询结果、中止执行 | 驱动 Agent Loop |
| Execution SSE | 回放并实时推送平台事件 | 透传 Pi 原始事件 |
| SessionService | Session 状态和 Pi 进程生命周期 | Agent 配置管理 |
| ExecutionService | Execution 状态、超时和事件处理 | Tool 业务逻辑 |
| AgentRuntime SPI | 定义 Runtime 的最小统一契约 | 暴露 Pi 类型 |
| PiAgentRuntime | 管理 Pi RPC 进程和 JSONL 协议 | 保存平台业务数据 |
| ToolRegistry | 管理 Java Tool 实例 | 动态检索 Tool |
| Tool Gateway | 校验并执行允许的 Tool | 对外提供公共业务 API |
| Tool Proxy Extension | 向 Pi 注册 Tool 并转发 HTTP | 保存或判断业务数据 |
| Persistence | 保存 Session、Execution 和 Event | 恢复 Pi 对话上下文 |

## 6. 固定 Agent 配置

固定配置由 Spring 外部配置加载，不建设 Agent 配置表。

```text
agentKey: sales-inquiry-assistant
provider: zai-coding-cn
model: glm-5.3
allowedTools:
  - get_customer_info
  - get_inquiry_info
timeoutSeconds: 120
```

System Prompt 必须要求 Agent：

1. 仅依据用户输入和 Tool 返回结果分析。
2. 需要客户信息时调用 `get_customer_info`。
3. 需要询盘信息时调用 `get_inquiry_info`。
4. 数据不存在或不足时明确告知用户，不编造信息。
5. 最终回答固定包含以下四个章节：
   - 客户概况
   - 询盘摘要
   - 销售建议
   - 下一步行动

模型认证由现有 Pi 环境负责。平台不读取、保存或展示认证凭据。

## 7. Runtime 设计

### 7.1 AgentRuntime 最小契约

MVP 的 `AgentRuntime` 只需要支持：

```text
type
createSession
execute
abort
closeSession
```

MVP 不为 steering、branching、compaction 等未使用能力建立公开 API。

### 7.2 Pi 进程模型

- 全局最多存在一个 ACTIVE Platform Session。
- 创建 Platform Session 时启动一个 Pi RPC 子进程。
- 一个 Session 中的多轮 Execution 复用同一子进程和上下文。
- 关闭 Session 时终止对应子进程。
- 应用停止时关闭受管 Pi 子进程。
- Pi 子进程仅保存在内存中，不根据数据库记录重建。

固定启动参数：

```text
pi
--mode rpc
--no-session
--no-builtin-tools
--no-extensions
--extension pi/extensions/sales-tool-proxy.ts
--no-skills
--no-prompt-templates
--no-context-files
--provider zai-coding-cn
--model glm-5.3
```

`--no-session` 表示不使用 Pi 文件持久化，但同一子进程存活期间仍保持多轮上下文。
启动器额外添加 `--system-prompt` 参数，其参数值来自 Spring 中的固定 Agent 配置。
Tool Gateway 地址、Platform Session ID 和临时 Tool 调用凭证通过 Pi 子进程环境传递，不拼接到命令行，也不写入日志。

### 7.3 RPC 通信约束

- stdin 和 stdout 使用严格的 LF 分隔 JSONL。
- 每条 RPC 命令携带平台生成的关联 ID。
- stderr 与协议 stdout 分开消费，避免缓冲区阻塞。
- stdout 中无法解析的内容视为协议错误，不写入 SSE 原始数据。
- Runtime Adapter 将 Pi 原始事件转换成平台事件后再交给应用层。

## 8. 领域模型

```text
Fixed AgentSpec
      │
      ▼
Platform Session
      │
      ├── Execution 1
      │      └── ExecutionEvent *
      └── Execution 2
             └── ExecutionEvent *
```

### 8.1 Platform Session

Session 代表一段持续的销售询盘对话，并绑定一个 Pi Runtime Session。

状态：

```text
ACTIVE
CLOSED
```

### 8.2 Execution

Execution 代表 Session 内的一次用户消息和完整 Agent Run。

状态：

```text
CREATED
RUNNING
SUCCEEDED
FAILED
ABORTED
TIMEOUT
```

合法状态转换：

```text
CREATED → RUNNING → SUCCEEDED
                  → FAILED
                  → ABORTED
                  → TIMEOUT
```

终态不可再次变更。并发终态通过数据库条件更新保证只有一个结果生效。

### 8.3 Execution Event

平台事件类型：

```text
RUN_STARTED
MESSAGE_DELTA
TOOL_CALL_STARTED
TOOL_CALL_COMPLETED
USAGE_UPDATED
RUN_COMPLETED
RUN_FAILED
RUN_ABORTED
RUN_TIMEOUT
```

平台不持久化或对外输出模型原始思考过程。

## 9. 数据库设计

MVP 使用以下数据库名称：

```text
java_agent_ops_mvp
```

使用 Flyway 管理 Schema。连接地址、账号和密码全部通过环境变量或外部配置注入。

### 9.1 agent_session

```text
id
agent_key
runtime_type
runtime_session_id
status
created_at
closed_at
```

### 9.2 agent_execution

```text
id
session_id
status
input
output
error_code
error_message
input_tokens
output_tokens
tool_call_count
started_at
finished_at
created_at
```

### 9.3 agent_execution_event

```text
id
execution_id
sequence_no
event_type
event_data JSON
created_at
```

约束：

- `agent_execution_event` 对 `(execution_id, sequence_no)` 建立唯一约束。
- Event 的 `sequence_no` 在单个 Execution 内单调递增。
- Session 与 Execution 状态字段使用固定枚举值。
- Runtime Session ID 仅用于追踪，不用于应用重启后的恢复。

## 10. Tool 设计

### 10.1 get_customer_info

输入：

```json
{
  "customerId": "C001"
}
```

固定样例输出：

```json
{
  "found": true,
  "customerId": "C001",
  "companyName": "示例科技有限公司",
  "industry": "工业自动化",
  "region": "华东",
  "customerLevel": "A",
  "cooperationStatus": "POTENTIAL",
  "notes": "关注交付周期和售后响应"
}
```

### 10.2 get_inquiry_info

输入：

```json
{
  "inquiryId": "I001"
}
```

固定样例输出：

```json
{
  "found": true,
  "inquiryId": "I001",
  "customerId": "C001",
  "productName": "工业控制模块",
  "quantity": 100,
  "targetPrice": "模拟价格区间",
  "expectedDeliveryDate": "2026-09-30",
  "status": "PENDING_ANALYSIS",
  "requirements": "需要技术参数确认及交期回复"
}
```

未知 ID 返回正常 Tool 结果：

```json
{
  "found": false,
  "message": "未找到对应的模拟数据"
}
```

模拟数据直接定义在 Java Tool 实现中。MySQL 不建立模拟 CRM 或询盘业务表。

### 10.3 Tool 调用链路

```text
Pi Agent
  → Tool Proxy Extension
  → POST /internal/tools/{toolName}/invoke
  → Tool Gateway
  → ToolRegistry
  → Java AgentTool
  → Tool Result
  → Pi Agent Loop
```

Tool Gateway 必须校验：

1. 临时调用凭证有效。
2. Platform Session 为 ACTIVE。
3. Session 当前存在唯一 RUNNING Execution。
4. Tool 名称属于固定 Agent 的允许列表。
5. Tool 参数满足 Schema。

## 11. API 设计

### 11.1 对外 API

```http
POST   /api/sessions
DELETE /api/sessions/{sessionId}

POST   /api/sessions/{sessionId}/executions
GET    /api/executions/{executionId}
GET    /api/executions/{executionId}/events
GET    /api/executions/{executionId}/trace
POST   /api/executions/{executionId}/abort
```

创建 Execution：

```json
{
  "message": "请分析客户 C001 的询盘 I001，并给出销售跟进建议"
}
```

返回 `202 Accepted`：

```json
{
  "executionId": "exec_xxx",
  "status": "CREATED",
  "eventsUrl": "/api/executions/exec_xxx/events"
}
```

### 11.2 Internal Tool API

```http
POST /internal/tools/{toolName}/invoke
```

Internal Tool API 只服务于本机 Pi Tool Proxy，不应通过外部网关暴露。

### 11.3 SSE 事件

统一事件结构：

```json
{
  "executionId": "exec_xxx",
  "sequence": 3,
  "type": "TOOL_CALL_STARTED",
  "timestamp": "2026-08-21T10:00:00+08:00",
  "data": {
    "toolName": "get_customer_info",
    "toolCallId": "call_xxx"
  }
}
```

所有 Event 由单一 Event Publisher 按“分配序号、提交数据库、发布实时事件”的顺序处理。SSE 订阅先注册实时监听，再读取当前事件快照，最后按 `sequence_no` 去重合并历史和实时事件。客户端稍晚连接不应丢失开头事件。
Execution 已进入终态时，SSE 完成历史回放后关闭连接。

## 12. 执行流程

```text
创建 Platform Session
  → 持久化 Session
  → 启动 Pi RPC 子进程
  → 加载 Tool Proxy Extension
  → 获取并保存 runtimeSessionId

提交用户消息
  → 校验 Session 为 ACTIVE
  → 校验没有 RUNNING Execution
  → 创建 Execution
  → 异步发送 Pi prompt
  → Pi 产生 JSONL Event
  → Java 转换、持久化并推送 SSE
  → Pi 调用 Tool Proxy
  → Tool Gateway 执行 Java 模拟 Tool
  → Tool 结果返回 Pi
  → Pi 继续 Agent Loop
  → Pi 输出四个固定章节
  → Java 保存 output、usage 和终态
```

## 13. 并发与生命周期规则

- 全局只允许一个 ACTIVE Platform Session。
- 关闭当前 Session 后才能创建新 Session。
- 创建 Pi 子进程失败时，Session 创建失败并返回 HTTP 503，不保留 ACTIVE Session。
- 关闭存在 RUNNING Execution 的 Session 时，先中止 Execution，再终止 Pi 子进程并关闭 Session。
- 同一 Session 同时只允许一个 RUNNING Execution。
- 多轮消息严格按顺序执行，不提供内部排队。
- 并发提交第二个 Execution 返回 HTTP `409 Conflict`。
- `abort`、timeout、Runtime 回调和正常完成可能并发到达，终态必须通过条件更新竞争。
- 应用启动时关闭数据库中遗留的 ACTIVE Session。
- 应用启动时将遗留的 CREATED 或 RUNNING Execution 标记为 FAILED，并使用稳定错误码表示应用重启中断。

## 14. 错误处理

### 14.1 业务型结果

以下情况不将 Execution 标记为失败：

- 客户 ID 不存在
- 询盘 ID 不存在
- 用户没有提供足够的 ID

Agent 应明确告知数据不足并要求用户补充，不得编造结果。

### 14.2 技术型错误

以下情况将 Execution 标记为 FAILED：

- Pi 子进程异常退出
- RPC JSONL 协议异常
- 模型调用失败且 Pi 未恢复
- Tool Gateway 技术异常
- Tool Proxy 通信失败

超时处理：

```text
到达 timeout
  → 向 Pi 发送 abort
  → Execution 条件更新为 TIMEOUT
  → 发布 RUN_TIMEOUT
```

主动中止：

```text
POST abort
  → 校验 Execution 为 RUNNING
  → 向 Pi 发送 abort
  → Execution 条件更新为 ABORTED
  → 发布 RUN_ABORTED
```

对外错误只包含稳定 `errorCode` 和清理后的错误描述。底层堆栈、命令行、请求头和环境变量不进入 SSE 或数据库 Trace。

## 15. 安全边界

- 使用 `--no-builtin-tools` 禁止 Pi 的文件、Shell 和进程工具。
- 使用 `--no-extensions` 后仅显式加载销售 Tool Proxy Extension。
- 禁用自动发现的 Skill、Prompt Template 和上下文文件。
- 每个 Platform Session 生成随机临时 Tool 调用凭证。
- 临时凭证只保存在内存和 Pi 子进程环境中，Session 关闭后失效。
- Tool Gateway 同时校验 Session、Execution、Tool allowlist 和参数 Schema。
- Internal Tool API 不通过外部网关暴露。
- MySQL 凭据和 Pi 认证信息不写入源码、文档、数据库事件或日志。
- 应用日志不记录 System Prompt、用户完整输入、完整 Tool 参数、Tool 结果和客户资料。
- MVP 模拟数据不得使用真实客户资料。

## 16. 可观测性

Trace 包含：

- 用户输入
- 模型文本增量和最终回答
- Tool 名称、过滤后的参数和结果
- Execution 状态变化
- 输入和输出 Token
- Tool Call 次数
- 开始时间、结束时间和耗时
- 稳定错误码和清理后的错误描述

日志只记录：

- Session ID
- Execution ID
- Runtime 类型
- 状态
- 耗时
- Tool 名称
- 稳定错误码

平台不持久化或展示模型原始思考过程。

## 17. 测试策略

### 17.1 单元测试

- Execution 状态转换
- 终态条件更新规则
- Pi Event 到 Platform Event 的映射
- Tool Registry
- 两个模拟 Tool
- Tool allowlist 和参数校验
- 敏感字段过滤

### 17.2 组件测试

使用 Fake Runtime 验证：

- Session 创建和关闭
- Execution 异步执行
- SSE 历史回放和实时推送
- abort
- timeout
- Runtime 失败

组件测试不依赖真实模型。

### 17.3 持久化集成测试

- Flyway Migration
- Session、Execution 和 Event Repository
- `(execution_id, sequence_no)` 唯一约束
- Execution 终态条件更新
- 应用重启清理逻辑

测试连接信息通过独立环境变量提供，不写入仓库。

### 17.4 真实 Pi 冒烟测试

使用当前 Pi 环境和已认证模型验证：

- RPC 启动
- 固定模型选择
- Tool Proxy 加载
- 两个 Java Tool 调用
- SSE 事件
- 同一 Session 多轮上下文
- 四章节最终回答

模型输出具有非确定性，测试只验证章节、Tool Call 和状态，不精确匹配回答全文。

测试配置可以为模拟 Tool 增加固定延迟，以稳定验证 abort 和 timeout。延迟配置不进入生产 API。

## 18. MVP 验收标准

1. 创建 Session 后 Pi 子进程正常启动，且可用 Tool 只有两个销售 Tool。
2. 输入客户 `C001` 和询盘 `I001` 后，Agent 调用两个 Tool。
3. 最终回答包含“客户概况、询盘摘要、销售建议、下一步行动”四个章节。
4. 在同一 Session 继续询问跟进话术时，Agent 使用上一轮上下文并创建新的 Execution。
5. 输入不存在的 ID 时，Agent 明确说明数据不存在，不编造信息。
6. SSE 能回放已有事件并继续接收实时事件。
7. Execution 能进入 SUCCEEDED、FAILED、ABORTED 和 TIMEOUT 终态。
8. 同一 Session 并发提交第二个 Execution 时返回 HTTP 409。
9. 应用重启后历史 Trace 仍可查询，旧 Session 被关闭，未完成 Execution 被标记失败。
10. 日志、数据库错误字段和 SSE 中不出现凭据、环境变量或底层堆栈。
11. 无需前端即可通过 Swagger 或 Postman 演示全部功能。

## 19. 后续演进触发条件

只有出现明确需求后，才进入以下演进：

| 触发条件 | 后续能力 |
|---|---|
| 需要多个并发 Session | Runtime 进程池或独立 Worker |
| 需要重启后继续对话 | Pi Session 文件生命周期和恢复 |
| 需要在线修改 Agent | Agent / AgentVersion 持久化与发布 |
| Tool 数量明显增长 | Tool Binding 和规则型 ToolResolver |
| 接入第二种 Runtime | RuntimeCapabilities 和 Runtime 路由 |
| 接入真实业务系统 | 认证、审计、数据脱敏和调用治理 |
| 需要衡量回答质量 | Dataset、Evaluation 和回归测试 |

在上述条件出现前，MVP 不提前实现对应能力。
