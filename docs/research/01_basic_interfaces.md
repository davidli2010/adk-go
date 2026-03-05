# ADK Go 核心接口定义

> 任务 1.1：理解项目整体结构
> 阅读文件：agent/agent.go, agent/context.go, session/session.go, model/llm.go, tool/tool.go

---

## 1. Agent 接口 (`agent/agent.go`)

Agent 是所有代理实现的基础接口，定义在 `agent/agent.go:43`。

```go
type Agent interface {
    Name() string
    Description() string
    Run(InvocationContext) iter.Seq2[*session.Event, error]
    SubAgents() []Agent
    internal() *agent
}
```

### 核心方法

| 方法 | 说明 |
|------|------|
| `Name()` | 返回代理名称，必须唯一 |
| `Description()` | 返回代理能力描述，LLM 用于决定是否委派任务 |
| `Run()` | 执行代理逻辑，返回事件序列和错误 |
| `SubAgents()` | 返回子代理列表，支持任务委派 |

### Agent 配置 (`agent/agent.go:74`)

```go
type Config struct {
    Name        string
    Description string
    SubAgents   []Agent
    
    // 回调函数
    BeforeAgentCallbacks []BeforeAgentCallback
    Run                  func(InvocationContext) iter.Seq2[*session.Event, error]
    AfterAgentCallbacks  []AfterAgentCallback
}
```

---

## 2. InvocationContext 接口 (`agent/context.go`)

InvocationContext 表示代理调用的上下文，贯穿整个请求的生命周期。

```go
type InvocationContext interface {
    context.Context
    
    Agent() Agent
    Artifacts() Artifacts
    Memory() Memory
    Session() session.Session
    InvocationID() string
    Branch() string
    UserContent() *genai.Content
    RunConfig() *RunConfig
    EndInvocation()
    Ended() bool
    WithContext(ctx context.Context) InvocationContext
}
```

### 核心方法

| 方法 | 说明 |
|------|------|
| `Agent()` | 当前执行的代理 |
| `Session()` | 会话状态 |
| `Artifacts()` | 资源文件服务 |
| `Memory()` | 长期记忆服务 |
| `InvocationID()` | 调用唯一标识 |
| `Branch()` | 调用分支路径，格式：`agent1.agent2.agent3` |
| `UserContent()` | 用户输入内容 |
| `RunConfig()` | 运行时配置 |
| `EndInvocation()` | 结束当前调用 |
| `Ended()` | 检查是否已结束 |

### 调用层级结构 (`agent/context.go:55`)

```
┌─────────────────────── invocation ──────────────────────────┐
┌──────────── llm_agent_call_1 ────────────┐ ┌─ agent_call_2 ─┐
┌──── step_1 ────────┐ ┌───── step_2 ──────┐
[call_llm] [call_tool] [call_llm] [transfer]
```

- **Invocation**: 从用户消息开始到最终响应结束
- **Agent Call**: 由 agent.Run() 处理
- **Step**: 单次 LLM 调用 + 可能的工具调用

---

## 3. Session 接口 (`session/session.go`)

Session 表示用户与代理之间的对话系列。

```go
type Session interface {
    ID() string
    AppName() string
    UserID() string
    State() State
    Events() Events
    LastUpdateTime() time.Time
}
```

### State 接口 (`session/session.go:51`)

```go
type State interface {
    Get(string) (any, error)
    Set(string, any) error
    All() iter.Seq2[string, any]
}
```

### 状态键前缀 (`session/session.go:162`)

| 前缀 | 说明 | 作用域 |
|------|------|--------|
| `app:` | 应用级别状态 | 所有用户和会话共享 |
| `user:` | 用户级别状态 | 同一用户的会话共享 |
| `temp:` | 临时状态 | 当前调用内有效 |

---

## 4. Event 结构 (`session/session.go:92`)

Event 记录对话中的每一次交互。

```go
type Event struct {
    model.LLMResponse
    
    ID           string
    Timestamp    time.Time
    InvocationID string
    Branch       string
    Author       string
    Actions      EventActions
    LongRunningToolIDs []string
}
```

### EventActions (`session/session.go:142`)

```go
type EventActions struct {
    StateDelta              map[string]any
    ArtifactDelta           map[string]int64
    RequestedToolConfirmations map[string]toolconfirmation.ToolConfirmation
    SkipSummarization       bool
    TransferToAgent         string
    Escalate                bool
}
```

### IsFinalResponse (`session/session.go:120`)

判断是否为最终响应。满足以下条件时返回 true：
- `SkipSummarization` 为 true
- 有 `LongRunningToolIDs`
- 没有 FunctionCall
- 没有 FunctionResponse
- 不是 Partial
- 没有尾随代码执行结果

---

## 5. LLM 接口 (`model/llm.go`)

LLM 提供对底层大语言模型的访问。

```go
type LLM interface {
    Name() string
    GenerateContent(ctx context.Context, req *LLMRequest, stream bool) iter.Seq2[*LLMResponse, error]
}
```

### LLMRequest (`model/llm.go:31`)

```go
type LLMRequest struct {
    Model    string
    Contents []*genai.Content
    Config   *genai.GenerateContentConfig
    Tools    map[string]any
}
```

### LLMResponse (`model/llm.go:41`)

```go
type LLMResponse struct {
    Content            *genai.Content
    CitationMetadata   *genai.CitationMetadata
    GroundingMetadata  *genai.GroundingMetadata
    UsageMetadata      *genai.GenerateContentResponseUsageMetadata
    CustomMetadata     map[string]any
    LogprobsResult     *genai.LogprobsResult
    Partial            bool      // 流式模式下的部分内容
    TurnComplete       bool      // 流式模式下的响应完成标志
    Interrupted        bool      // 中断标志
    ErrorCode          string
    ErrorMessage       string
    FinishReason       genai.FinishReason
    AvgLogprobs        float64
}
```

---

## 6. Tool 接口 (`tool/tool.go`)

Tool 定义可调用工具的接口。

```go
type Tool interface {
    Name() string
    Description() string
    IsLongRunning() bool
}
```

### Tool Context (`tool/tool.go:45`)

```go
type Context interface {
    agent.CallbackContext
    FunctionCallID() string
    Actions() *session.EventActions
    SearchMemory(context.Context, string) (*memory.SearchResponse, error)
    ToolConfirmation() *toolconfirmation.ToolConfirmation
    RequestConfirmation(hint string, payload any) error
}
```

### Toolset 接口 (`tool/tool.go:97`)

```go
type Toolset interface {
    Name() string
    Tools(ctx agent.ReadonlyContext) ([]Tool, error)
}
```

### runnableTool 接口 (`tool/tool.go:225`)

```go
type runnableTool interface {
    Tool
    Declaration() *genai.FunctionDeclaration
    Run(ctx Context, args any) (result map[string]any, err error)
}
```

---

## 7. 回调接口

### CallbackContext (`agent/context.go:122`)

```go
type CallbackContext interface {
    ReadonlyContext
    Artifacts() Artifacts
    State() session.State
}
```

### BeforeAgentCallback (`agent/agent.go:123`)

```go
type BeforeAgentCallback func(CallbackContext) (*genai.Content, error)
```

### AfterAgentCallback (`agent/agent.go:129`)

```go
type AfterAgentCallback func(CallbackContext) (*genai.Content, error)
```

---

## 接口关系图

```
┌─────────────────────────────────────────────────────────────┐
│                      InvocationContext                       │
│  (agent.Context)                                            │
├─────────────────────────────────────────────────────────────┤
│  Agent()  Session()  Artifacts()  Memory()  RunConfig()    │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
       ┌────────┐       ┌──────────┐      ┌──────────┐
       │ Agent  │       │ Session  │      │   LLM    │
       └────────┘       └──────────┘      └──────────┘
                              │                 │
                              ▼                 ▼
                        ┌──────────┐      ┌──────────┐
                        │  State   │      │ Tool     │
                        └──────────┘      └──────────┘
                              │
                              ▼
                        ┌──────────┐
                        │  Event   │
                        └──────────┘
```

---

## 关键设计模式

### 1. 泛型迭代器

大量使用 Go 1.22+ 的 `iter.Seq` 和 `iter.Seq2`：
- `iter.Seq2[*session.Event, error]` - 事件序列
- `iter.Seq2[string, any]` - 键值对迭代

### 2. 接口驱动

所有核心组件通过接口定义，确保松耦合和可测试性。

### 3. 回调机制

通过 BeforeAgentCallback、AfterAgentCallback 等扩展代理行为。
