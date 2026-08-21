# 销售询盘助手 MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个固定销售询盘助手，通过 Spring Boot 管理单 Session 多轮 Execution，使用 Pi RPC 调用两个 Java 模拟 Tool，并通过 SSE 与 MySQL 提供完整 Trace、Abort 和 Timeout 能力。

**Architecture:** Spring Boot 单体拥有固定 Agent 配置、Platform Session、Execution、Tool 与 Trace；每个 ACTIVE Platform Session 绑定一个 Pi RPC 子进程，Pi 负责 Agent Loop 和上下文。Pi Extension 仅代理 `get_customer_info` 与 `get_inquiry_info` 到 Java Internal Tool Gateway，所有外部事件先映射为平台事件再持久化和推送。

**Tech Stack:** Java 21、Maven 3.9+、Spring Boot 4.1.0、Spring MVC、MyBatis-Plus 3.5.17、Flyway、MySQL 8、springdoc-openapi 3.0.3、JUnit 5、Mockito、Pi 0.84.2、TypeScript Pi Extension。

**Version Basis:** [MyBatis-Plus 官方安装文档](https://baomidou.com/en/getting-started/install/)说明从 3.5.13 开始提供 Spring Boot 4 Starter；本计划固定使用官方当前版本 `mybatis-plus-spring-boot4-starter:3.5.17`。

**Spec:** `doc/project/sales-inquiry-agent-mvp-design.md`

## Global Constraints

- 遵循 KISS：不添加 Agent CRUD、前端、认证、多 Runtime、多 Session 并发、进程池和重启恢复。
- Java 包根路径固定为 `com.iumyx.agentops`。
- 每个新 Java 文件的类型级 Javadoc 必须包含 `@author iumyx`、准确的 `@description` 和创建时刻的 `@date yyyy-MM-dd HH:mm`。
- 所有非 getter/setter 方法必须有 Javadoc；说明方法、全部参数、返回值，并说明可能抛出的异常。无返回值或无异常的方法也必须在描述中明确写出。
- 不在源码、配置、测试、文档、日志或 Git 提交中写入 MySQL 密码、Pi 认证信息或 Tool 临时凭证。
- 运行数据库固定为 `java_agent_ops_mvp`；集成测试数据库固定为 `java_agent_ops_mvp_test`。
- 运行数据库配置只读取 `MVP_DB_URL`、`MVP_DB_USERNAME`、`MVP_DB_PASSWORD`。
- 测试数据库配置只读取 `MVP_TEST_DB_URL`、`MVP_TEST_DB_USERNAME`、`MVP_TEST_DB_PASSWORD`。
- Pi 固定使用 `zai-coding-cn/glm-5.3`，执行超时固定为 120 秒。
- Pi 必须禁用内置 Tool、自动 Extension、Skill、Prompt Template 和上下文文件。
- 模型原始思考过程不得进入平台 Event、MySQL、SSE 或日志。
- 每个任务必须先运行指定测试看到预期失败，再实现最小代码并运行到通过。
- 每个任务只提交其列出的文件；不得提交现有 `.gitignore` 修改或其他未跟踪文件。
- 文中的 Java 代码块展示必须实现的行为与签名；落盘时连同测试方法一起补齐本节要求的类型级和方法级 Javadoc，不得原样省略注释。

## File Structure

```text
pom.xml                                         Maven 依赖与构建配置
pi/extensions/sales-tool-proxy.ts              两个 Pi Tool 的 HTTP 代理
src/main/java/com/iumyx/agentops/
├── AgentOpsApplication.java                   Spring Boot 入口
├── config/
│   ├── FixedAgentProperties.java               固定 Agent 配置
│   ├── PiRuntimeProperties.java                Pi 可执行文件与 Gateway 配置
│   └── AsyncConfiguration.java                 Execution 异步执行器与调度器
├── common/
│   ├── ApiError.java                           稳定错误响应
│   ├── ErrorCode.java                          稳定错误码
│   └── GlobalExceptionHandler.java             HTTP 异常映射
├── session/
│   ├── AgentSession.java                       Platform Session 模型
│   ├── SessionStatus.java                      ACTIVE/CLOSED
│   ├── SessionMapper.java                      MyBatis-Plus Session Mapper
│   ├── SessionRuntimeContext.java              内存 Runtime/凭证上下文
│   ├── SessionRuntimeContextRegistry.java      单 ACTIVE Session 内存注册表
│   └── SessionService.java                     Session 生命周期
├── execution/
│   ├── AgentExecution.java                     Execution 模型
│   ├── ExecutionStatus.java                    Execution 状态
│   ├── PlatformEventType.java                  平台事件类型
│   ├── ExecutionEvent.java                     平台事件模型
│   ├── ExecutionMapper.java                    MyBatis-Plus Execution Mapper
│   ├── ExecutionEventMapper.java               MyBatis-Plus Event Mapper
│   ├── ActiveExecutionRegistry.java            Session 到 RUNNING Execution 映射
│   ├── ExecutionAborter.java                    Session 关闭时的中止契约
│   ├── PlatformEventListener.java              Event 实时监听契约
│   ├── PlatformEventPublisher.java             排序、持久化、实时发布
│   └── ExecutionService.java                   执行、完成、失败、中止和超时
├── runtime/
│   ├── AgentRuntime.java                       Runtime SPI
│   ├── AgentRuntimeRegistry.java               Runtime 注册表
│   ├── RuntimeSessionRequest.java              创建 Runtime Session 请求
│   ├── RuntimeSession.java                     Runtime Session 结果
│   ├── RuntimeExecutionRequest.java            Runtime 执行请求
│   ├── RuntimeEvent.java                       统一 Runtime Event
│   ├── RuntimeEventType.java                   Runtime Event 类型
│   ├── RuntimeCompletion.java                  Runtime 完成结果
│   ├── RuntimeEventListener.java               Runtime 回调契约
│   └── pi/
│       ├── PiAgentRuntime.java                 Pi Runtime 实现
│       ├── PiEventMapper.java                  Pi Event 映射
│       ├── PiProcessLauncher.java              进程启动契约
│       ├── SystemPiProcessLauncher.java        ProcessBuilder 实现
│       └── PiRpcClient.java                    JSONL RPC 客户端
├── tool/
│   ├── AgentTool.java                          Java Tool SPI
│   ├── ToolResult.java                         Tool 结果
│   ├── ToolRegistry.java                       Tool 注册表
│   ├── GetCustomerInfoTool.java                固定客户样例
│   ├── GetInquiryInfoTool.java                 固定询盘样例
│   ├── ToolInvocationRequest.java              Internal Tool 请求
│   ├── ToolInvocationResponse.java             Internal Tool 响应
│   └── ToolGatewayController.java              Internal Tool Gateway
├── api/
│   ├── SessionController.java                  Session HTTP API
│   ├── ExecutionController.java                Execution HTTP API
│   ├── ExecutionSseController.java             SSE API
│   ├── ExecutionEventStream.java               实时 SSE 订阅表
│   └── dto/                                    请求和响应 record
└── recovery/
    └── StartupRecovery.java                    启动清理遗留状态

src/main/resources/
├── application.yml                             外部配置占位
└── db/migration/V1__create_agent_runtime.sql   三张 MVP 表

src/test/java/com/iumyx/agentops/                与主包一一对应的测试
src/test/resources/application-mysql-test.yml    MySQL 测试 Profile
doc/project/sales-inquiry-agent-mvp-runbook.md   启动与验收手册
scripts/sales-inquiry-mvp-smoke.sh               HTTP/SSE 冒烟脚本
```

---

### Task 1: Maven 工程与固定 Agent 配置

**Files:**
- Create: `pom.xml`
- Create: `src/main/java/com/iumyx/agentops/AgentOpsApplication.java`
- Create: `src/main/java/com/iumyx/agentops/config/FixedAgentProperties.java`
- Create: `src/main/java/com/iumyx/agentops/config/PiRuntimeProperties.java`
- Create: `src/main/resources/application.yml`
- Test: `src/test/java/com/iumyx/agentops/config/FixedAgentPropertiesTest.java`

**Interfaces:**
- Produces: `FixedAgentProperties(String agentKey, String provider, String model, Set<String> allowedTools, Duration timeout, String systemPrompt)`.
- Produces: `PiRuntimeProperties(String executable, Path extensionPath, URI toolGatewayBaseUrl)`.
- Consumes: Java 21 and Maven 3.9+ installed on the host.

- [ ] **Step 1: Create `pom.xml` and write the failing configuration binding test**

Use Spring Boot parent `4.1.0`. Add `spring-boot-starter-webmvc`, `spring-boot-starter-validation`, `spring-boot-starter-actuator`, `com.baomidou:mybatis-plus-spring-boot4-starter:3.5.17`, `flyway-core`, `flyway-mysql`, runtime `mysql-connector-j`, `springdoc-openapi-starter-webmvc-ui:3.0.3`, `spring-boot-starter-test`, and test-scope `org.awaitility:awaitility`. Do not add `spring-boot-starter-jdbc` separately because the MyBatis-Plus starter already brings the required JDBC infrastructure.

```java
class FixedAgentPropertiesTest {

    private final ApplicationContextRunner contextRunner = new ApplicationContextRunner()
            .withUserConfiguration(TestConfiguration.class)
            .withPropertyValues(
                    "agent.fixed.agent-key=sales-inquiry-assistant",
                    "agent.fixed.provider=zai-coding-cn",
                    "agent.fixed.model=glm-5.3",
                    "agent.fixed.allowed-tools[0]=get_customer_info",
                    "agent.fixed.allowed-tools[1]=get_inquiry_info",
                    "agent.fixed.timeout=120s",
                    "agent.fixed.system-prompt=You are a sales inquiry assistant",
                    "agent.runtime.pi.executable=pi",
                    "agent.runtime.pi.extension-path=pi/extensions/sales-tool-proxy.ts",
                    "agent.runtime.pi.tool-gateway-base-url=http://127.0.0.1:8080");

    @Test
    void shouldBindFixedAgentAndPiRuntimeProperties() {
        contextRunner.run(context -> {
            FixedAgentProperties agent = context.getBean(FixedAgentProperties.class);
            assertThat(agent.model()).isEqualTo("glm-5.3");
            assertThat(agent.allowedTools()).containsExactlyInAnyOrder(
                    "get_customer_info", "get_inquiry_info");
            assertThat(agent.timeout()).isEqualTo(Duration.ofSeconds(120));
            assertThat(context.getBean(PiRuntimeProperties.class).executable()).isEqualTo("pi");
        });
    }

    @Configuration(proxyBeanMethods = false)
    @EnableConfigurationProperties({FixedAgentProperties.class, PiRuntimeProperties.class})
    static class TestConfiguration {
    }
}
```

- [ ] **Step 2: Run the test and verify the missing configuration classes fail compilation**

Run: `mvn -q -Dtest=FixedAgentPropertiesTest test`

Expected: FAIL with compilation errors for `FixedAgentProperties` and `PiRuntimeProperties`.

- [ ] **Step 3: Implement the application entry point and immutable properties**

```java
@Validated
@ConfigurationProperties("agent.fixed")
public record FixedAgentProperties(
        @NotBlank String agentKey,
        @NotBlank String provider,
        @NotBlank String model,
        @NotEmpty Set<String> allowedTools,
        @NotNull Duration timeout,
        @NotBlank String systemPrompt) {
}

@Validated
@ConfigurationProperties("agent.runtime.pi")
public record PiRuntimeProperties(
        @NotBlank String executable,
        @NotNull Path extensionPath,
        @NotNull URI toolGatewayBaseUrl) {
}
```

`AgentOpsApplication` must use `@SpringBootApplication` and `@ConfigurationPropertiesScan`. `application.yml` must use only environment-variable indirection:

```yaml
spring:
  application:
    name: java-agent-ops
  datasource:
    url: ${MVP_DB_URL:jdbc:mysql://127.0.0.1:3306/java_agent_ops_mvp}
    username: ${MVP_DB_USERNAME:}
    password: ${MVP_DB_PASSWORD:}
  flyway:
    enabled: true

mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
    default-enum-type-handler: org.apache.ibatis.type.EnumTypeHandler

agent:
  fixed:
    agent-key: sales-inquiry-assistant
    provider: zai-coding-cn
    model: glm-5.3
    allowed-tools: [get_customer_info, get_inquiry_info]
    timeout: 120s
    system-prompt: |-
      你是销售询盘助手。仅依据用户输入和工具结果回答；缺少数据时明确说明，不得编造。
      需要客户信息时调用 get_customer_info，需要询盘信息时调用 get_inquiry_info。
      最终回答必须包含：客户概况、询盘摘要、销售建议、下一步行动。
  runtime:
    pi:
      executable: ${PI_EXECUTABLE:pi}
      extension-path: ${PI_EXTENSION_PATH:pi/extensions/sales-tool-proxy.ts}
      tool-gateway-base-url: ${TOOL_GATEWAY_BASE_URL:http://127.0.0.1:8080}
```

- [ ] **Step 4: Run the focused test and compile**

Run: `mvn -q -Dtest=FixedAgentPropertiesTest test && mvn -q -DskipTests compile`

Expected: PASS; compilation completes without warnings about missing configuration properties.

- [ ] **Step 5: Commit**

```bash
git add pom.xml src/main/java/com/iumyx/agentops/AgentOpsApplication.java \
  src/main/java/com/iumyx/agentops/config src/main/resources/application.yml \
  src/test/java/com/iumyx/agentops/config/FixedAgentPropertiesTest.java
git commit -m "build: bootstrap sales inquiry agent service"
```

### Task 2: Flyway Schema 与 MyBatis-Plus 持久化

**Files:**
- Create: `src/main/resources/db/migration/V1__create_agent_runtime.sql`
- Create: `src/main/java/com/iumyx/agentops/session/SessionStatus.java`
- Create: `src/main/java/com/iumyx/agentops/session/AgentSession.java`
- Create: `src/main/java/com/iumyx/agentops/session/SessionMapper.java`
- Create: `src/main/java/com/iumyx/agentops/execution/ExecutionStatus.java`
- Create: `src/main/java/com/iumyx/agentops/execution/PlatformEventType.java`
- Create: `src/main/java/com/iumyx/agentops/execution/AgentExecution.java`
- Create: `src/main/java/com/iumyx/agentops/execution/ExecutionEvent.java`
- Create: `src/main/java/com/iumyx/agentops/execution/ExecutionMapper.java`
- Create: `src/main/java/com/iumyx/agentops/execution/ExecutionEventMapper.java`
- Create: `src/test/resources/application-mysql-test.yml`
- Test: `src/test/java/com/iumyx/agentops/persistence/PersistenceIntegrationTest.java`

**Interfaces:**
- Produces: `SessionMapper extends BaseMapper<AgentSession>`.
- Produces: `ExecutionMapper extends BaseMapper<AgentExecution>`.
- Produces: `ExecutionEventMapper extends BaseMapper<ExecutionEvent>` with `long selectNextSequence(String executionId)`.
- Uses: MyBatis-Plus `LambdaQueryWrapper` and `LambdaUpdateWrapper` for CRUD, active-session lookup, running-Execution lookup, conditional terminal updates and startup recovery.
- Consumes: MySQL test schema `java_agent_ops_mvp_test` and the three `MVP_TEST_DB_*` environment variables.

- [ ] **Step 1: Create the test profile and failing Mapper integration test**

```yaml
spring:
  datasource:
    url: ${MVP_TEST_DB_URL}
    username: ${MVP_TEST_DB_USERNAME}
    password: ${MVP_TEST_DB_PASSWORD}
  flyway:
    clean-disabled: false
```

```java
@SpringBootTest
@ActiveProfiles("mysql-test")
@EnabledIfEnvironmentVariable(named = "RUN_MYSQL_INTEGRATION", matches = "true")
class PersistenceIntegrationTest {

    @Autowired SessionMapper sessions;
    @Autowired ExecutionMapper executions;
    @Autowired ExecutionEventMapper events;

    @BeforeEach
    void clearTables() {
        events.delete(null);
        executions.delete(null);
        sessions.delete(null);
    }

    @Test
    void shouldPersistSessionExecutionAndOrderedEvents() {
        Instant now = Instant.now();
        AgentSession session = new AgentSession("sess-1", "sales-inquiry-assistant",
                "pi", "pi-session-1", SessionStatus.ACTIVE, now, null);
        sessions.insert(session);

        AgentExecution execution = AgentExecution.created("exec-1", session.getId(), "hello", now);
        executions.insert(execution);
        int runningUpdated = executions.update(null,
                Wrappers.<AgentExecution>lambdaUpdate()
                        .eq(AgentExecution::getId, execution.getId())
                        .eq(AgentExecution::getStatus, ExecutionStatus.CREATED)
                        .set(AgentExecution::getStatus, ExecutionStatus.RUNNING)
                        .set(AgentExecution::getStartedAt, now));
        assertThat(runningUpdated).isEqualTo(1);

        events.insert(new ExecutionEvent("event-1", execution.getId(), 1,
                PlatformEventType.RUN_STARTED, "{}", now));
        assertThat(events.selectList(Wrappers.<ExecutionEvent>lambdaQuery()
                        .eq(ExecutionEvent::getExecutionId, execution.getId())
                        .orderByAsc(ExecutionEvent::getSequenceNo)))
                .extracting(ExecutionEvent::getSequenceNo)
                .containsExactly(1L);
    }
}
```

- [ ] **Step 2: Run the integration test and verify missing tables/types fail**

Run: `RUN_MYSQL_INTEGRATION=true mvn -q -Dspring.profiles.active=mysql-test -Dtest=PersistenceIntegrationTest test`

Expected: FAIL because the migration, entity and Mapper types do not exist.

- [ ] **Step 3: Create the exact Flyway migration**

```sql
CREATE TABLE agent_session (
    id VARCHAR(36) PRIMARY KEY,
    agent_key VARCHAR(100) NOT NULL,
    runtime_type VARCHAR(32) NOT NULL,
    runtime_session_id VARCHAR(128) NOT NULL,
    status VARCHAR(16) NOT NULL,
    created_at DATETIME(6) NOT NULL,
    closed_at DATETIME(6) NULL,
    INDEX idx_agent_session_status (status)
);

CREATE TABLE agent_execution (
    id VARCHAR(36) PRIMARY KEY,
    session_id VARCHAR(36) NOT NULL,
    status VARCHAR(16) NOT NULL,
    input LONGTEXT NOT NULL,
    output LONGTEXT NULL,
    error_code VARCHAR(64) NULL,
    error_message VARCHAR(500) NULL,
    input_tokens BIGINT NOT NULL DEFAULT 0,
    output_tokens BIGINT NOT NULL DEFAULT 0,
    tool_call_count INT NOT NULL DEFAULT 0,
    started_at DATETIME(6) NULL,
    finished_at DATETIME(6) NULL,
    created_at DATETIME(6) NOT NULL,
    CONSTRAINT fk_execution_session FOREIGN KEY (session_id) REFERENCES agent_session(id),
    INDEX idx_execution_session_status (session_id, status)
);

CREATE TABLE agent_execution_event (
    id VARCHAR(36) PRIMARY KEY,
    execution_id VARCHAR(36) NOT NULL,
    sequence_no BIGINT NOT NULL,
    event_type VARCHAR(32) NOT NULL,
    event_data JSON NOT NULL,
    created_at DATETIME(6) NOT NULL,
    CONSTRAINT fk_event_execution FOREIGN KEY (execution_id) REFERENCES agent_execution(id),
    CONSTRAINT uk_event_execution_sequence UNIQUE (execution_id, sequence_no)
);
```

- [ ] **Step 4: Implement MyBatis-Plus entities and Mappers**

Use ordinary Java entity classes with a no-argument constructor, an all-arguments constructor, explicit getters/setters, `@TableName`, and `@TableId(type = IdType.INPUT)`. Do not add Lombok, XML Mapper files, generic Repository wrappers, or MyBatis-Plus code generator. `AgentExecution` must provide the exact factory `created(String id, String sessionId, String input, Instant createdAt)`, returning zero usage counters and status `CREATED`.

```java
public enum SessionStatus {
    ACTIVE, CLOSED
}

public enum ExecutionStatus {
    CREATED, RUNNING, SUCCEEDED, FAILED, ABORTED, TIMEOUT;

    public boolean isTerminal() {
        return this == SUCCEEDED || this == FAILED || this == ABORTED || this == TIMEOUT;
    }
}

public enum PlatformEventType {
    RUN_STARTED, MESSAGE_DELTA, TOOL_CALL_STARTED, TOOL_CALL_COMPLETED,
    USAGE_UPDATED, RUN_COMPLETED, RUN_FAILED, RUN_ABORTED, RUN_TIMEOUT
}
```

Use the exact model fields from the migration: `AgentSession(id, agentKey, runtimeType, runtimeSessionId, status, createdAt, closedAt)`, `AgentExecution(id, sessionId, status, input, output, errorCode, errorMessage, inputTokens, outputTokens, toolCallCount, startedAt, finishedAt, createdAt)`, and `ExecutionEvent(id, executionId, sequenceNo, eventType, eventData, createdAt)`.

```java
@Mapper
public interface SessionMapper extends BaseMapper<AgentSession> {
}

@Mapper
public interface ExecutionMapper extends BaseMapper<AgentExecution> {
}

@Mapper
public interface ExecutionEventMapper extends BaseMapper<ExecutionEvent> {

    @Select("""
            SELECT COALESCE(MAX(sequence_no), 0) + 1
              FROM agent_execution_event
             WHERE execution_id = #{executionId}
            """)
    long selectNextSequence(@Param("executionId") String executionId);
}
```

Configure `mybatis-plus.configuration.map-underscore-to-camel-case: true` and `default-enum-type-handler: org.apache.ibatis.type.EnumTypeHandler` in `application.yml`. `selectNextSequence(executionId)` is the only custom SQL in this task and must run inside the publisher's serialized critical section. Map all timestamps through UTC `Instant`.

- [ ] **Step 5: Re-run persistence tests**

Run: `RUN_MYSQL_INTEGRATION=true mvn -q -Dspring.profiles.active=mysql-test -Dtest=PersistenceIntegrationTest test`

Expected: PASS and Flyway reports migration version `1`.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/db src/test/resources/application-mysql-test.yml \
  src/main/java/com/iumyx/agentops/session \
  src/main/java/com/iumyx/agentops/execution \
  src/test/java/com/iumyx/agentops/persistence/PersistenceIntegrationTest.java
git commit -m "feat: persist sessions executions and events"
```

### Task 3: Java Tool SPI、Registry 与固定模拟数据

**Files:**
- Create: `src/main/java/com/iumyx/agentops/tool/AgentTool.java`
- Create: `src/main/java/com/iumyx/agentops/tool/ToolResult.java`
- Create: `src/main/java/com/iumyx/agentops/tool/ToolRegistry.java`
- Create: `src/main/java/com/iumyx/agentops/tool/GetCustomerInfoTool.java`
- Create: `src/main/java/com/iumyx/agentops/tool/GetInquiryInfoTool.java`
- Test: `src/test/java/com/iumyx/agentops/tool/ToolRegistryTest.java`
- Test: `src/test/java/com/iumyx/agentops/tool/GetCustomerInfoToolTest.java`
- Test: `src/test/java/com/iumyx/agentops/tool/GetInquiryInfoToolTest.java`

**Interfaces:**
- Produces: `AgentTool#name()`, `description()`, `execute(Map<String, Object>)`.
- Produces: `ToolRegistry#getRequired(String)` and `contains(String)`.
- Produces: `ToolResult(boolean found, Map<String, Object> data, String message)`.
- Consumes: no database or Runtime dependency.

- [ ] **Step 1: Write failing Tool behavior and duplicate-name tests**

```java
@Test
void shouldReturnKnownCustomerAndRejectUnknownCustomerWithoutFabrication() {
    GetCustomerInfoTool tool = new GetCustomerInfoTool();
    ToolResult known = tool.execute(Map.of("customerId", "C001"));
    ToolResult unknown = tool.execute(Map.of("customerId", "C999"));

    assertThat(known.found()).isTrue();
    assertThat(known.data()).containsEntry("companyName", "示例科技有限公司");
    assertThat(unknown.found()).isFalse();
    assertThat(unknown.data()).isEmpty();
}

@Test
void shouldRejectDuplicateToolNames() {
    AgentTool first = mockTool("duplicate");
    AgentTool second = mockTool("duplicate");
    assertThatThrownBy(() -> new ToolRegistry(List.of(first, second)))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("duplicate");
}

private AgentTool mockTool(String name) {
    AgentTool tool = mock(AgentTool.class);
    when(tool.name()).thenReturn(name);
    return tool;
}
```

- [ ] **Step 2: Run tests and verify missing Tool types fail**

Run: `mvn -q -Dtest=ToolRegistryTest,GetCustomerInfoToolTest,GetInquiryInfoToolTest test`

Expected: FAIL with missing Tool classes.

- [ ] **Step 3: Implement the minimal Tool SPI and fixed maps**

```java
public interface AgentTool {
    String name();
    String description();
    ToolResult execute(Map<String, Object> arguments);
}

public record ToolResult(boolean found, Map<String, Object> data, String message) {
    public static ToolResult found(Map<String, Object> data) {
        return new ToolResult(true, Map.copyOf(data), "");
    }

    public static ToolResult notFound() {
        return new ToolResult(false, Map.of(), "未找到对应的模拟数据");
    }
}
```

`GetCustomerInfoTool` must require nonblank `customerId` and return the exact `C001` data from the spec. `GetInquiryInfoTool` must require nonblank `inquiryId` and return the exact `I001` data. Missing arguments throw `IllegalArgumentException` with stable, non-sensitive messages.

- [ ] **Step 4: Run Tool tests**

Run: `mvn -q -Dtest=ToolRegistryTest,GetCustomerInfoToolTest,GetInquiryInfoToolTest test`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/iumyx/agentops/tool \
  src/test/java/com/iumyx/agentops/tool
git commit -m "feat: add sales inquiry tools"
```

### Task 4: Runtime SPI、统一事件与 Fake Runtime

**Files:**
- Create: `src/main/java/com/iumyx/agentops/runtime/AgentRuntime.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/AgentRuntimeRegistry.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/RuntimeSessionRequest.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/RuntimeSession.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/RuntimeExecutionRequest.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/RuntimeEvent.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/RuntimeEventType.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/RuntimeCompletion.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/RuntimeEventListener.java`
- Test: `src/test/java/com/iumyx/agentops/runtime/AgentRuntimeRegistryTest.java`
- Test support: `src/test/java/com/iumyx/agentops/runtime/FakeAgentRuntime.java`

**Interfaces:**
- Produces: the Runtime contract used by Session and Execution tasks.
- Produces: `FakeAgentRuntime` with deterministic `emit`, `complete`, and `fail` controls.
- Consumes: no Pi-specific types.

- [ ] **Step 1: Write failing registry tests**

```java
@Test
void shouldResolveRuntimeByTypeAndRejectUnknownType() {
    FakeAgentRuntime runtime = new FakeAgentRuntime("pi");
    AgentRuntimeRegistry registry = new AgentRuntimeRegistry(List.of(runtime));

    assertThat(registry.getRequired("pi")).isSameAs(runtime);
    assertThatThrownBy(() -> registry.getRequired("missing"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("Runtime not found: missing");
}
```

- [ ] **Step 2: Run test and verify missing Runtime contract fails**

Run: `mvn -q -Dtest=AgentRuntimeRegistryTest test`

Expected: FAIL with missing `AgentRuntime` and `AgentRuntimeRegistry`.

- [ ] **Step 3: Implement the exact SPI**

```java
public interface AgentRuntime {
    String type();
    RuntimeSession createSession(RuntimeSessionRequest request);
    void execute(String runtimeSessionId, RuntimeExecutionRequest request,
            RuntimeEventListener listener);
    void abort(String runtimeSessionId);
    void closeSession(String runtimeSessionId);
}

public interface RuntimeEventListener {
    void onEvent(RuntimeEvent event);
    void onCompleted(RuntimeCompletion completion);
    void onFailed(String errorCode, String safeMessage);
}

public record RuntimeCompletion(
        String output, long inputTokens, long outputTokens, int toolCallCount) {
}
```

`RuntimeSessionRequest` fields must be `platformSessionId`, `systemPrompt`, `provider`, `model`, `allowedTools`, `toolGatewayBaseUrl`, and `toolToken`. `RuntimeExecutionRequest` fields must be `executionId` and `message`. `RuntimeEvent` fields must be `RuntimeEventType type` and `Map<String, Object> data`.

```java
public record RuntimeSessionRequest(String platformSessionId, String systemPrompt,
        String provider, String model, Set<String> allowedTools,
        URI toolGatewayBaseUrl, String toolToken) {
}

public record RuntimeSession(String runtimeType, String runtimeSessionId) {
}

public record RuntimeExecutionRequest(String executionId, String message) {
}

public record RuntimeEvent(RuntimeEventType type, Map<String, Object> data) {
}
```

`RuntimeEventType` must contain exactly `RUN_STARTED`, `MESSAGE_DELTA`, `TOOL_CALL_STARTED`, `TOOL_CALL_COMPLETED`, and `USAGE_UPDATED`. Runtime terminal outcomes use `onCompleted` and `onFailed`, not additional event enum values.

- [ ] **Step 4: Implement Fake Runtime and run tests**

Run: `mvn -q -Dtest=AgentRuntimeRegistryTest test`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/iumyx/agentops/runtime \
  src/test/java/com/iumyx/agentops/runtime
git commit -m "feat: define agent runtime contract"
```

### Task 5: Pi RPC 进程、JSONL 协议与事件映射

**Files:**
- Create: `src/main/java/com/iumyx/agentops/runtime/pi/PiProcessLauncher.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/pi/SystemPiProcessLauncher.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/pi/PiRpcClient.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/pi/PiEventMapper.java`
- Create: `src/main/java/com/iumyx/agentops/runtime/pi/PiAgentRuntime.java`
- Test: `src/test/java/com/iumyx/agentops/runtime/pi/SystemPiProcessLauncherTest.java`
- Test: `src/test/java/com/iumyx/agentops/runtime/pi/PiEventMapperTest.java`
- Test: `src/test/java/com/iumyx/agentops/runtime/pi/PiAgentRuntimeTest.java`
- Test fixture: `src/test/resources/pi/events.jsonl`

**Interfaces:**
- Consumes: all Runtime interfaces from Task 4 and properties from Task 1.
- Produces: Spring bean `PiAgentRuntime` with type `pi`.
- Produces: `CompletableFuture<JsonNode> PiRpcClient.sendCommand(ObjectNode command)`, `void prompt(RuntimeExecutionRequest request, RuntimeEventListener listener)`, `void abort()`, `JsonNode getState()`, `void close()`.

- [ ] **Step 1: Write failing command construction and event mapping tests**

```java
@Test
void shouldDisableAllUnapprovedPiCapabilities() {
    List<String> command = launcher.commandFor(request());
    assertThat(command).containsSubsequence("pi", "--mode", "rpc", "--no-session");
    assertThat(command).contains("--no-builtin-tools", "--no-extensions",
            "--no-skills", "--no-prompt-templates", "--no-context-files");
    assertThat(command).containsSubsequence("--provider", "zai-coding-cn");
    assertThat(command).containsSubsequence("--model", "glm-5.3");
    assertThat(String.join(" ", command)).doesNotContain("tool-token");
}

private RuntimeSessionRequest request() {
    return new RuntimeSessionRequest("sess-1", "fixed prompt", "zai-coding-cn",
            "glm-5.3", Set.of("get_customer_info", "get_inquiry_info"),
            URI.create("http://127.0.0.1:8080"), "tool-token");
}

@ParameterizedTest
@CsvSource({
        "agent_start,RUN_STARTED",
        "tool_execution_start,TOOL_CALL_STARTED",
        "tool_execution_end,TOOL_CALL_COMPLETED"
})
void shouldMapSupportedPiEvents(String piType, RuntimeEventType expected) {
    assertThat(mapper.map(objectMapper.readTree("{\"type\":\"" + piType + "\"}")))
            .get().extracting(RuntimeEvent::type).isEqualTo(expected);
}
```

- [ ] **Step 2: Run focused tests and verify missing Pi classes fail**

Run: `mvn -q -Dtest=SystemPiProcessLauncherTest,PiEventMapperTest,PiAgentRuntimeTest test`

Expected: FAIL with missing Pi adapter classes.

- [ ] **Step 3: Implement safe ProcessBuilder command and environment**

The command list must be built without a shell:

```java
List<String> command = List.of(
        properties.executable(), "--mode", "rpc", "--no-session",
        "--no-builtin-tools", "--no-extensions", "--extension",
        properties.extensionPath().toString(), "--no-skills",
        "--no-prompt-templates", "--no-context-files",
        "--system-prompt", request.systemPrompt(),
        "--provider", request.provider(), "--model", request.model());
ProcessBuilder builder = new ProcessBuilder(command);
builder.environment().put("AGENT_PLATFORM_SESSION_ID", request.platformSessionId());
builder.environment().put("AGENT_TOOL_GATEWAY_URL", request.toolGatewayBaseUrl().toString());
builder.environment().put("AGENT_TOOL_TOKEN", request.toolToken());
```

Never log `command`, the environment, System Prompt, or token.

- [ ] **Step 4: Implement JSONL client and Pi Runtime orchestration**

`PiRpcClient` must:

1. Read stdout with `BufferedReader.readLine()` because Pi framing is LF-delimited JSONL.
2. Consume stderr on a separate daemon thread and log only a generic line count plus process exit code.
3. Correlate command responses with a generated UUID `id` and `CompletableFuture<JsonNode>`.
4. Reject malformed stdout with `PI_PROTOCOL_ERROR` without forwarding raw content.
5. On `agent_settled`, request `get_last_assistant_text` and `get_session_stats`, then call `onCompleted`.
6. On unexpected process exit, call `onFailed("PI_PROCESS_EXITED", "Pi runtime process exited unexpectedly")`.
7. When `tool_execution_end.isError` is true, emit a sanitized TOOL_CALL_COMPLETED event, abort the current Pi run, and call `onFailed("TOOL_PROXY_FAILED", "Tool proxy execution failed")`.

`PiEventMapper` must ignore thinking deltas and unsupported events. For `message_update`, only map `assistantMessageEvent.type == "text_delta"` to `MESSAGE_DELTA`. Tool events may retain only `toolName`, `toolCallId`, `success`, `customerId`, `inquiryId`, and the fixed mock result fields from the spec; never forward an entire raw Pi event.

- [ ] **Step 5: Run Pi adapter tests**

Run: `mvn -q -Dtest=SystemPiProcessLauncherTest,PiEventMapperTest,PiAgentRuntimeTest test`

Expected: PASS using a fake `PiProcessLauncher`; no real model call occurs.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/com/iumyx/agentops/runtime/pi \
  src/test/java/com/iumyx/agentops/runtime/pi src/test/resources/pi
git commit -m "feat: integrate pi rpc runtime"
```

### Task 6: Platform Session 生命周期

**Files:**
- Create: `src/main/java/com/iumyx/agentops/session/SessionRuntimeContext.java`
- Create: `src/main/java/com/iumyx/agentops/session/SessionRuntimeContextRegistry.java`
- Create: `src/main/java/com/iumyx/agentops/session/SessionService.java`
- Test: `src/test/java/com/iumyx/agentops/session/SessionServiceTest.java`

**Interfaces:**
- Consumes: `FixedAgentProperties`, `PiRuntimeProperties`, `SessionMapper`, `AgentRuntimeRegistry`.
- Produces: `AgentSession SessionService.createSession()`, `void closeSession(String sessionId)`, `AgentSession getRequired(String sessionId)`.
- Produces: `SessionRuntimeContext(platformSessionId, runtimeSessionId, runtimeType, toolToken)`.

- [ ] **Step 1: Write failing lifecycle tests**

```java
@Test
void shouldCreateOnlyOneActiveSessionAndCloseRuntimeOnSessionClose() {
    AgentSession created = service.createSession();
    assertThat(created.getStatus()).isEqualTo(SessionStatus.ACTIVE);
    assertThatThrownBy(service::createSession)
            .isInstanceOf(IllegalStateException.class)
            .hasMessage("An active session already exists");

    service.closeSession(created.getId());
    assertThat(sessionMapper.selectById(created.getId()).getStatus())
            .isEqualTo(SessionStatus.CLOSED);
    assertThat(fakeRuntime.closedSessionIds()).contains(created.getRuntimeSessionId());
}

@Test
void shouldCloseRuntimeWhenPersistenceFailsAfterProcessStart() {
    doThrow(new DataAccessResourceFailureException("db unavailable"))
            .when(sessionMapper).insert(any());
    assertThatThrownBy(service::createSession).isInstanceOf(DataAccessException.class);
    assertThat(fakeRuntime.closedSessionIds()).hasSize(1);
}
```

- [ ] **Step 2: Run test and verify Session service is missing**

Run: `mvn -q -Dtest=SessionServiceTest test`

Expected: FAIL with missing Session runtime context and service.

- [ ] **Step 3: Implement single-session lifecycle and secure token generation**

Generate a 256-bit token with `SecureRandom`, Base64 URL encoding without padding. `createSession()` must execute this exact order:

```text
query `SessionMapper` with a lambda wrapper and confirm no ACTIVE Session
generate platformSessionId and toolToken
call runtime.createSession(request)
insert ACTIVE AgentSession through `SessionMapper`
register in-memory SessionRuntimeContext
return AgentSession
```

If persistence or registry setup fails, close the newly created Runtime Session before rethrowing. `closeSession()` must abort a running execution through a callback added in Task 7, close Runtime, update MySQL, then remove the in-memory context. Do not persist the token.

Use MyBatis-Plus wrappers directly; do not recreate a Repository façade:

```java
private AgentSession findActiveSession() {
    return sessionMapper.selectOne(Wrappers.<AgentSession>lambdaQuery()
            .eq(AgentSession::getStatus, SessionStatus.ACTIVE)
            .last("LIMIT 1"));
}

private int markClosed(String sessionId, Instant closedAt) {
    return sessionMapper.update(null, Wrappers.<AgentSession>lambdaUpdate()
            .eq(AgentSession::getId, sessionId)
            .eq(AgentSession::getStatus, SessionStatus.ACTIVE)
            .set(AgentSession::getStatus, SessionStatus.CLOSED)
            .set(AgentSession::getClosedAt, closedAt));
}
```

- [ ] **Step 4: Run Session tests**

Run: `mvn -q -Dtest=SessionServiceTest test`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/iumyx/agentops/session/SessionRuntimeContext.java \
  src/main/java/com/iumyx/agentops/session/SessionRuntimeContextRegistry.java \
  src/main/java/com/iumyx/agentops/session/SessionService.java \
  src/test/java/com/iumyx/agentops/session/SessionServiceTest.java
git commit -m "feat: manage platform session lifecycle"
```

### Task 7: Execution、事件发布、Timeout 与 Abort

**Files:**
- Create: `src/main/java/com/iumyx/agentops/config/AsyncConfiguration.java`
- Create: `src/main/java/com/iumyx/agentops/execution/ActiveExecutionRegistry.java`
- Create: `src/main/java/com/iumyx/agentops/execution/ExecutionAborter.java`
- Create: `src/main/java/com/iumyx/agentops/execution/PlatformEventListener.java`
- Create: `src/main/java/com/iumyx/agentops/execution/PlatformEventPublisher.java`
- Create: `src/main/java/com/iumyx/agentops/execution/ExecutionService.java`
- Modify: `src/main/java/com/iumyx/agentops/session/SessionService.java`
- Test: `src/test/java/com/iumyx/agentops/execution/PlatformEventPublisherTest.java`
- Test: `src/test/java/com/iumyx/agentops/execution/ExecutionServiceTest.java`

**Interfaces:**
- Consumes: Runtime, Session and MyBatis-Plus Mappers from Tasks 2, 4 and 6.
- Produces: `AgentExecution ExecutionService.createExecution(String sessionId, String message)`, `void abortExecution(String executionId)`, `void failFromToolGateway(String executionId, String errorCode, String safeMessage)`, `AgentExecution getRequired(String executionId)`, `List<ExecutionEvent> trace(String executionId)`.
- Produces: `void ActiveExecutionRegistry.register(String sessionId, String executionId)`, `Optional<String> findBySessionId(String sessionId)`, `void removeByExecutionId(String executionId)`.
- Produces: `ExecutionEvent PlatformEventPublisher.publish(String executionId, PlatformEventType eventType, Map<String,Object> data)`.

- [ ] **Step 1: Write failing happy-path, conflict, abort and timeout tests**

```java
@Test
void shouldPersistEventsAndCompleteExecutionOnce() {
    AgentExecution execution = service.createExecution("sess-1", "分析 C001 和 I001");
    fakeRuntime.awaitExecutionStarted();
    fakeRuntime.emit(new RuntimeEvent(RuntimeEventType.MESSAGE_DELTA,
            Map.of("delta", "正在分析")));
    fakeRuntime.complete(new RuntimeCompletion("客户概况\n...", 100, 50, 2));

    await().untilAsserted(() -> assertThat(executionMapper.selectById(execution.getId()).getStatus())
            .isEqualTo(ExecutionStatus.SUCCEEDED));
    assertThat(eventMapper.selectList(Wrappers.<ExecutionEvent>lambdaQuery()
                    .eq(ExecutionEvent::getExecutionId, execution.getId())
                    .orderByAsc(ExecutionEvent::getSequenceNo)))
            .extracting(ExecutionEvent::getEventType)
            .containsExactly(PlatformEventType.RUN_STARTED,
                    PlatformEventType.MESSAGE_DELTA,
                    PlatformEventType.RUN_COMPLETED);
}

@Test
void shouldRejectSecondRunningExecution() {
    service.createExecution("sess-1", "first");
    assertThatThrownBy(() -> service.createExecution("sess-1", "second"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessage("The session already has a running execution");
}
```

Use a controllable `ScheduledExecutorService` in tests to trigger timeout without waiting 120 seconds.

- [ ] **Step 2: Run tests and verify execution orchestration is missing**

Run: `mvn -q -Dtest=PlatformEventPublisherTest,ExecutionServiceTest test`

Expected: FAIL with missing publisher, registry and service.

- [ ] **Step 3: Implement serialized Event publishing**

```java
public synchronized ExecutionEvent publish(String executionId,
        PlatformEventType eventType, Map<String, Object> data) {
    long sequence = eventMapper.selectNextSequence(executionId);
    ExecutionEvent event = new ExecutionEvent(UUID.randomUUID().toString(),
            executionId, sequence, eventType,
            objectMapper.writeValueAsString(data), Instant.now(clock));
    eventMapper.insert(event);
    listeners.forEach(listener -> listener.onEvent(event));
    return event;
}
```

Convert JSON serialization failures to a stable internal exception without including `data` in the message.

- [ ] **Step 4: Implement Execution orchestration and terminal races**

`createExecution` must synchronously insert CREATED through `ExecutionMapper`, register it as active, then submit asynchronous work. Async work conditionally changes CREATED to RUNNING with a `LambdaUpdateWrapper`, publishes RUN_STARTED, schedules 120-second timeout, and calls Runtime. Runtime callbacks map events, aggregate message deltas, and compete for the terminal state through a second conditional wrapper.

```java
private void finish(String executionId, ExecutionStatus status,
        RuntimeCompletion completion, String errorCode, String safeMessage,
        PlatformEventType terminalEvent) {
    int updated = executionMapper.update(null,
            Wrappers.<AgentExecution>lambdaUpdate()
                    .eq(AgentExecution::getId, executionId)
                    .eq(AgentExecution::getStatus, ExecutionStatus.RUNNING)
                    .set(AgentExecution::getStatus, status)
                    .set(AgentExecution::getOutput,
                            completion == null ? null : completion.output())
                    .set(AgentExecution::getErrorCode, errorCode)
                    .set(AgentExecution::getErrorMessage, safeMessage)
                    .set(AgentExecution::getInputTokens,
                            completion == null ? 0 : completion.inputTokens())
                    .set(AgentExecution::getOutputTokens,
                            completion == null ? 0 : completion.outputTokens())
                    .set(AgentExecution::getToolCallCount,
                            completion == null ? 0 : completion.toolCallCount())
                    .set(AgentExecution::getFinishedAt, Instant.now(clock)));
    boolean won = updated == 1;
    if (won) {
        eventPublisher.publish(executionId, terminalEvent, Map.of());
        activeExecutionRegistry.removeByExecutionId(executionId);
        cancelTimeout(executionId);
    }
}
```

Abort sends Runtime abort before attempting `ABORTED`; timeout sends Runtime abort before attempting `TIMEOUT`. Only the winner publishes a terminal event.

- [ ] **Step 5: Update Session close to abort a running Execution first**

Inject a small `ExecutionAborter` interface rather than a circular `SessionService`/`ExecutionService` dependency:

```java
public interface ExecutionAborter {
    void abortRunningExecutionForSession(String sessionId);
}
```

`ExecutionService` implements it; `SessionService` invokes it before closing Runtime.

- [ ] **Step 6: Run Execution tests**

Run: `mvn -q -Dtest=PlatformEventPublisherTest,ExecutionServiceTest,SessionServiceTest test`

Expected: PASS, including deterministic abort/timeout terminal races.

- [ ] **Step 7: Commit**

```bash
git add src/main/java/com/iumyx/agentops/config/AsyncConfiguration.java \
  src/main/java/com/iumyx/agentops/execution \
  src/main/java/com/iumyx/agentops/session/SessionService.java \
  src/test/java/com/iumyx/agentops/execution \
  src/test/java/com/iumyx/agentops/session/SessionServiceTest.java
git commit -m "feat: orchestrate agent executions"
```

### Task 8: Internal Tool Gateway 与 Pi Tool Proxy Extension

**Files:**
- Create: `src/main/java/com/iumyx/agentops/tool/ToolInvocationRequest.java`
- Create: `src/main/java/com/iumyx/agentops/tool/ToolInvocationResponse.java`
- Create: `src/main/java/com/iumyx/agentops/tool/ToolGatewayController.java`
- Create: `pi/extensions/sales-tool-proxy.ts`
- Test: `src/test/java/com/iumyx/agentops/tool/ToolGatewayControllerTest.java`

**Interfaces:**
- Consumes: `ToolRegistry`, `SessionRuntimeContextRegistry`, `ActiveExecutionRegistry`, `ExecutionService`, fixed Tool allowlist.
- Produces: `POST /internal/tools/{toolName}/invoke`.
- Produces: Pi tools `get_customer_info` and `get_inquiry_info`.
- Produces: `ToolInvocationRequest(Map<String, Object> arguments)` and `ToolInvocationResponse(boolean found, Map<String, Object> data, String message)`.

- [ ] **Step 1: Write failing Gateway authorization and invocation tests**

```java
@Test
void shouldInvokeAllowedToolForActiveExecution() throws Exception {
    mockMvc.perform(post("/internal/tools/get_customer_info/invoke")
                    .header("X-Platform-Session-Id", "sess-1")
                    .header("X-Tool-Token", "valid-token")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"arguments\":{\"customerId\":\"C001\"}}"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.found").value(true))
            .andExpect(jsonPath("$.data.companyName").value("示例科技有限公司"));
}

@Test
void shouldRejectInvalidTokenWithoutEchoingIt() throws Exception {
    mockMvc.perform(post("/internal/tools/get_customer_info/invoke")
                    .header("X-Platform-Session-Id", "sess-1")
                    .header("X-Tool-Token", "secret-value")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"arguments\":{\"customerId\":\"C001\"}}"))
            .andExpect(status().isForbidden())
            .andExpect(content().string(not(containsString("secret-value"))));
}
```

- [ ] **Step 2: Run Gateway tests and verify missing controller fails**

Run: `mvn -q -Dtest=ToolGatewayControllerTest test`

Expected: FAIL because the Gateway types do not exist.

- [ ] **Step 3: Implement Gateway checks in fixed order**

Validate session, constant-time token equality, running Execution, allowlist, registered Tool, and arguments. Return `400` for invalid arguments, `403` for invalid token or Tool, `409` when no running Execution, and `200` with `found=false` for unknown business IDs.

```java
@PostMapping("/internal/tools/{toolName}/invoke")
public ToolInvocationResponse invoke(@PathVariable String toolName,
        @RequestHeader("X-Platform-Session-Id") String sessionId,
        @RequestHeader("X-Tool-Token") String token,
        @Valid @RequestBody ToolInvocationRequest request) {
    SessionRuntimeContext context = sessionContexts.getRequired(sessionId);
    verifyToken(context.toolToken(), token);
    activeExecutions.getRequiredBySessionId(sessionId);
    verifyAllowed(toolName);
    ToolResult result = toolRegistry.getRequired(toolName).execute(request.arguments());
    return ToolInvocationResponse.from(result);
}
```

If `AgentTool.execute` throws a technical exception, call `executionService.failFromToolGateway(executionId, "TOOL_EXECUTION_FAILED", "Tool execution failed")`, return HTTP 502, and never include the exception message or arguments. Add the exact method `void ExecutionService.failFromToolGateway(String executionId, String errorCode, String safeMessage)`; it aborts the Runtime and competes for terminal state `FAILED` through the same conditional MyBatis-Plus update as other failures.

- [ ] **Step 4: Implement the exact Pi Extension**

```typescript
import { Type } from "@earendil-works/pi-ai";
import { defineTool, type ExtensionAPI } from "@earendil-works/pi-coding-agent";

const gatewayUrl = requireEnvironment("AGENT_TOOL_GATEWAY_URL");
const sessionId = requireEnvironment("AGENT_PLATFORM_SESSION_ID");
const toolToken = requireEnvironment("AGENT_TOOL_TOKEN");

function requireEnvironment(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error("Missing required extension environment");
  return value;
}

async function invoke(name: string, argumentsValue: Record<string, string>, signal?: AbortSignal) {
  const response = await fetch(`${gatewayUrl}/internal/tools/${name}/invoke`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Platform-Session-Id": sessionId,
      "X-Tool-Token": toolToken,
    },
    body: JSON.stringify({ arguments: argumentsValue }),
    signal,
  });
  if (!response.ok) throw new Error(`Tool gateway request failed with status ${response.status}`);
  return await response.json();
}

export default function (pi: ExtensionAPI) {
  pi.registerTool(defineTool({
    name: "get_customer_info",
    label: "Get Customer Info",
    description: "根据客户 ID 查询客户信息",
    parameters: Type.Object({ customerId: Type.String({ minLength: 1 }) }),
    async execute(_id, params, signal) {
      const result = await invoke("get_customer_info", params, signal);
      return { content: [{ type: "text", text: JSON.stringify(result) }], details: result };
    },
  }));
  pi.registerTool(defineTool({
    name: "get_inquiry_info",
    label: "Get Inquiry Info",
    description: "根据询盘 ID 查询询盘信息",
    parameters: Type.Object({ inquiryId: Type.String({ minLength: 1 }) }),
    async execute(_id, params, signal) {
      const result = await invoke("get_inquiry_info", params, signal);
      return { content: [{ type: "text", text: JSON.stringify(result) }], details: result };
    },
  }));
}
```

Implement `requireEnvironment(name)` to throw only `Missing required extension environment` without printing the missing name's value.

- [ ] **Step 5: Run Gateway tests and Pi extension load check**

Run: `mvn -q -Dtest=ToolGatewayControllerTest test`

Run with dummy non-secret environment values and immediately send `get_state`:

```bash
AGENT_TOOL_GATEWAY_URL=http://127.0.0.1:8080 \
AGENT_PLATFORM_SESSION_ID=load-check \
AGENT_TOOL_TOKEN=load-check-token \
sh -c 'printf "%s\n" "{\"type\":\"get_state\"}" | timeout 10s pi --mode rpc --no-session --no-builtin-tools --no-extensions --extension pi/extensions/sales-tool-proxy.ts --no-skills --no-prompt-templates --no-context-files'
```

Expected: Java tests PASS; Pi outputs a successful `get_state` response and no `extension_error`.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/com/iumyx/agentops/tool/ToolInvocationRequest.java \
  src/main/java/com/iumyx/agentops/tool/ToolInvocationResponse.java \
  src/main/java/com/iumyx/agentops/tool/ToolGatewayController.java \
  src/test/java/com/iumyx/agentops/tool/ToolGatewayControllerTest.java \
  pi/extensions/sales-tool-proxy.ts
git commit -m "feat: proxy pi tool calls to java"
```

### Task 9: Session/Execution API、SSE 与 Trace

**Files:**
- Create: `src/main/java/com/iumyx/agentops/api/dto/CreateExecutionRequest.java`
- Create: `src/main/java/com/iumyx/agentops/api/dto/CreateExecutionResponse.java`
- Create: `src/main/java/com/iumyx/agentops/api/dto/SessionResponse.java`
- Create: `src/main/java/com/iumyx/agentops/api/dto/ExecutionResponse.java`
- Create: `src/main/java/com/iumyx/agentops/api/dto/ExecutionEventResponse.java`
- Create: `src/main/java/com/iumyx/agentops/api/SessionController.java`
- Create: `src/main/java/com/iumyx/agentops/api/ExecutionController.java`
- Create: `src/main/java/com/iumyx/agentops/api/ExecutionSseController.java`
- Create: `src/main/java/com/iumyx/agentops/api/ExecutionEventStream.java`
- Test: `src/test/java/com/iumyx/agentops/api/SessionControllerTest.java`
- Test: `src/test/java/com/iumyx/agentops/api/ExecutionControllerTest.java`
- Test: `src/test/java/com/iumyx/agentops/api/ExecutionEventStreamTest.java`

**Interfaces:**
- Produces: all public endpoints listed in the spec.
- Consumes: `SessionService`, `ExecutionService`, `ExecutionEventMapper`, `PlatformEventListener`.
- Produces: live listener registered before historical snapshot, with sequence-based deduplication.
- Produces: `CreateExecutionRequest(String message)`, `CreateExecutionResponse(String executionId, ExecutionStatus status, String eventsUrl)`, `SessionResponse(String id, String agentKey, String runtimeType, SessionStatus status, Instant createdAt, Instant closedAt)`, `ExecutionResponse` mirroring the persisted Execution fields, and `ExecutionEventResponse(String executionId, long sequence, PlatformEventType type, Instant timestamp, Map<String,Object> data)`.

- [ ] **Step 1: Write failing MVC contract tests**

```java
@Test
void shouldAcceptExecutionAndReturnEventsUrl() throws Exception {
    when(executionService.createExecution("sess-1", "分析 C001 和 I001"))
            .thenReturn(AgentExecution.created("exec-1", "sess-1",
                    "分析 C001 和 I001", Instant.EPOCH));

    mockMvc.perform(post("/api/sessions/sess-1/executions")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"message\":\"分析 C001 和 I001\"}"))
            .andExpect(status().isAccepted())
            .andExpect(jsonPath("$.executionId").value("exec-1"))
            .andExpect(jsonPath("$.eventsUrl").value("/api/executions/exec-1/events"));
}

@Test
void shouldRejectBlankMessage() throws Exception {
    mockMvc.perform(post("/api/sessions/sess-1/executions")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"message\":\" \"}"))
            .andExpect(status().isBadRequest());
}
```

- [ ] **Step 2: Run API tests and verify controllers are missing**

Run: `mvn -q -Dtest=SessionControllerTest,ExecutionControllerTest,ExecutionEventStreamTest test`

Expected: FAIL with missing API and DTO classes.

- [ ] **Step 3: Implement REST endpoints**

Required status codes:

```text
POST   /api/sessions                              201
DELETE /api/sessions/{sessionId}                  204
POST   /api/sessions/{sessionId}/executions       202
GET    /api/executions/{executionId}              200
GET    /api/executions/{executionId}/trace        200
POST   /api/executions/{executionId}/abort        202
GET    /api/executions/{executionId}/events       200 text/event-stream
```

`CreateExecutionRequest` must use `@NotBlank` and `@Size(max = 10000)`.

- [ ] **Step 4: Implement race-free SSE replay**

`ExecutionEventStream.subscribe(executionId)` must register the emitter first, load the ordered snapshot second, send snapshot events, and then send only queued live events whose sequence is greater than the highest sent sequence. Event IDs are decimal `sequence_no`; event names are `PlatformEventType.name()`.

On terminal Event, complete the emitter after sending it. On client timeout or disconnect, remove the emitter without changing Execution state.

- [ ] **Step 5: Run API/SSE tests**

Run: `mvn -q -Dtest=SessionControllerTest,ExecutionControllerTest,ExecutionEventStreamTest test`

Expected: PASS, including a test that injects a live event between subscription and snapshot completion and receives each sequence exactly once.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/com/iumyx/agentops/api \
  src/test/java/com/iumyx/agentops/api
git commit -m "feat: expose session execution and sse apis"
```

### Task 10: 稳定错误、启动恢复与敏感日志验证

**Files:**
- Create: `src/main/java/com/iumyx/agentops/common/ErrorCode.java`
- Create: `src/main/java/com/iumyx/agentops/common/ApiError.java`
- Create: `src/main/java/com/iumyx/agentops/common/GlobalExceptionHandler.java`
- Create: `src/main/java/com/iumyx/agentops/recovery/StartupRecovery.java`
- Modify: `src/main/java/com/iumyx/agentops/execution/ExecutionService.java`
- Modify: `src/main/java/com/iumyx/agentops/api/SessionController.java`
- Modify: `src/main/java/com/iumyx/agentops/api/ExecutionController.java`
- Modify: `src/main/java/com/iumyx/agentops/tool/ToolGatewayController.java`
- Test: `src/test/java/com/iumyx/agentops/common/GlobalExceptionHandlerTest.java`
- Test: `src/test/java/com/iumyx/agentops/recovery/StartupRecoveryTest.java`
- Test: `src/test/java/com/iumyx/agentops/common/SensitiveLoggingTest.java`

**Interfaces:**
- Produces: stable `ApiError(code, message, timestamp)` without stack trace.
- Produces: startup cleanup of ACTIVE Sessions and CREATED/RUNNING Executions.
- Consumes: `SessionMapper` and `ExecutionMapper` from Task 2.

- [ ] **Step 1: Write failing error mapping and recovery tests**

```java
@Test
void shouldMapConcurrentExecutionToConflictWithoutStackTrace() throws Exception {
    when(executionService.createExecution(anyString(), anyString()))
            .thenThrow(new IllegalStateException("The session already has a running execution"));
    mockMvc.perform(post("/api/sessions/sess-1/executions")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"message\":\"hello\"}"))
            .andExpect(status().isConflict())
            .andExpect(jsonPath("$.code").value("EXECUTION_ALREADY_RUNNING"))
            .andExpect(jsonPath("$.stackTrace").doesNotExist());
}

@Test
void shouldCloseStaleSessionsAndFailNonTerminalExecutions() {
    recovery.run(mock(ApplicationArguments.class));
    verify(sessionMapper).update(isNull(), any(LambdaUpdateWrapper.class));
    verify(executionMapper).update(isNull(), any(LambdaUpdateWrapper.class));
}
```

- [ ] **Step 2: Run tests and verify error/recovery types are missing**

Run: `mvn -q -Dtest=GlobalExceptionHandlerTest,StartupRecoveryTest,SensitiveLoggingTest test`

Expected: FAIL with missing common and recovery classes.

- [ ] **Step 3: Implement stable error codes and mappings**

Define exactly:

```text
SESSION_NOT_FOUND
SESSION_NOT_ACTIVE
ACTIVE_SESSION_ALREADY_EXISTS
EXECUTION_NOT_FOUND
EXECUTION_ALREADY_RUNNING
EXECUTION_NOT_RUNNING
TOOL_NOT_ALLOWED
TOOL_ARGUMENT_INVALID
TOOL_GATEWAY_FORBIDDEN
TOOL_EXECUTION_FAILED
TOOL_PROXY_FAILED
RUNTIME_START_FAILED
PI_PROTOCOL_ERROR
PI_PROCESS_EXITED
APPLICATION_RESTARTED
INTERNAL_ERROR
```

Map validation to 400, missing resources to 404, conflicts to 409, invalid Internal Tool access to 403, Runtime startup failure to 503, and unknown exceptions to 500 with message `Internal server error`.

- [ ] **Step 4: Implement startup recovery**

```java
@Component
public class StartupRecovery implements ApplicationRunner {
    @Override
    @Transactional
    public void run(ApplicationArguments arguments) {
        Instant now = Instant.now(clock);
        int closedSessions = sessionMapper.update(null,
                Wrappers.<AgentSession>lambdaUpdate()
                        .eq(AgentSession::getStatus, SessionStatus.ACTIVE)
                        .set(AgentSession::getStatus, SessionStatus.CLOSED)
                        .set(AgentSession::getClosedAt, now));
        int failedExecutions = executionMapper.update(null,
                Wrappers.<AgentExecution>lambdaUpdate()
                        .in(AgentExecution::getStatus,
                                ExecutionStatus.CREATED, ExecutionStatus.RUNNING)
                        .set(AgentExecution::getStatus, ExecutionStatus.FAILED)
                        .set(AgentExecution::getErrorCode, "APPLICATION_RESTARTED")
                        .set(AgentExecution::getErrorMessage,
                                "Execution interrupted by application restart")
                        .set(AgentExecution::getFinishedAt, now));
        log.info("Recovered runtime state: closedSessions={}, failedExecutions={}",
                closedSessions, failedExecutions);
    }
}
```

Log only counts of affected rows. Do not log records, prompts, SQL parameters, environment variables, command lines, Tool arguments or Tool results.

- [ ] **Step 5: Add sensitive logging assertions**

Capture application logs with a test appender, invoke a request containing markers `db-password-marker`, `tool-token-marker`, and `customer-data-marker`, then assert the rendered log text contains none of them. The test must also assert the HTTP error body omits them.

- [ ] **Step 6: Run common/recovery tests and full unit suite**

Run: `mvn -q -Dtest=GlobalExceptionHandlerTest,StartupRecoveryTest,SensitiveLoggingTest test && mvn -q test`

Expected: PASS. Tests that require `mysql-test` remain outside the default suite.

- [ ] **Step 7: Commit**

```bash
git add src/main/java/com/iumyx/agentops/common \
  src/main/java/com/iumyx/agentops/recovery \
  src/main/java/com/iumyx/agentops/execution/ExecutionService.java \
  src/main/java/com/iumyx/agentops/api \
  src/main/java/com/iumyx/agentops/tool/ToolGatewayController.java \
  src/test/java/com/iumyx/agentops/common \
  src/test/java/com/iumyx/agentops/recovery
git commit -m "feat: recover runtime state and sanitize errors"
```

### Task 11: 真实 Pi 冒烟、运行手册与最终验收

**Files:**
- Create: `src/test/java/com/iumyx/agentops/runtime/pi/PiRuntimeSmokeTest.java`
- Create: `scripts/sales-inquiry-mvp-smoke.sh`
- Create: `doc/project/sales-inquiry-agent-mvp-runbook.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: running MySQL, authenticated Pi `0.84.2`, fixed model and all public APIs.
- Produces: repeatable startup instructions and an opt-in real-model smoke test.

- [ ] **Step 1: Write the opt-in real Pi smoke test**

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
@EnabledIfEnvironmentVariable(named = "RUN_PI_SMOKE", matches = "true")
class PiRuntimeSmokeTest {

    @Autowired private SessionService sessionService;
    @Autowired private ExecutionService executionService;

    @Test
    void shouldCallBothSalesToolsAndKeepMultiTurnContext() {
        AgentSession session = sessionService.createSession();
        AgentExecution first = executionService.createExecution(session.getId(),
                "请分析客户 C001 的询盘 I001，并给出销售建议");
        awaitTerminal(first.getId());
        AgentExecution completed = executionService.getRequired(first.getId());
        assertThat(completed.getStatus()).isEqualTo(ExecutionStatus.SUCCEEDED);
        assertThat(completed.getOutput()).contains("客户概况", "询盘摘要", "销售建议", "下一步行动");
        assertThat(completed.getToolCallCount()).isGreaterThanOrEqualTo(2);

        AgentExecution second = executionService.createExecution(session.getId(),
                "请基于上一轮结果给出一句跟进开场白");
        awaitTerminal(second.getId());
        assertThat(executionService.getRequired(second.getId()).getStatus())
                .isEqualTo(ExecutionStatus.SUCCEEDED);
    }

    private void awaitTerminal(String executionId) {
        await().atMost(Duration.ofSeconds(180)).untilAsserted(() ->
                assertThat(executionService.getRequired(executionId).getStatus().isTerminal())
                        .isTrue());
    }
}
```

Use a 180-second Awaitility limit only in this opt-in smoke test. Do not assert exact model prose.

- [ ] **Step 2: Run the default suite first**

Run: `mvn -q test`

Expected: PASS; `PiRuntimeSmokeTest` is skipped because `RUN_PI_SMOKE` is absent.

- [ ] **Step 3: Write the runbook and curl smoke script**

The runbook must document:

1. Creating `java_agent_ops_mvp` and `java_agent_ops_mvp_test`.
2. Exporting the six `MVP_DB_*` and `MVP_TEST_DB_*` variables without showing values.
3. Confirming `pi --version` is `0.84.2` and the default auth can access `zai-coding-cn/glm-5.3`.
4. Starting with `mvn spring-boot:run`.
5. Opening `/swagger-ui.html`.
6. Creating one Session, submitting `C001/I001`, subscribing to SSE, checking Trace, sending a second turn, and closing Session.
7. Running `ExecutionServiceTest` to verify deterministic abort and timeout behavior with Fake Runtime.

The shell script must use `set -euo pipefail`, parse IDs with `jq`, never echo environment variables, and fail unless the final answer contains all four required headings.

- [ ] **Step 4: Run MySQL integration tests**

Run: `RUN_MYSQL_INTEGRATION=true mvn -q -Dspring.profiles.active=mysql-test -Dtest=PersistenceIntegrationTest test`

Expected: PASS against `java_agent_ops_mvp_test`.

- [ ] **Step 5: Run the real Pi smoke test**

Run: `RUN_PI_SMOKE=true mvn -q -Dtest=PiRuntimeSmokeTest test`

Expected: PASS; Trace contains both Tool calls and two SUCCEEDED Executions in one Session.

- [ ] **Step 6: Run final verification**

```bash
mvn -q test
mvn -q -DskipTests package
git diff --check
rg -n "Root@|password: [^$]|AGENT_TOOL_TOKEN=.*[^}]" \
  src pi scripts doc/project/sales-inquiry-agent-mvp-runbook.md
```

Expected: tests and package PASS; `git diff --check` is silent; secret scan has no matches.

- [ ] **Step 7: Commit**

```bash
git add src/test/java/com/iumyx/agentops/runtime/pi/PiRuntimeSmokeTest.java \
  scripts/sales-inquiry-mvp-smoke.sh \
  doc/project/sales-inquiry-agent-mvp-runbook.md README.md
git commit -m "docs: add sales inquiry agent runbook"
```

## Final Acceptance Checklist

- [ ] `mvn -q test` passes without MySQL or Pi credentials.
- [ ] MySQL integration test passes against `java_agent_ops_mvp_test`.
- [ ] Real Pi smoke test calls only `get_customer_info` and `get_inquiry_info`.
- [ ] First Execution output contains all four required headings.
- [ ] Second Execution reuses Pi context and creates a separate platform Execution.
- [ ] SSE replay/live merge contains each sequence exactly once.
- [ ] SUCCEEDED、FAILED、ABORTED、TIMEOUT terminal states are covered.
- [ ] A second concurrent Execution returns HTTP 409.
- [ ] Restart recovery closes stale Sessions and fails nonterminal Executions.
- [ ] Logs, SSE and error bodies contain no database password, Pi auth, Tool token or raw stack trace.
- [ ] Swagger UI can demonstrate all public MVP endpoints without a frontend.
