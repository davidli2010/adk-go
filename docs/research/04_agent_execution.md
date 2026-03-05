# ADK Go Agent 执行入口分析

## 概述

本文档深入分析 `agent/agent.go` 中的 Agent 执行机制，包括 Agent 接口定义、Run() 方法实现、前后置回调执行流程。

---

## Agent 接口定义

**位置**: `agent/agent.go:40-50`

```go
type Agent interface {
    Name() string
    Description() string
    Run(InvocationContext) iter.Seq2[*session.Event, error]
    SubAgents() []Agent
    
    internal() *agent  // 内部方法，获取内部结构
}
```

**接口方法说明**:

| 方法 | 返回值 | 用途 |
|------|--------|------|
| `Name()` | string | 代理唯一标识名 |
| `Description()` | string | 能力描述，LLM 用于判断是否委托 |
| `Run()` | iter.Seq2 | 执行入口，产出事件流 |
| `SubAgents()` | []Agent | 子代理列表 |
| `internal()` | *agent | 获取内部实现结构 |

**设计特点**:
- 所有 Agent 实现必须返回迭代器流
- 支持树状层级结构（SubAgents）
- 内部方法 `internal()` 暴露实现细节给框架使用

---

## agent 结构实现

**位置**: `agent/agent.go:137-146`

```go
type agent struct {
    agentinternal.State
    
    name, description string
    subAgents         []Agent
    
    beforeAgentCallbacks []BeforeAgentCallback
    run                  func(InvocationContext) iter.Seq2[*session.Event, error]
    afterAgentCallbacks  []AfterAgentCallback
}
```

**嵌入结构**: `agentinternal.State` - 包含 Agent 类型等内部状态

**核心字段**:
- `name`, `description`: 基本属性
- `subAgents`: 子代理列表
- `beforeAgentCallbacks`: 前置回调列表
- `run`: 实际执行函数（可自定义）
- `afterAgentCallbacks`: 后置回调列表

---

## Agent.Run() 核心方法

**位置**: `agent/agent.go:160-213`

### 方法签名

```go
func (a *agent) Run(ctx InvocationContext) iter.Seq2[*session.Event, error]
```

### 完整执行流程

```go
return func(yield func(*session.Event, error) bool) {
    // 1. 启动追踪 Span
    spanCtx, span := telemetry.StartInvokeAgentSpan(...)
    yield, endSpan := telemetry.WrapYield(...)
    defer endSpan()
    
    // 2. 构建 invocationContext
    ctx := &invocationContext{...}
    
    // 3. 执行前置回调
    event, err := runBeforeAgentCallbacks(ctx)
    if event != nil || err != nil {
        if !yield(event, err) { return }
    }
    
    // 4. 检查是否提前结束
    if ctx.Ended() { return }
    
    // 5. 执行实际 run 函数
    for event, err := range a.run(ctx) {
        if event != nil && event.Author == "" {
            event.Author = getAuthorForEvent(ctx, event)
        }
        if !yield(event, err) { return }
    }
    
    // 6. 再次检查是否结束
    if ctx.Ended() { return }
    
    // 7. 执行后置回调
    event, err = runAfterAgentCallbacks(ctx)
    if event != nil || err != nil {
        yield(event, err)
    }
}
```

### 流程图

```
Agent.Run()
    │
    ├─ 1. 启动 Telemetry 追踪
    │   └─ StartInvokeAgentSpan()
    │
    ├─ 2. 构建 invocationContext
    │   └─ 包装上下文，注入追踪
    │
    ├─ 3. runBeforeAgentCallbacks()
    │   ├─ 插件 BeforeAgentCallback
    │   └─ 用户定义 BeforeAgentCallbacks
    │   │
    │   └─ 如有返回 → yield 事件 → 检查 Ended() → 返回
    │
    ├─ 4. 检查提前结束
    │   └─ ctx.Ended() ? 是 → 返回
    │
    ├─ 5. 执行 a.run(ctx)
    │   ├─ 遍历事件流
    │   ├─ 自动设置 Author（如为空）
    │   └─ yield 每个事件
    │
    ├─ 6. 再次检查结束
    │   └─ ctx.Ended() ? 是 → 返回
    │
    └─ 7. runAfterAgentCallbacks()
        ├─ 插件 AfterAgentCallback
        └─ 用户定义 AfterAgentCallbacks
        │
        └─ 如有返回 → yield 事件
```

---

## 关键步骤详解

### 步骤 1：启动 Telemetry 追踪

**位置**: `agent/agent.go:162-169`

```go
spanCtx, span := telemetry.StartInvokeAgentSpan(ctx, a, ctx.Session().ID(), ctx.InvocationID())
yield, endSpan := telemetry.WrapYield(span, yield, func(span trace.Span, event *session.Event, err error) {
    telemetry.TraceAgentResult(span, telemetry.TraceAgentResultParams{
        ResponseEvent: event,
        Error:         err,
    })
})
defer endSpan()
```

**功能**:
- 创建 OpenTelemetry Span，追踪 Agent 调用
- `WrapYield` 包装 yield 函数，自动记录事件结果
- `defer endSpan()` 确保 Span 正确结束

**追踪内容**:
- Agent 名称、SessionID、InvocationID
- 响应事件内容
- 错误信息

---

### 步骤 2：构建 invocationContext

**位置**: `agent/agent.go:171-181`

```go
ctx := &invocationContext{
    Context:   ctx.WithContext(spanCtx),
    agent:     a,
    artifacts: ctx.Artifacts(),
    memory:    ctx.Memory(),
    session:   ctx.Session(),
    
    invocationID:  ctx.InvocationID(),
    branch:        ctx.Branch(),
    userContent:   ctx.UserContent(),
    runConfig:     ctx.RunConfig(),
    endInvocation: ctx.Ended(),
}
```

**目的**: 创建新的上下文包装，注入追踪上下文，保存当前状态快照。

---

### 步骤 3：执行前置回调

**位置**: `agent/agent.go:182-189`

```go
event, err := runBeforeAgentCallbacks(ctx)
if event != nil || err != nil {
    if !yield(event, err) {
        return
    }
}
```

**详细实现**: 见 `runBeforeAgentCallbacks()` 函数

**提前退出条件**:
- 回调返回非 nil 内容
- 回调返回错误
- 插件回调提前退出

---

### 步骤 4：检查提前结束

**位置**: `agent/agent.go:191-193`

```go
if ctx.Ended() {
    return
}
```

**触发条件**: 前置回调中调用 `ctx.EndInvocation()`

---

### 步骤 5：执行实际 run 函数

**位置**: `agent/agent.go:195-202`

```go
for event, err := range a.run(ctx) {
    if event != nil && event.Author == "" {
        event.Author = getAuthorForEvent(ctx, event)
    }
    if !yield(event, err) {
        return
    }
}
```

**关键点**:
- 遍历用户定义的 `run` 函数产出的事件流
- 自动设置 `Author` 字段（如为空）
- 支持消费者提前终止（`yield` 返回 false）

---

### getAuthorForEvent() 函数

**位置**: `agent/agent.go:219-225`

```go
func getAuthorForEvent(ctx InvocationContext, event *session.Event) string {
    if event.LLMResponse.Content != nil && event.LLMResponse.Content.Role == genai.RoleUser {
        return genai.RoleUser
    }
    return ctx.Agent().Name()
}
```

**逻辑**:
- 用户角色内容 → 返回 `"user"`
- 其他情况 → 返回当前 Agent 名称

---

### 步骤 6-7：后置回调

**位置**: `agent/agent.go:204-212`

```go
if ctx.Ended() {
    return
}

event, err = runAfterAgentCallbacks(ctx)
if event != nil || err != nil {
    yield(event, err)
}
```

**注意**: 后置回调只在未提前结束时执行

---

## runBeforeAgentCallbacks() 详解

**位置**: `agent/agent.go:229-286`

### 函数签名

```go
func runBeforeAgentCallbacks(ctx InvocationContext) (*session.Event, error)
```

### 执行流程

```go
agent := ctx.Agent()
pluginManager := pluginManagerFromContext(ctx)

callbackCtx := &callbackContext{
    Context:           ctx,
    invocationContext: ctx,
    actions:           &session.EventActions{StateDelta: make(map[string]any)},
}

// 1. 执行插件回调
if pluginManager != nil {
    content, err := pluginManager.RunBeforeAgentCallback(callbackCtx)
    if err != nil {
        return nil, fmt.Errorf("failed to run plugin before agent callback: %w", err)
    }
    if content != nil {
        // 创建事件并提前返回
        event := session.NewEvent(ctx.InvocationID())
        event.LLMResponse = model.LLMResponse{Content: content}
        event.Author = agent.Name()
        event.Branch = ctx.Branch()
        event.Actions = *callbackCtx.actions
        ctx.EndInvocation()
        return event, nil
    }
}

// 2. 执行用户定义回调
for _, callback := range ctx.Agent().internal().beforeAgentCallbacks {
    content, err := callback(callbackCtx)
    if err != nil {
        return nil, fmt.Errorf("failed to run before agent callback: %w", err)
    }
    if content == nil {
        continue
    }
    
    // 创建事件并提前返回
    event := session.NewEvent(ctx.InvocationID())
    event.LLMResponse = model.LLMResponse{Content: content}
    event.Author = agent.Name()
    event.Branch = ctx.Branch()
    event.Actions = *callbackCtx.actions
    ctx.EndInvocation()
    return event, nil
}

// 3. 检查状态变更
if len(callbackCtx.actions.StateDelta) > 0 {
    event := session.NewEvent(ctx.InvocationID())
    event.Author = agent.Name()
    event.Branch = ctx.Branch()
    event.Actions = *callbackCtx.actions
    return event, nil
}

return nil, nil
```

### 执行顺序

1. **插件回调优先**: `pluginManager.RunBeforeAgentCallback()`
2. **用户回调依次**: 按顺序执行 `beforeAgentCallbacks`
3. **状态变更检查**: 即使无内容返回，有状态变更也创建事件

### 提前退出机制

```go
if content != nil {
    ctx.EndInvocation()  // 标记调用结束
    return event, nil    // 跳过后续回调和 Agent 执行
}
```

---

## runAfterAgentCallbacks() 详解

**位置**: `agent/agent.go:289-348`

### 函数签名

```go
func runAfterAgentCallbacks(ctx InvocationContext) (*session.Event, error)
```

### 执行流程

```go
agent := ctx.Agent()
pluginManager := pluginManagerFromContext(ctx)

callbackCtx := &callbackContext{
    Context:           ctx,
    invocationContext: ctx,
    actions:           &session.EventActions{StateDelta: make(map[string]any)},
}

// 1. 执行插件回调
if pluginManager != nil {
    content, err := pluginManager.RunAfterAgentCallback(callbackCtx)
    if err != nil {
        return nil, fmt.Errorf("failed to run plugin after agent callback: %w", err)
    }
    if content != nil {
        event := session.NewEvent(ctx.InvocationID())
        event.LLMResponse = model.LLMResponse{Content: content}
        event.Author = agent.Name()
        event.Branch = ctx.Branch()
        event.Actions = *callbackCtx.actions
        return event, nil
    }
}

// 2. 执行用户定义回调
for _, callback := range agent.internal().afterAgentCallbacks {
    newContent, err := callback(callbackCtx)
    if err != nil {
        return nil, fmt.Errorf("failed to run after agent callback: %w", err)
    }
    if newContent == nil {
        continue
    }
    
    event := session.NewEvent(ctx.InvocationID())
    event.LLMResponse = model.LLMResponse{Content: newContent}
    event.Author = agent.Name()
    event.Branch = ctx.Branch()
    event.Actions = *callbackCtx.actions
    return event, nil
}

// 3. 检查状态变更
if len(callbackCtx.actions.StateDelta) > 0 {
    event := session.NewEvent(ctx.InvocationID())
    event.Author = agent.Name()
    event.Branch = ctx.Branch()
    event.Actions = *callbackCtx.actions
    return event, nil
}
return nil, nil
```

### 与前置回调的区别

| 特性 | BeforeAgentCallback | AfterAgentCallback |
|------|---------------------|---------------------|
| 执行时机 | Agent 执行前 | Agent 执行后 |
| 提前退出 | 调用 `ctx.EndInvocation()` | 不调用 |
| 跳过条件 | 返回非 nil 内容/错误 | 返回非 nil 内容/错误 |
| 状态变更 | 创建事件 | 创建事件 |

---

## callbackContext 结构

**位置**: `agent/agent.go:352-402`

```go
type callbackContext struct {
    context.Context
    invocationContext InvocationContext
    actions           *session.EventActions
}
```

### 实现的方法

```go
func (c *callbackContext) AgentName() string
func (c *callbackContext) ReadonlyState() session.ReadonlyState
func (c *callbackContext) State() session.State
func (c *callbackContext) Artifacts() Artifacts
func (c *callbackContext) InvocationID() string
func (c *callbackContext) UserContent() *genai.Content
func (c *callbackContext) AppName() string
func (c *callbackContext) Branch() string
func (c *callbackContext) SessionID() string
func (c *callbackContext) UserID() string
```

### callbackContextState 嵌套结构

**位置**: `agent/agent.go:404-428`

```go
type callbackContextState struct {
    ctx *callbackContext
}

func (c *callbackContextState) Get(key string) (any, error) {
    // 优先从 StateDelta 读取
    if c.ctx.actions != nil && c.ctx.actions.StateDelta != nil {
        if val, ok := c.ctx.actions.StateDelta[key]; ok {
            return val, nil
        }
    }
    return c.ctx.invocationContext.Session().State().Get(key)
}

func (c *callbackContextState) Set(key string, val any) error {
    // 同时更新 StateDelta 和 Session State
    if c.ctx.actions != nil && c.ctx.actions.StateDelta != nil {
        c.ctx.actions.StateDelta[key] = val
    }
    return c.ctx.invocationContext.Session().State().Set(key, val)
}

func (c *callbackContextState) All() iter.Seq2[string, any] {
    return c.ctx.invocationContext.Session().State().All()
}
```

**设计意图**:
- 回调中的状态变更先写入 `StateDelta`
- 事件创建时携带状态变更记录
- 后续持久化到 Session

---

## invocationContext 结构

**位置**: `agent/agent.go:428-487`

```go
type invocationContext struct {
    context.Context
    
    agent     Agent
    artifacts Artifacts
    memory    Memory
    session   session.Session
    
    invocationID  string
    branch        string
    userContent   *genai.Content
    runConfig     *RunConfig
    endInvocation bool
}
```

### 实现的方法

```go
func (c *invocationContext) Agent() Agent
func (c *invocationContext) Artifacts() Artifacts
func (c *invocationContext) Memory() Memory
func (c *invocationContext) Session() session.Session
func (c *invocationContext) InvocationID() string
func (c *invocationContext) Branch() string
func (c *invocationContext) UserContent() *genai.Content
func (c *invocationContext) RunConfig() *RunConfig
func (c *invocationContext) EndInvocation()
func (c *invocationContext) Ended() bool
func (c *invocationContext) WithContext(ctx context.Context) InvocationContext
```

**核心功能**:
- 提供统一的上下文访问接口
- 支持 `EndInvocation()` 提前结束调用
- `WithContext()` 创建新副本（用于追踪上下文更新）

---

## Config 配置结构

**位置**: `agent/agent.go:74-105`

```go
type Config struct {
    Name              string
    Description       string
    SubAgents         []Agent
    BeforeAgentCallbacks []BeforeAgentCallback
    Run               func(InvocationContext) iter.Seq2[*session.Event, error]
    AfterAgentCallbacks []AfterAgentCallback
}
```

**字段说明**:

| 字段 | 必填 | 用途 |
|------|------|------|
| `Name` | ✓ | 代理唯一名称 |
| `Description` | ✓ | 能力描述 |
| `SubAgents` | ✗ | 子代理列表 |
| `BeforeAgentCallbacks` | ✗ | 前置回调列表 |
| `Run` | ✓ | 执行函数 |
| `AfterAgentCallbacks` | ✗ | 后置回调列表 |

---

## New() 构造函数

**位置**: `agent/agent.go:52-72`

```go
func New(cfg Config) (Agent, error) {
    // 1. 检查子代理唯一性
    subAgentSet := make(map[Agent]bool)
    for _, subAgent := range cfg.SubAgents {
        if _, ok := subAgentSet[subAgent]; ok {
            return nil, fmt.Errorf("error creating agent: subagent %q appears multiple times in subAgents", subAgent.Name())
        }
        subAgentSet[subAgent] = true
    }
    
    // 2. 创建 agent 实例
    return &agent{
        name:                 cfg.Name,
        description:          cfg.Description,
        subAgents:            cfg.SubAgents,
        beforeAgentCallbacks: cfg.BeforeAgentCallbacks,
        run:                  cfg.Run,
        afterAgentCallbacks:  cfg.AfterAgentCallbacks,
        State: agentinternal.State{
            AgentType: agentinternal.TypeCustomAgent,
        },
    }, nil
}
```

**验证逻辑**:
- 检查子代理不重复
- 确保配置有效性

---

## 回调类型定义

### BeforeAgentCallback

**位置**: `agent/agent.go:129`

```go
type BeforeAgentCallback func(CallbackContext) (*genai.Content, error)
```

**用途**: Agent 执行前钩子，可：
- 修改状态
- 提前返回内容（跳过 Agent 执行）
- 返回错误

### AfterAgentCallback

**位置**: `agent/agent.go:136`

```go
type AfterAgentCallback func(CallbackContext) (*genai.Content, error)
```

**用途**: Agent 执行后钩子，可：
- 修改状态
- 返回补充内容
- 返回错误

---

## 插件管理器集成

### 获取插件管理器

**位置**: `agent/agent.go:489-497`

```go
func pluginManagerFromContext(ctx context.Context) pluginManager {
    a := ctx.Value(plugincontext.PluginManagerCtxKey)
    m, ok := a.(pluginManager)
    if !ok {
        return nil
    }
    return m
}
```

### pluginManager 接口

**位置**: `agent/agent.go:498-502`

```go
type pluginManager interface {
    RunBeforeAgentCallback(cctx CallbackContext) (*genai.Content, error)
    RunAfterAgentCallback(cctx CallbackContext) (*genai.Content, error)
}
```

**执行顺序**: 插件回调优先于用户定义回调

---

## 错误处理策略

### 1. 插件回调错误

```go
if err != nil {
    return nil, fmt.Errorf("failed to run plugin before agent callback: %w", err)
}
```

### 2. 用户回调错误

```go
if err != nil {
    return nil, fmt.Errorf("failed to run before agent callback: %w", err)
}
```

### 3. 事件产出错误

```go
if !yield(event, err) {
    return  // 消费者提前终止
}
```

**策略总结**:
- 错误包装清晰的上下文信息
- 插件回调和用户回调分别处理
- 支持消费者提前终止迭代

---

## 关键设计决策

### 1. 迭代器返回类型

**选择**: `iter.Seq2[*session.Event, error]`

**优势**:
- 流式输出，支持实时响应
- 错误和事件同时处理
- 消费者可控制迭代终止

### 2. 回调链执行

**顺序**:
1. 插件 BeforeAgentCallback
2. 用户 BeforeAgentCallbacks（顺序执行）
3. Agent.run()
4. 用户 AfterAgentCallbacks（顺序执行）
5. 插件 AfterAgentCallback

**提前退出**: 任一回调返回非 nil 内容，跳过后续回调

### 3. 状态变更追踪

**机制**:
- 回调中通过 `State()` 修改状态
- 变更先写入 `StateDelta`
- 事件创建时携带 `StateDelta`
- 持久化到 Session

### 4. Telemetry 集成

**方式**:
- `StartInvokeAgentSpan()` 创建追踪
- `WrapYield()` 自动记录结果
- `defer endSpan()` 确保清理

### 5. Author 自动设置

**逻辑**:
- 用户角色 → `"user"`
- 其他 → Agent 名称

**目的**: 确保事件可追溯

---

## 与 Runner 的关系

```
Runner.Run()
    │
    └─ agentToRun.Run(ctx)  ← Agent.Run()
        │
        ├─ runBeforeAgentCallbacks()
        │   ├─ Plugin BeforeAgentCallback
        │   └─ User BeforeAgentCallbacks
        │
        ├─ a.run(ctx)  ← LLMAgent.run() / Custom Run
        │
        └─ runAfterAgentCallbacks()
            ├─ User AfterAgentCallbacks
            └─ Plugin AfterAgentCallback
```

**职责划分**:
- **Runner**: 会话管理、Agent 选择、事件持久化
- **Agent**: 回调执行、事件产出、追踪集成

---

## 总结

Agent 作为 ADK Go 的执行核心，具有以下特点：

1. **统一接口**: 所有 Agent 实现 `Run(InvocationContext) iter.Seq2[*session.Event, error]`
2. **回调链机制**: 支持前后置钩子，扩展执行逻辑
3. **流式输出**: 迭代器返回，支持实时响应和提前终止
4. **状态管理**: 通过 `StateDelta` 追踪状态变更
5. **Telemetry 集成**: 自动追踪 Agent 调用和结果
6. **插件支持**: 插件回调优先执行，支持全局扩展

设计优势：
- 灵活的回调机制，支持复杂业务逻辑
- 清晰的执行流程，便于调试和追踪
- 统一的接口设计，便于扩展和组合
