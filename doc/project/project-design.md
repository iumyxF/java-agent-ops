# Java Agent 治理与执行平台设计方案

## 1. 项目定位

本项目基于 **Java + Spring Boot** 构建一套轻量级的 **Agent 治理与执行平台**。

平台本身不负责实现完整 Agent Loop，而是负责：

- Agent 定义与版本治理
- Prompt、Model、Tool、Runtime 配置
- Session 生命周期管理
- Agent Execution 执行管理
- Tool 注册与调用治理
- Runtime 路由与能力抽象
- Execution Event / Trace 记录
- 运行结果、耗时、Token、Tool Call 等观测能力

第一阶段将 **Pi** 作为底层 Agent Runtime。

后续可继续接入：

- 自研 Harness Runtime
- LangChain4j Runtime
- DeepSeek Harness Runtime
- 其他 Agent Runtime

核心目标是：

> Agent Platform owns Agent，Runtime owns Agent Loop，Tool belongs to Platform，Pi is only one Runtime implementation.

即：

- **Agent Platform** 管理 Agent
- **Runtime** 负责 Agent Loop
- **Tool** 属于平台能力
- **Pi** 只是 Runtime 的一种实现

---

## 2. 设计原则

### 2.1 Runtime 可替换

Spring Boot 业务层不能直接依赖 Pi 的内部对象或协议。

例如不应该在业务层大量出现：

```java
PiAgentService
PiSessionService
PiExecutionService
PiToolService
```

而应该统一通过：

```java
AgentRuntime
```

进行调用。

整体关系：

```text
Agent Platform
      │
      ▼
AgentRuntime SPI
      │
      ├── PiAgentRuntime
      │
      ├── HarnessAgentRuntime
      │
      └── OtherAgentRuntime
```

---

### 2.2 Agent 配置属于平台，Agent Loop 属于 Runtime

Spring Boot 负责决定：

- 当前运行哪个 Agent
- 使用哪个 AgentVersion
- System Prompt
- Model
- 允许哪些 Tool
- 使用哪个 Runtime
- 执行超时
- Session
- Execution 状态
- Event / Trace

Runtime 负责：

```text
LLM
 ↓
Think
 ↓
Tool Call
 ↓
Observe
 ↓
LLM
 ↓
...
 ↓
Final Answer
```

Spring Boot 不参与 Runtime 内部每一步 Agent Loop。

---

### 2.3 Tool 属于平台，不属于 Pi

业务 Tool 不应该全部写在 Pi Extension 内部。

例如：

```text
query_order
query_customer
query_product
search_knowledge
create_ticket
send_email
```

这些应该定义在 Spring Boot Agent Platform 中。

Pi 只是调用这些 Tool。

这样未来切换到自研 Harness 时，不需要重新实现全部业务 Tool。

---

### 2.4 V1 优先 Agent 核心能力

第一阶段暂不重点建设：

- 用户体系
- 复杂登录认证
- RBAC
- 部门
- 租户
- 数据权限
- 审批流
- Agent Marketplace
- Prompt Marketplace
- MQ
- 分布式调度
- 多 Agent Workflow Designer

V1 可以直接使用简单管理员模式。

重点投入：

```text
Agent
AgentVersion
Runtime
Tool
Session
Execution
Event / Trace
```

---

# 3. 总体架构

```text
                         ┌──────────────────────┐
                         │      Web / API       │
                         └──────────┬───────────┘
                                    │
                             HTTP / SSE
                                    │
                     ┌──────────────▼──────────────┐
                     │     Spring Boot Agent       │
                     │         Platform            │
                     │                             │
                     │  Agent Management           │
                     │  Agent Version              │
                     │  Session Management         │
                     │  Execution Management       │
                     │  Tool Registry              │
                     │  Runtime Governance         │
                     │  Event / Trace              │
                     └──────────────┬──────────────┘
                                    │
                           AgentRuntime SPI
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
             ▼                      ▼                      ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │ PiRuntimeAdapter│    │HarnessRuntime   │    │ Other Runtime   │
    │                 │    │    Adapter      │    │    Adapter      │
    └────────┬────────┘    └─────────────────┘    └─────────────────┘
             │
             │ RPC / HTTP
             ▼
    ┌────────────────────────┐
    │    Pi Runtime Worker   │
    │                        │
    │  AgentSession          │
    │  Agent Loop            │
    │  Model Runtime         │
    │  Tool Calling          │
    │  Context / Compaction  │
    └───────────┬────────────┘
                │
                │ Tool Invocation
                ▼
       ┌───────────────────────┐
       │ Spring Tool Gateway   │
       │                       │
       │ CRM Tool              │
       │ Order Tool            │
       │ Knowledge Tool        │
       │ Email Tool            │
       │ Internal API Tool     │
       └───────────────────────┘
```

---

# 4. 核心领域模型

建议第一阶段重点围绕以下对象设计：

```text
Agent
AgentVersion
RuntimeBinding
Session
Execution
ExecutionEvent
Tool
```

---

## 4.1 Agent

Agent 代表一个逻辑智能体。

例如：

```text
售后客服 Agent
询盘分析 Agent
订单助手 Agent
知识库问答 Agent
```

建议模型：

```java
public class Agent {

    private Long id;

    private String name;

    private String description;

    private Long activeVersionId;

    private AgentStatus status;
}
```

Agent 本身不要保存大量具体运行配置。

---

# 5. AgentVersion

Prompt、Model、Tool、Runtime 等运行参数应该进行版本化。

例如：

```text
AfterSaleAgent

V1
Prompt A
DeepSeek
Tool A/B

V2
Prompt B
Claude
Tool A/B/C
```

建议模型：

```java
public class AgentVersion {

    private Long id;

    private Long agentId;

    private Integer version;

    private String systemPrompt;

    private ModelConfig model;

    private List<String> tools;

    private RuntimeBinding runtime;

    private ExecutionPolicy executionPolicy;
}
```

示例配置：

```json
{
  "agentId": 1001,
  "version": 3,
  "systemPrompt": "你是公司的售后客服Agent...",
  "model": {
    "provider": "deepseek",
    "model": "deepseek-chat"
  },
  "tools": [
    "query_order",
    "query_product",
    "search_knowledge"
  ],
  "runtime": {
    "type": "pi",
    "options": {}
  },
  "executionPolicy": {
    "timeoutSeconds": 300
  }
}
```

---

# 6. RuntimeBinding

Runtime 特有配置不能污染 Agent Domain。

错误设计：

```java
public class Agent {

    private String piSkill;

    private String piExtension;

    private String piSessionConfig;
}
```

推荐设计：

```java
public class RuntimeBinding {

    private String runtimeType;

    private Map<String, Object> options;
}
```

Pi 示例：

```json
{
  "runtimeType": "pi",
  "options": {
    "thinkingLevel": "medium",
    "skills": [
      "after-sale"
    ]
  }
}
```

未来自研 Harness：

```json
{
  "runtimeType": "harness",
  "options": {
    "planner": true,
    "reflection": true,
    "maxReflection": 2
  }
}
```

---

# 7. AgentRuntime SPI

这是整个系统实现 Runtime 可替换的核心。

```java
public interface AgentRuntime {

    /**
     * Runtime 唯一类型。
     */
    String type();

    /**
     * Runtime 能力描述。
     */
    RuntimeCapabilities capabilities();

    /**
     * 创建 Runtime Session。
     */
    RuntimeSession createSession(RuntimeSessionRequest request);

    /**
     * 执行 Agent。
     */
    void execute(
        RuntimeExecutionRequest request,
        RuntimeEventListener listener
    );

    /**
     * 中断执行。
     */
    void abort(String runtimeSessionId);

    /**
     * 关闭 Session。
     */
    void closeSession(String runtimeSessionId);
}
```

Pi 实现：

```java
@Component
public class PiAgentRuntime implements AgentRuntime {

    @Override
    public String type() {
        return "pi";
    }

    @Override
    public RuntimeCapabilities capabilities() {
        return new RuntimeCapabilities(
            true,
            true,
            true,
            true,
            true,
            true,
            true
        );
    }

    @Override
    public RuntimeSession createSession(
            RuntimeSessionRequest request) {
        // 调用 Pi Runtime
        return null;
    }

    @Override
    public void execute(
            RuntimeExecutionRequest request,
            RuntimeEventListener listener) {
        // 调用 Pi Runtime
    }

    @Override
    public void abort(String runtimeSessionId) {
    }

    @Override
    public void closeSession(String runtimeSessionId) {
    }
}
```

以后：

```java
@Component
public class HarnessAgentRuntime implements AgentRuntime {

    @Override
    public String type() {
        return "harness";
    }

    // ...
}
```

---

# 8. RuntimeCapabilities

不同 Runtime 支持的能力可能不同。

因此不要强行要求所有 Runtime 完全一致。

建议定义：

```java
public record RuntimeCapabilities(

    boolean streaming,

    boolean session,

    boolean abort,

    boolean steering,

    boolean customTools,

    boolean compaction,

    boolean branching

) {}
```

例如：

| Capability | Pi | Self Harness |
|---|---:|---:|
| Streaming | ✅ | ✅ |
| Session | ✅ | ✅ |
| Abort | ✅ | ✅ |
| Steering | ✅ | ❌ |
| Custom Tool | ✅ | ✅ |
| Compaction | ✅ | ❌ |
| Branching | ✅ | ❌ |

这样可以在保持统一 Runtime SPI 的同时，保留不同 Runtime 的高级能力。

---

# 9. RuntimeRegistry

Runtime 使用 Registry 管理。

```java
@Component
public class AgentRuntimeRegistry {

    private final Map<String, AgentRuntime> runtimeMap;

    public AgentRuntimeRegistry(
            List<AgentRuntime> runtimes) {

        this.runtimeMap = runtimes.stream()
            .collect(Collectors.toMap(
                AgentRuntime::type,
                Function.identity()
            ));
    }

    public AgentRuntime get(String type) {

        AgentRuntime runtime = runtimeMap.get(type);

        if (runtime == null) {
            throw new IllegalArgumentException(
                "Runtime not found: " + type
            );
        }

        return runtime;
    }
}
```

调用：

```java
AgentRuntime runtime =
    runtimeRegistry.get(agentVersion.getRuntimeType());

runtime.execute(request, listener);
```

---

# 10. AgentSpecCompiler

建议增加一个 AgentSpecCompiler。

数据库实体不应该直接传给 Runtime。

执行前先将：

```text
Agent
AgentVersion
Model
Tool Binding
Runtime Binding
Execution Policy
```

转换成统一运行描述。

```text
AgentVersion
      │
      ▼
AgentSpecCompiler
      │
      ▼
AgentExecutionSpec
```

定义：

```java
public record AgentExecutionSpec(

    String agentId,

    String agentVersion,

    String systemPrompt,

    ModelSpec model,

    List<String> allowedTools,

    ExecutionPolicy policy,

    RuntimeSpec runtime

) {}
```

这样 Runtime 不需要了解数据库设计。

---

# 11. Agent、Session、Execution 的关系

这三个概念必须明确区分。

## Agent

代表：

> 谁来干活。

例如：

```text
AfterSaleAgent
```

---

## Session

代表：

> 一段持续会话上下文。

例如：

```text
客户 A 和售后 Agent 的一次持续对话。
```

一个 Session 可以包含多个 Execution：

```text
Session
 ├── Execution 1
 ├── Execution 2
 ├── Execution 3
 └── Execution 4
```

---

## Execution

代表：

> 一次独立的 Agent Run。

例如用户输入：

```text
帮我查询订单 20260821001
```

这就是一次 Execution。

内部可能发生：

```text
LLM
 ↓
query_order
 ↓
LLM
 ↓
query_shipping
 ↓
LLM
 ↓
Final Answer
```

这些内部步骤属于 Runtime Turn，不需要由 Spring Boot 控制。

整体关系：

```text
Agent
   │
   └── Session
          │
          ├── Execution
          ├── Execution
          └── Execution
```

---

# 12. Platform Session 与 Runtime Session 隔离

不能直接使用 Pi Session ID 作为平台 Session ID。

建议：

```text
Platform Session
       │
       ▼
Runtime Session Reference
```

数据结构：

```text
agent_session

id
agent_id
agent_version_id
runtime_type
runtime_session_id
status
created_at
updated_at
```

例如：

```text
Platform Session:

sess_10001

Runtime:

runtimeType = PI
runtimeSessionId = pi_xxxxx
```

以后换 Harness：

```text
runtimeType = HARNESS
runtimeSessionId = harness_xxxxx
```

平台 Session 模型不变。

---

# 13. Execution

Execution 是整个系统最重要的运行态对象之一。

建议：

```java
public class AgentExecution {

    private String id;

    private Long agentId;

    private Long agentVersionId;

    private String sessionId;

    private String runtimeType;

    private ExecutionStatus status;

    private String input;

    private String output;

    private LocalDateTime startedAt;

    private LocalDateTime finishedAt;

    private String errorMessage;
}
```

ExecutionStatus：

```java
public enum ExecutionStatus {

    CREATED,

    RUNNING,

    SUCCEEDED,

    FAILED,

    ABORTED,

    TIMEOUT
}
```

第一版不建议把执行状态拆得过细。

例如暂时不要设计：

```text
WAITING_WORKER
THINKING
TOOL_PENDING
REFLECTING
PLANNING
```

Runtime 内部细节通过 Event 表达即可。

---

# 14. Event 模型

平台 Event 不能直接使用 Pi 原始 Event。

应该增加 Runtime Event 抽象层。

建议：

```java
public enum AgentRuntimeEventType {

    RUN_STARTED,

    MESSAGE_STARTED,

    MESSAGE_DELTA,

    THINKING_DELTA,

    TOOL_CALL_STARTED,

    TOOL_CALL_PROGRESS,

    TOOL_CALL_COMPLETED,

    USAGE_UPDATED,

    RUN_COMPLETED,

    RUN_FAILED,

    RUN_ABORTED
}
```

Pi Adapter 做事件转换：

```text
Pi Event
message_update
      │
      ▼
PiRuntimeAdapter
      │
      ▼
MESSAGE_DELTA
```

又如：

```text
Pi
tool_execution_start
      │
      ▼
TOOL_CALL_STARTED
```

未来 Harness：

```text
Harness
ActionStart
      │
      ▼
TOOL_CALL_STARTED
```

前端无需感知 Runtime 类型。

---

# 15. Streaming

推荐使用 SSE 输出运行事件。

调用：

```http
POST /api/agents/{agentId}/executions
```

返回：

```text
executionId
```

前端订阅：

```http
GET /api/executions/{executionId}/events
```

事件示例：

```text
RUN_STARTED

MESSAGE_DELTA
你好

MESSAGE_DELTA
，我正在查询订单

TOOL_CALL_STARTED
query_order

TOOL_CALL_COMPLETED

MESSAGE_DELTA
订单目前已经发货

RUN_COMPLETED
```

第一阶段不需要 MQ。

---

# 16. Tool Registry

Tool 是平台统一管理的能力。

建议接口：

```java
public interface AgentTool {

    /**
     * Tool 唯一名称。
     */
    String name();

    /**
     * Tool 描述。
     */
    String description();

    /**
     * 输入 Schema。
     */
    JsonSchema inputSchema();

    /**
     * 执行 Tool。
     */
    ToolResult execute(
        ToolContext context,
        Map<String, Object> arguments
    );
}
```

例如：

```java
@Component
public class QueryOrderTool implements AgentTool {

    @Override
    public String name() {
        return "query_order";
    }

    @Override
    public String description() {
        return "根据订单号查询订单信息";
    }

    // ...
}
```

---

# 17. Tool 的三层模型

Tool 建议分成三层：

```text
Tool Registry
     │
     │ 系统全部 Tool
     ▼
AgentVersion Tool Binding
     │
     │ Agent 允许使用的 Tool
     ▼
Execution Tool Resolution
     │
     │ 当前任务真正暴露给 Runtime 的 Tool
     ▼
Pi Runtime
```

---

## 17.1 Tool Registry

代表系统拥有的全部 Tool。

例如：

```text
query_order
query_shipping
query_customer
query_product
search_knowledge
create_ticket
send_email
cancel_order
```

---

## 17.2 Agent Allowed Tools

AgentVersion 配置的是 Agent 的能力边界。

例如售后 Agent：

```text
query_order
query_shipping
query_customer
query_product
search_knowledge
create_ticket
```

没有：

```text
delete_customer
change_price
```

这一层属于治理配置。

---

## 17.3 Runtime Tools

一次独立任务真正运行时，可以进一步裁剪 Tool。

例如用户：

```text
帮我查询订单 20260821001 的物流状态
```

AgentVersion 允许：

```text
query_order
query_customer
query_product
search_knowledge
query_shipping
create_ticket
send_email
```

但本次 Runtime 实际只暴露：

```text
query_order
query_shipping
```

因此关系为：

```text
RuntimeTools
    ⊆
AgentAllowedTools
    ⊆
ToolRegistry
```

一个重要原则：

> ToolResolver 只能做减法，不能突破 AgentVersion 的 Tool 能力边界。

---

# 18. ToolResolver

建议增加：

```java
public interface ToolResolver {

    List<ToolDefinition> resolve(
        AgentExecutionContext context,
        AgentExecutionSpec spec
    );
}
```

第一版实现：

```java
@Component
public class DefaultToolResolver
        implements ToolResolver {

    @Override
    public List<ToolDefinition> resolve(
            AgentExecutionContext context,
            AgentExecutionSpec spec) {

        return spec.allowedTools()
            .stream()
            .map(...)
            .toList();
    }
}
```

V1：

```text
RuntimeTools = AgentAllowedTools
```

也就是：

> 架构支持动态 Tool，但第一版实际全部加载。

这样可以避免第一阶段过度设计。

---

# 19. RuntimeExecutionSpec

建议区分两个 Spec。

## AgentExecutionSpec

表示 Agent 静态配置。

```java
public record AgentExecutionSpec(

    String systemPrompt,

    ModelSpec model,

    List<String> allowedTools,

    ExecutionPolicy policy,

    RuntimeSpec runtime

) {}
```

---

## RuntimeExecutionSpec

表示当前一次 Execution 最终交给 Runtime 的配置。

```java
public record RuntimeExecutionSpec(

    String executionId,

    String sessionId,

    String systemPrompt,

    ModelSpec model,

    List<ToolDefinition> tools,

    RuntimeSpec runtime,

    String userMessage

) {}
```

流程：

```text
AgentVersion
      │
      ▼
AgentSpecCompiler
      │
      ▼
AgentExecutionSpec
      │
      ▼
ToolResolver
      │
      ▼
RuntimeExecutionSpec
      │
      ▼
RuntimeRouter
      │
      ▼
PiRuntime
```

---

# 20. Tool 动态选择演进路线

## V1：全部 Allowed Tool

```text
AgentVersion
      ↓
Allowed Tools
      ↓
DefaultToolResolver
      ↓
全部加载
      ↓
Pi
```

优点：

- 简单
- 快速跑通
- 不增加额外 LLM
- 便于验证 Agent / Runtime / Tool 基础架构

---

## V2：规则型 ToolResolver

增加：

```text
Tool Group
Tool Tag
Execution Context
Rule-based Resolver
```

例如：

```text
ORDER

query_order
query_shipping
```

```text
PRODUCT

query_product
search_knowledge
```

```text
TICKET

query_ticket
create_ticket
```

订单任务：

```text
ORDER Tools
```

产品故障：

```text
PRODUCT + TICKET
```

---

## V3：Semantic Tool Retrieval

当 Tool 数量达到几十甚至上百时，再考虑语义检索。

流程：

```text
100 Tools
   │
   ▼
Tool Description Embedding
   │
   ▼
根据 User Message 检索
   │
   ▼
Top 5 ~ 10
   │
   ▼
Pi Runtime
```

它本质类似：

```text
RAG Document Retrieval
```

只是检索对象从 Document 变成 Tool。

---

# 21. Pi 与 Java Tool 的调用方式

建议：

```text
Pi Agent
   │
   │ Tool Call
   ▼
Pi Tool Proxy
   │
   │ HTTP
   ▼
Spring Boot Tool Gateway
   │
   ▼
Tool Registry
   │
   ▼
AgentTool
   │
   ▼
Business Service
```

例如 Pi 调用：

```json
{
  "name": "query_order",
  "arguments": {
    "orderNo": "20260821001"
  }
}
```

Pi Runtime Worker 请求：

```http
POST /internal/tools/query_order/invoke
```

Spring：

```java
@RestController
@RequestMapping("/internal/tools")
public class ToolGatewayController {

    private final ToolRegistry toolRegistry;

    @PostMapping("/{name}/invoke")
    public ToolResult invoke(
            @PathVariable String name,
            @RequestBody ToolInvokeRequest request) {

        AgentTool tool = toolRegistry.get(name);

        return tool.execute(
            request.getContext(),
            request.getArguments()
        );
    }
}
```

返回：

```json
{
  "success": true,
  "data": {
    "status": "SHIPPED"
  }
}
```

结果重新交给 Pi，由 Pi 继续完成 Agent Loop。

---

# 22. Pi Runtime 接入方式

建议分两个阶段。

---

## 22.1 V1：Spring Boot 直接启动 Pi RPC

第一版可以：

```text
Spring Boot
     │
     ▼
PiRuntimeAdapter
     │
     ▼
ProcessBuilder
     │
     ▼
pi --mode rpc
```

示意代码：

```java
Process process = new ProcessBuilder(
    "pi",
    "--mode",
    "rpc"
).start();
```

Java 与 Pi 通过 stdin/stdout JSONL 通信。

优点：

- 实现简单
- 快速验证
- 不增加额外 Runtime 服务

适合第一阶段验证：

```text
Java
 ↓
AgentRuntime
 ↓
Pi
 ↓
LLM
 ↓
Tool
 ↓
Result
```

完整闭环。

---

## 22.2 V2：独立 Pi Runtime Worker

随着以下能力增加：

```text
多 Session
并发执行
动态 Tool
Runtime Pool
Worker Health
Session Reuse
```

可以将 Pi 包装成一个很薄的 Worker：

```text
Spring Boot
       │
       │ HTTP / SSE
       ▼
Pi Runtime Worker
       │
       ▼
Pi SDK
```

目录示例：

```text
pi-runtime-worker/

src/
 ├── runtime.ts
 ├── session-manager.ts
 ├── tool-proxy.ts
 ├── event-mapper.ts
 └── server.ts
```

Worker 只负责：

1. 创建 Pi AgentSession
2. 注入 Prompt
3. 配置 Model
4. 注册 Tool Proxy
5. Session 生命周期
6. Event 转发
7. Abort

不要在 Worker 中加入业务逻辑。

---

# 23. 完整 Execution 流程

用户请求：

```text
查询订单 20260821001 当前状态
```

完整流程：

```text
User
 │
 ▼
AgentController
 │
 ▼
AgentExecutionService
 │
 │ 获取 AgentVersion
 ▼
AgentSpecCompiler
 │
 ▼
AgentExecutionSpec
 │
 ▼
ToolResolver
 │
 ▼
RuntimeExecutionSpec
 │
 ▼
RuntimeRegistry
 │
 │ runtimeType = PI
 ▼
PiAgentRuntime
 │
 ▼
Pi Runtime
 │
 ▼
LLM
 │
 ▼
query_order
 │
 ▼
Pi Tool Proxy
 │
 ▼
Spring Tool Gateway
 │
 ▼
QueryOrderTool
 │
 ▼
OrderService
 │
 ▼
ToolResult
 │
 ▼
Pi Runtime
 │
 ▼
LLM
 │
 ▼
Final Answer
 │
 ▼
Runtime Event
 │
 ▼
ExecutionService
 │
 ▼
SSE
 │
 ▼
Frontend
```

Spring Boot 不控制：

```text
Think
Tool Call
Observe
Think
```

只负责治理和执行生命周期。

---

# 24. Execution Trace

Execution Trace 是平台非常重要的 Agent 专属能力。

建议展示：

```text
Run #100083
────────────────────────────

09:31:01

User
查询订单 20260821001

09:31:02

LLM
Thinking...

09:31:04

Tool Call
query_order

Arguments

{
  "orderNo": "20260821001"
}

09:31:04

Tool Result

{
  "status": "SHIPPED"
}

09:31:06

LLM

您的订单目前已经发货。

────────────────────────────

Duration: 5.4s

Input Tokens: 1203

Output Tokens: 285

Tool Calls: 1
```

相比传统系统后台里的：

```text
菜单管理
部门管理
字典管理
角色管理
```

Execution Trace 更值得在 V1 投入。

---

# 25. Spring Boot 包结构

第一阶段不建议拆成微服务。

也不一定需要 Maven 多 Module。

推荐单体模块化：

```text
com.xxx.agent
│
├── api
│
│   ├── AgentController
│
│   ├── SessionController
│
│   └── ExecutionController
│
├── application
│
│   ├── AgentApplicationService
│
│   ├── AgentExecutionService
│
│   └── SessionApplicationService
│
├── domain
│
│   ├── agent
│
│   ├── session
│
│   ├── execution
│
│   └── tool
│
├── runtime
│
│   ├── spi
│
│   ├── registry
│
│   └── pi
│
├── tool
│
│   ├── registry
│
│   ├── resolver
│
│   ├── gateway
│
│   └── builtin
│
├── compiler
│
│   └── AgentSpecCompiler
│
└── infrastructure
    ├── persistence
    ├── client
    └── config
```

---

# 26. 数据库设计

V1 建议控制在 6～7 张核心表。

---

## 26.1 agent

保存 Agent 基础信息。

建议字段：

```text
id
name
description
active_version_id
status
created_at
updated_at
```

---

## 26.2 agent_version

保存 Agent 版本。

```text
id
agent_id
version
system_prompt
model_config
runtime_type
runtime_config
execution_policy
status
created_at
```

---

## 26.3 agent_tool

Tool Registry。

```text
id
tool_name
description
input_schema
status
created_at
updated_at
```

如果 Tool 全部通过 Spring Bean 注册，数据库也可以只保存 Tool Metadata。

---

## 26.4 agent_tool_binding

AgentVersion 与 Tool 绑定。

```text
id
agent_version_id
tool_id
created_at
```

---

## 26.5 agent_session

```text
id
agent_id
agent_version_id
runtime_type
runtime_session_id
status
created_at
updated_at
```

---

## 26.6 agent_execution

```text
id
agent_id
agent_version_id
session_id
runtime_type
status
input
output
error_message
started_at
finished_at
created_at
```

---

## 26.7 agent_execution_event

```text
id
execution_id
event_type
sequence_no
event_data
created_at
```

---

# 27. V1 管理页面

后台第一版只需要：

```text
Agent
Runtime
Tool
Session
Execution
```

---

## 27.1 Agent

管理：

```text
名称
描述
System Prompt
Model
Tools
Runtime
Execution Policy
Version
```

---

## 27.2 Runtime

展示：

```text
PI
Enabled

Harness
Disabled
```

以及 Runtime Capability。

---

## 27.3 Tool

展示：

```text
query_order
query_customer
query_product
search_knowledge
create_ticket
```

可以查看：

```text
Tool Description
Input Schema
Status
```

---

## 27.4 Session

查看：

```text
Session ID
Agent
AgentVersion
Runtime
Runtime Session ID
Status
Created Time
```

---

## 27.5 Execution

查看：

```text
Execution
Input
Output
Runtime
Status
Duration
Token
Tool Calls
Execution Trace
```

这是 V1 最值得重点实现的页面。

---

# 28. Runtime 对比能力

由于平台设计目标包含：

```text
Pi
 ↓
Self Harness
```

建议未来支持同一个 AgentVersion 在不同 Runtime 下运行。

例如：

```text
AfterSaleAgent V5
```

执行：

```text
Run A
runtime = PI
```

和：

```text
Run B
runtime = HARNESS
```

比较：

| 指标 | Pi | Harness |
|---|---:|---:|
| 是否成功 | ✅ | ✅ |
| Duration | 8.1s | 5.2s |
| Tool Calls | 4 | 3 |
| Tokens | 8432 | 6218 |
| Answer Score | 90 | 92 |

这将成为未来自研 Harness 时非常重要的评估能力。

---

# 29. 安全边界

虽然 V1 不重点建设复杂权限体系，但 Runtime 与 Tool 的安全边界不能完全忽略。

业务 Agent 默认不应该拥有：

```text
filesystem read
filesystem write
shell
process execute
arbitrary network access
```

业务 Agent 应优先只开放：

```text
query_order
query_customer
query_product
search_knowledge
create_ticket
send_email
```

即：

> Agent 能做什么，由平台 Tool Registry 和 AgentVersion Tool Binding 决定。

Coding Agent 才考虑开放：

```text
read
write
edit
bash
```

---

# 30. V1 功能范围

## 必须实现

### Agent

- Agent CRUD
- AgentVersion
- System Prompt
- Model 配置
- Runtime 配置
- AgentVersion 发布

### Runtime

- AgentRuntime SPI
- RuntimeRegistry
- RuntimeCapabilities
- PiAgentRuntime
- Pi RPC 接入

### Tool

- Tool Registry
- AgentTool SPI
- AgentVersion Tool Binding
- ToolResolver
- DefaultToolResolver
- Tool Gateway

### Session

- Platform Session
- Runtime Session Mapping
- Session 生命周期

### Execution

- Execution 创建
- RUNNING / SUCCESS / FAILED / ABORT
- Runtime Event
- SSE Streaming
- Execution Trace
- Abort

### Observability

- Runtime
- Duration
- Token
- Tool Call
- Error
- Event Timeline

---

# 31. V1 暂不实现

```text
复杂登录认证
RBAC
部门
组织
租户
复杂数据权限
Agent Marketplace
Prompt Marketplace
复杂审批流
分布式 MQ
复杂调度
Agent Workflow Designer
Multi-Agent Orchestration
Semantic Tool Retrieval
复杂 Agent Evaluation
自动 Runtime 调度
```

原则：

> 先把 Agent Runtime 与 Agent Governance 主链路做扎实。

---

# 32. 后续演进路线

## V1：Pi Agent Platform

目标：

```text
Spring Boot
      │
      ▼
AgentRuntime
      │
      ▼
Pi
      │
      ▼
Tool
      │
      ▼
Execution Trace
```

重点解决：

- AgentVersion
- Runtime 抽象
- Tool Gateway
- Session
- Execution
- Event

---

## V2：Runtime Worker + Tool Governance

增加：

- 独立 Pi Runtime Worker
- Runtime Pool
- Worker Health
- Rule-based ToolResolver
- Tool Group
- Tool Tag
- Session Reuse
- Runtime Metrics

---

## V3：自研 Harness

新增：

```java
HarnessAgentRuntime
```

并实现：

```text
Planner
Executor
Tool Calling
Context Manager
Retry
Reflection
Memory
Compaction
```

然后与 Pi Runtime 做对比。

---

## V4：Agent Evaluation

增加：

```text
Test Case
Dataset
Expected Result
Execution Evaluation
Runtime Comparison
Regression Test
```

用于回答：

```text
Pi Runtime 和自研 Harness 哪个效果更好？
```

---

# 33. 最终核心架构

整个系统最终可以收敛成：

```text
                    Agent Platform

        ┌──────────────────────────────┐
        │          Agent               │
        │                              │
        │ Agent Definition             │
        │ Agent Version                │
        │ Prompt                       │
        │ Model                        │
        │ Allowed Tools                │
        │ Runtime Binding              │
        └──────────────┬───────────────┘
                       │
                       ▼
              AgentSpecCompiler
                       │
                       ▼
              AgentExecutionSpec
                       │
                       ▼
                  ToolResolver
                       │
                       ▼
             RuntimeExecutionSpec
                       │
                       ▼
                 RuntimeRouter
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
   PiRuntimeAdapter          HarnessRuntimeAdapter
          │
          ▼
      Pi Runtime
          │
          ▼
      Tool Gateway
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
   CRM   ERP    RAG
```

外围运行态：

```text
Session
Execution
Event
Trace
```

---

# 34. 关键架构原则总结

建议项目长期坚持以下原则。

## 原则一

> Agent Platform owns Agent.

Agent 的定义、版本、Prompt、Model、Tool 权限由 Spring Boot 平台管理。

---

## 原则二

> Runtime owns Agent Loop.

Think、Act、Tool Call、Observe、Retry、Compaction 等 Loop 细节由 Runtime 实现。

---

## 原则三

> Tool belongs to Platform.

业务 Tool 属于 Agent Platform，不属于 Pi。

---

## 原则四

> Pi is only one Runtime implementation.

Pi 不进入核心 Domain。

Pi 仅实现：

```text
AgentRuntime
```

---

## 原则五

> RuntimeTools ⊆ AgentAllowedTools ⊆ ToolRegistry.

动态 Tool 选择只能缩小范围，不能突破 AgentVersion 的能力边界。

---

## 原则六

> V1 架构允许动态，实际实现保持简单。

V1：

```text
RuntimeTools = AgentAllowedTools
```

先跑通主链路。

未来再增加：

```text
Rule-based ToolResolver
Semantic Tool Retrieval
```

---

# 35. 结论

本项目第一阶段不应该被传统后台系统的基础建设能力占据主要开发时间。

真正需要重点投入的是：

```text
AgentVersion
        +
AgentRuntime SPI
        +
AgentSpecCompiler
        +
Tool Registry
        +
ToolResolver
        +
Tool Gateway
        +
Session
        +
Execution
        +
Event / Trace
```

其中最核心的四个架构骨架为：

```text
AgentRuntime SPI
AgentSpecCompiler
Tool Gateway
Execution Trace
```

Pi 在整个系统中的定位非常明确：

```text
Pi ≠ Agent Platform
```

而是：

```text
Pi implements AgentRuntime
```

因此当前可以充分利用 Pi 已经成熟的 Agent Loop、Session、Tool Calling、Context Management 等 Runtime 能力。

未来需要自研 Harness 时，只需要新增：

```java
HarnessAgentRuntime implements AgentRuntime
```

而 Agent、Tool、Session、Execution、Event 等平台核心模型仍然可以继续复用。

这也是本项目最重要的长期架构价值。
