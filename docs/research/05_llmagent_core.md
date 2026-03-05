# ADK Go LLMAgent 核心实现分析

## 概述

LLMAgent 是 ADK Go 中最核心的 Agent 实现，负责与 LLM 模型交互、工具调用和响应生成。本文档深入分析 `agent/llmagent/llmagent.go` 的核心实现。

---

## LLMAgent 结构

### 结构定义

**位置**: `agent/llmagent/llmagent.go:341-353`

```go
type llmAgent struct {
    agent.Agent              // 嵌入基础 Agent 接口
    llminternal.State        // 嵌入 LLM 内部状态
    agentState
    
    beforeModelCallbacks  []llminternal.BeforeModelCallback
    model                 model.LLM
    afterModelCallbacks   []llminternal.AfterModelCallback
    instruction           string
    onModelErrorCallbacks []llminternal.OnModelErrorCallback
    
    beforeToolCallbacks  []llminternal.BeforeToolCallback
    afterToolCallbacks   []llminternal.AfterToolCallback
    onToolErrorCallbacks []llminternal.OnToolErrorCallback
    
    inputSchema  *genai.Schema
    outputSchema *genai.Schema
}
```

**嵌入结构**:
- `agent.Agent`: 基础 Agent 接口，提供 Name/Description/SubAgents 等方法
- `llminternal.State`: LLM 内部状态，包含 Model/Tools/Config 等

---

## New() 构造函数

**位置**: `agent/llmagent/llmagent.go:34-112`

### 方法签名

```go
func New(cfg Config) (agent.Agent, error)
```

### 创建流程

```go
// 1. 转换回调类型（用户定义 → 内部类型）
beforeModelCallbacks := make([]llminternal.BeforeModelCallback, 0, len(cfg.BeforeModelCallbacks))
for _, c := range cfg.BeforeModelCallbacks {
    beforeModelCallbacks = append(beforeModelCallbacks, llminternal.BeforeModelCallback(c))
}
// ... 其他回调类型转换

// 2. 创建 llmAgent 实例
a := &llmAgent{
    model:                 cfg.Model,
    beforeModelCallbacks:  beforeModelCallbacks,
    afterModelCallbacks:   afterModelCallbacks,
    onModelErrorCallbacks: onModelErrorCallbacks,
    beforeToolCallbacks:   beforeToolCallbacks,
    afterToolCallbacks:    afterToolCallbacks,
    onToolErrorCallbacks:  onToolErrorCallback,
    instruction:           cfg.Instruction,
    inputSchema:           cfg.InputSchema,
    outputSchema:          cfg.OutputSchema,
    
    State: llminternal.State{
        Model:                    cfg.Model,
        GenerateContentConfig:    cfg.GenerateContentConfig,
        Tools:                    cfg.Tools,
        Toolsets:                 cfg.Toolsets,
        DisallowTransferToParent: cfg.DisallowTransferToParent,
        DisallowTransferToPeers:  cfg.DisallowTransferToPeers,
        // ... 其他字段
    },
}

// 3. 创建基础 Agent
baseAgent, err := agent.New(agent.Config{
    Name:                 cfg.Name,
    Description:          cfg.Description,
    SubAgents:            cfg.SubAgents,
    BeforeAgentCallbacks: cfg.BeforeAgentCallbacks,
    Run:                  a.run,  // 关键：注入 llmAgent.run 方法
    AfterAgentCallbacks:  cfg.AfterAgentCallbacks,
})

// 4. 设置内部状态
a.Agent = baseAgent
a.AgentType = agentinternal.TypeLLMAgent
a.Config = cfg

return a, nil
```

**设计特点**:
- 类型转换：用户定义的回调类型转换为内部类型
- 组合模式：嵌入 `agent.Agent` 接口
- 注入 `run` 方法：作为基础 Agent 的执行函数

---

## Config 配置结构

**位置**: `agent/llmagent/llmagent.go:115-269`

### 核心配置字段

| 字段 | 类型 | 用途 |
|------|------|------|
| `Name` | string | 代理唯一名称 |
| `Description` | string | 能力描述，LLM 用于判断是否委托 |
| `SubAgents` | []agent.Agent | 子代理列表 |
| `Model` | model.LLM | LLM 模型实例 |
| `Tools` | []tool.Tool | 可用工具列表 |
| `Toolsets` | []tool.Toolset | 工具集（可扩展） |
| `Instruction` | string | 行为指导指令（支持模板） |
| `InstructionProvider` | InstructionProvider | 动态指令提供者 |
| `GlobalInstruction` | string | 全局指令（所有代理共享） |
| `InputSchema` | *genai.Schema | 输入模式（作为工具时） |
| `OutputSchema` | *genai.Schema | 输出模式（结构化响应） |
| `OutputKey` | string | 输出存储到 State 的键名 |
| `GenerateContentConfig` | *genai.GenerateContentConfig | 内容生成配置（温度、安全等） |
| `DisallowTransferToParent` | bool | 禁止转移到父代理 |
| `DisallowTransferToPeers` | bool | 禁止转移到同级代理 |
| `IncludeContents` | IncludeContents | 是否包含对话历史 |

### 回调配置字段

| 字段 | 类型 | 执行时机 |
|------|------|----------|
| `BeforeAgentCallbacks` | []agent.BeforeAgentCallback | Agent 执行前 |
| `AfterAgentCallbacks` | []agent.AfterAgentCallback | Agent 执行后 |
| `BeforeModelCallbacks` | []BeforeModelCallback | LLM 调用前 |
| `AfterModelCallbacks` | []AfterModelCallback | LLM 调用后 |
| `OnModelErrorCallbacks` | []OnModelErrorCallback | LLM 错误时 |
| `BeforeToolCallbacks` | []BeforeToolCallback | 工具调用前 |
| `AfterToolCallbacks` | []AfterToolCallback | 工具调用后 |
| `OnToolErrorCallbacks` | []OnToolErrorCallback | 工具错误时 |

---

## run() 方法实现

**位置**: `agent/llmagent/llmagent.go:347-380`

### 方法签名

```go
func (a *llmAgent) run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error]
```

### 执行流程

```go
func (a *llmAgent) run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error] {
    // 1. 更新上下文（确保 Agent 为当前 llmAgent）
    ctx = icontext.NewInvocationContext(ctx, icontext.InvocationContextParams{
        Artifacts:    ctx.Artifacts(),
        Memory:       ctx.Memory(),
        Session:      ctx.Session(),
        Branch:       ctx.Branch(),
        Agent:        a,  // 关键：设置为当前 llmAgent
        UserContent:  ctx.UserContent(),
        RunConfig:    ctx.RunConfig(),
        InvocationID: ctx.InvocationID(),
    })
    
    // 2. 创建 Flow 对象
    f := &llminternal.Flow{
        Model:                 a.model,
        RequestProcessors:     llminternal.DefaultRequestProcessors,
        ResponseProcessors:    llminternal.DefaultResponseProcessors,
        BeforeModelCallbacks:  a.beforeModelCallbacks,
        AfterModelCallbacks:   a.afterModelCallbacks,
        OnModelErrorCallbacks: a.onModelErrorCallbacks,
        BeforeToolCallbacks:   a.beforeToolCallbacks,
        AfterToolCallbacks:    a.afterToolCallbacks,
        OnToolErrorCallbacks:  a.onToolErrorCallbacks,
    }
    
    // 3. 执行 Flow.Run() 并产出事件
    return func(yield func(*session.Event, error) bool) {
        for ev, err := range f.Run(ctx) {
            a.maybeSaveOutputToState(ev)  // 可选：保存输出到 State
            if !yield(ev, err) {
                return
            }
        }
    }
}
```

**关键步骤**:
1. **上下文更新**: 确保 Agent 字段指向当前 llmAgent
2. **Flow 创建**: 注入 Model、回调、处理器等配置
3. **事件流执行**: 遍历 Flow.Run() 产出的事件流
4. **输出保存**: 可选地将输出保存到 Session State

---

## maybeSaveOutputToState() 方法

**位置**: `agent/llmagent/llmagent.go:382-417`

### 方法签名

```go
func (a *llmAgent) maybeSaveOutputToState(event *session.Event)
```

### 保存逻辑

```go
func (a *llmAgent) maybeSaveOutputToState(event *session.Event) {
    if event == nil {
        return
    }
    
    // 1. 检查事件作者是否为当前 Agent
    if event.Author != a.Name() {
        return  // 跳过其他代理的事件
    }
    
    // 2. 检查 OutputKey 和内容
    if a.OutputKey != "" && !event.Partial && event.Content != nil && len(event.Content.Parts) > 0 {
        var sb strings.Builder
        for _, part := range event.Content.Parts {
            if part.Text != "" && !part.Thought {
                sb.WriteString(part.Text)  // 收集非思考文本
            }
        }
        result := sb.String()
        
        // 3. OutputSchema 验证（如有）
        if a.OutputSchema != nil {
            if strings.TrimSpace(result) == "" {
                return  // 空结果不解析
            }
        }
        
        // 4. 保存到 StateDelta
        if event.Actions.StateDelta == nil {
            event.Actions.StateDelta = make(map[string]any)
        }
        event.Actions.StateDelta[a.OutputKey] = result
    }
}
```

**用途**:
- 将 Agent 输出保存到 Session State
- 支持后续工具、回调使用
- 多代理协作时的数据传递

**触发条件**:
1. `OutputKey` 配置非空
2. 事件作者为当前 Agent
3. 非 Partial 事件
4. 内容非空

---

## Flow 执行引擎

**位置**: `internal/llminternal/base_flow.go:97-122`

### Flow.Run() 循环

```go
func (f *Flow) Run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error] {
    return func(yield func(*session.Event, error) bool) {
        for {
            var lastEvent *session.Event
            
            // 执行一步
            for ev, err := range f.runOneStep(ctx) {
                if err != nil {
                    yield(nil, err)
                    return
                }
                if !yield(ev, nil) {
                    return
                }
                lastEvent = ev
            }
            
            // 检查退出条件
            if lastEvent == nil || lastEvent.IsFinalResponse() {
                return  // 最终响应，退出循环
            }
            
            if lastEvent.LLMResponse.Partial {
                yield(nil, fmt.Errorf("TODO: last event is not final"))
                return
            }
            // 否则继续循环
        }
    }
}
```

**循环逻辑**:
```
for {
    runOneStep() → 事件流
        │
        ├─ yield 每个事件
        └─ 记录 lastEvent
    
    检查退出条件:
    ├─ lastEvent == nil → 退出
    ├─ IsFinalResponse() → 退出
    ├─ Partial → 错误退出
    └─ 否则 → 继续循环
}
```

### runOneStep() 步骤执行

**位置**: `internal/llminternal/base_flow.go:125-243`

```go
func (f *Flow) runOneStep(ctx agent.InvocationContext) iter.Seq2[*session.Event, error] {
    return func(yield func(*session.Event, error) bool) {
        // 1. 检查 Model 配置
        if f.Model == nil {
            yield(nil, fmt.Errorf("agent %q: %w", ctx.Agent().Name(), ErrModelNotConfigured))
            return
        }
        
        req := &model.LLMRequest{Model: f.Model.Name()}
        
        // 2. 预处理
        for ev, err := range f.preprocess(ctx, req) {
            if err != nil {
                yield(nil, err)
                return
            }
            if ev != nil {
                if !yield(ev, nil) { return }
            }
        }
        if ctx.Ended() { return }
        
        // 3. 调用 LLM
        stateDelta := make(map[string]any)
        for resp, err := range f.callLLM(ctx, req, stateDelta) {
            if err != nil {
                yield(nil, err)
                return
            }
            
            // 4. 后处理
            if err := f.postprocess(ctx, req, resp); err != nil {
                yield(nil, err)
                return
            }
            
            // 5. 跳过空响应（代码执行器触发）
            if resp.Content == nil && resp.ErrorCode == "" && !resp.Interrupted {
                continue
            }
            
            // 6. 构建事件
            modelResponseEvent := f.finalizeModelResponseEvent(ctx, resp, tools, stateDelta)
            if !yield(modelResponseEvent, nil) { return }
            
            // 7. 处理函数调用
            ev, err := f.handleFunctionCalls(ctx, tools, resp.LLMResponse, nil)
            if err != nil {
                yield(nil, err)
                return
            }
            if ev == nil { continue }
            
            // 8. 工具确认事件（如需）
            toolConfirmationEvent := generateRequestConfirmationEvent(ctx, modelResponseEvent, ev)
            if toolConfirmationEvent != nil {
                if !yield(toolConfirmationEvent, nil) { return }
            }
            
            if !yield(ev, nil) { return }
            
            // 9. 结构化响应处理
            outputSchemaResponse, err := retrieveStructuredModelResponse(ev)
            if err != nil {
                yield(nil, err)
                return
            }
            if outputSchemaResponse != "" {
                if !yield(createFinalModelResponseEvent(ctx, outputSchemaResponse), nil) {
                    return
                }
            }
            
            // 10. 代理转移处理
            if ev.Actions.TransferToAgent == "" {
                return  // 无转移，退出
            }
            nextAgent := f.agentToRun(ctx, ev.Actions.TransferToAgent)
            if nextAgent == nil {
                yield(nil, fmt.Errorf("failed to find agent: %s", ev.Actions.TransferToAgent))
                return
            }
            // 执行下一个 Agent
            for ev, err := range nextAgent.Run(ctx) {
                if !yield(ev, err) || err != nil {
                    return
                }
            }
        }
    }
}
```

### runOneStep() 流程图

```
runOneStep()
    │
    ├─ 1. 检查 Model 配置
    │
    ├─ 2. preprocess()
    │   └─ 执行 RequestProcessors（指令、身份、内容等）
    │
    ├─ 3. callLLM()
    │   ├─ BeforeModelCallbacks
    │   ├─ generateContent() 实际调用
    │   └─ AfterModelCallbacks
    │
    ├─ 4. postprocess()
    │   └─ 执行 ResponseProcessors（NL Planning、代码执行）
    │
    ├─ 5. finalizeModelResponseEvent()
    │   └─ 构建事件对象
    │
    ├─ 6. handleFunctionCalls()
    │   ├─ 执行工具调用
    │   ├─ BeforeToolCallbacks
    │   ├─ tool.Run()
    │   └─ AfterToolCallbacks
    │
    ├─ 7. 工具确认事件（如需用户确认）
    │
    ├─ 8. 结构化响应处理（OutputSchema）
    │
    └─ 9. 代理转移处理
        └─ TransferToAgent != "" → 执行下一个 Agent
```

---

## RequestProcessors 处理器链

**位置**: `internal/llminternal/base_flow.go:73-90`

```go
var DefaultRequestProcessors = []func(ctx agent.InvocationContext, req *model.LLMRequest, f *Flow) iter.Seq2[*session.Event, error]{
    basicRequestProcessor,           // 基础请求处理
    toolProcessor,                   // 工具处理
    authPreprocessor,                // 认证预处理
    RequestConfirmationRequestProcessor,  // 请求确认
    instructionsRequestProcessor,    // 指令处理
    identityRequestProcessor,        // 身份处理
    ContentsRequestProcessor,        // 内容处理（对话历史）
    nlPlanningRequestProcessor,      // NL Planning
    codeExecutionRequestProcessor,   // 代码执行
    outputSchemaRequestProcessor,    // 输出模式
    AgentTransferRequestProcessor,   // 代理转移
    removeDisplayNameIfExists,       // 移除显示名称
}
```

**执行顺序**: 按定义顺序依次执行

**关键处理器**:
1. **basicRequestProcessor**: 初始化请求基础字段
2. **toolProcessor**: 添加可用工具到请求
3. **instructionsRequestProcessor**: 注入 Instruction 指令
4. **identityRequestProcessor**: 添加代理身份信息
5. **ContentsRequestProcessor**: 添加对话历史
6. **AgentTransferRequestProcessor**: 处理代理转移工具

---

## ResponseProcessors 响应处理器链

**位置**: `internal/llminternal/base_flow.go:91-94`

```go
var DefaultResponseProcessors = []func(ctx agent.InvocationContext, req *model.LLMRequest, resp *model.LLMResponse) error{
    nlPlanningResponseProcessor,     // NL Planning 后处理
    codeExecutionResponseProcessor,  // 代码执行后处理
}
```

**用途**: 对 LLM 响应进行后处理和转换

---

## callLLM() LLM 调用

**位置**: `internal/llminternal/base_flow.go:290-350`

```go
func (f *Flow) callLLM(ctx agent.InvocationContext, req *model.LLMRequest, stateDelta map[string]any) iter.Seq2[*responseWithEventID, error] {
    return func(yield func(*responseWithEventID, error) bool) {
        pluginManager := pluginManagerFromContext(ctx)
        
        // 1. 插件 BeforeModelCallback
        if pluginManager != nil {
            cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
            callbackResponse, callbackErr := pluginManager.RunBeforeModelCallback(cctx, req)
            if callbackResponse != nil || callbackErr != nil {
                yield(newResponseWithEventID(callbackResponse), callbackErr)
                return
            }
        }
        
        // 2. 用户 BeforeModelCallbacks
        for _, callback := range f.BeforeModelCallbacks {
            cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
            callbackResponse, callbackErr := callback(cctx, req)
            if callbackResponse != nil || callbackErr != nil {
                yield(newResponseWithEventID(callbackResponse), callbackErr)
                return
            }
        }
        
        // 3. 实际调用 LLM
        useStream := runconfig.FromContext(ctx).StreamingMode == runconfig.StreamingModeSSE
        for resp, err := range generateContent(ctx, f.Model, req, useStream) {
            // 4. 错误处理
            if err != nil {
                cbResp, cbErr := f.runOnModelErrorCallbacks(ctx, req, stateDelta, err)
                if cbErr != nil {
                    yield(nil, cbErr)
                    return
                }
                if cbResp == nil {
                    yield(nil, err)
                    return
                }
                resp = &responseWithEventID{LLMResponse: cbResp, eventID: resp.eventID}
            }
            
            // 5. 填充函数调用 ID
            utils.PopulateClientFunctionCallID(resp.Content)
            
            // 6. AfterModelCallbacks
            callbackResp, callbackErr := f.runAfterModelCallbacks(ctx, resp.LLMResponse, stateDelta, err)
            if callbackErr != nil {
                yield(nil, callbackErr)
                return
            }
            if callbackResp != nil {
                resp = &responseWithEventID{LLMResponse: callbackResp, eventID: resp.eventID}
            }
            
            yield(resp, nil)
        }
    }
}
```

**调用顺序**:
1. 插件 BeforeModelCallback
2. 用户 BeforeModelCallbacks
3. generateContent() 实际调用（流式/非流式）
4. OnModelErrorCallbacks（错误时）
5. AfterModelCallbacks

---

## 回调类型定义

### BeforeModelCallback

**位置**: `agent/llmagent/llmagent.go:276`

```go
type BeforeModelCallback func(ctx agent.CallbackContext, llmRequest *model.LLMRequest) (*model.LLMResponse, error)
```

**用途**:
- 检查/修改 LLM 请求
- 实现缓存（返回缓存响应跳过 LLM 调用）
- 日志记录

### AfterModelCallback

**位置**: `agent/llmagent/llmagent.go:286`

```go
type AfterModelCallback func(ctx agent.CallbackContext, llmResponse *model.LLMResponse, llmResponseError error) (*model.LLMResponse, error)
```

**用途**:
- 日志记录
- Token 使用统计
- 响应后处理

### OnModelErrorCallback

**位置**: `agent/llmagent/llmagent.go:292`

```go
type OnModelErrorCallback func(ctx agent.CallbackContext, llmRequest *model.LLMRequest, llmResponseError error) (*model.LLMResponse, error)
```

**用途**:
- 错误恢复
- 重试逻辑
- 降级处理

---

## InstructionProvider 动态指令

**位置**: `agent/llmagent/llmagent.go:419-425`

```go
type InstructionProvider func(ctx agent.ReadonlyContext) (string, error)
```

**特点**:
- 动态生成指令（每调用一次）
- 不使用模板替换（原始字符串）
- 优先级高于静态 Instruction

**使用场景**:
- 根据会话状态动态调整指令
- 多租户场景的个性化指令
- 需要复杂逻辑生成指令

---

## IncludeContents 配置

**位置**: `agent/llmagent/llmagent.go:319-325`

```go
type IncludeContents string

const (
    IncludeContentsNone    IncludeContents = "none"      // 不包含对话历史
    IncludeContentsDefault IncludeContents = "default"   // 包含相关历史（默认）
)
```

**用途**: 控制 LLM 接收的对话历史范围

---

## 关键设计决策

### 1. Flow 循环执行模式

**设计**: 多步循环执行，每步调用 LLM

**优势**:
- 支持工具调用→LLM 总结→更多工具调用的链式执行
- 自动处理多轮交互
- 通过 `IsFinalResponse()` 判断退出

### 2. Request/Response Processor 链

**设计**: 预定义处理器链，按顺序执行

**优势**:
- 模块化设计，每个处理器职责单一
- 易于扩展和定制
- 清晰的执行顺序

### 3. 回调优先级

**顺序**:
1. 插件回调（全局）
2. 用户定义回调（Agent 级别）
3. 实际执行

**优势**:
- 插件可全局拦截和修改
- 用户可定制特定 Agent 行为
- 层次清晰，职责分离

### 4. 输出保存到 State

**机制**: 通过 `OutputKey` 配置自动保存

**用途**:
- 多代理协作数据传递
- 后续工具调用参考
- 状态追踪

### 5. 代理转移机制

**流程**:
```
LLM 返回 FunctionCall("transfer_to_agent", targetName)
    ↓
handleFunctionCalls() 设置 ev.Actions.TransferToAgent
    ↓
检查 TransferToAgent != ""
    ↓
查找目标 Agent
    ↓
执行 nextAgent.Run(ctx)
```

**优势**:
- LLM 自主决定转移目标
- 自动处理上下文传递
- 支持层级转移

---

## 与 Agent 接口的关系

```
agent.Agent (接口)
    ↑
    │ 嵌入
llmAgent (结构)
    │
    └─ run() → Flow.Run()
                   │
                   ├─ preprocess()
                   ├─ callLLM()
                   ├─ postprocess()
                   └─ handleFunctionCalls()
```

**职责划分**:
- **Agent 接口**: 统一执行入口，回调处理
- **LLMAgent**: LLM 特定逻辑，Flow 创建
- **Flow**: 核心执行引擎，循环调用

---

## 总结

LLMAgent 作为 ADK Go 的核心实现，具有以下特点：

1. **组合设计**: 嵌入 `agent.Agent` 接口，复用基础功能
2. **Flow 引擎**: 循环执行模式，自动处理工具调用链
3. **处理器链**: Request/Response 处理器模块化设计
4. **回调机制**: 多层回调（插件/用户），灵活扩展
5. **代理转移**: LLM 自主决定转移目标
6. **输出保存**: 通过 `OutputKey` 自动保存到 State

设计优势：
- 清晰的职责划分（Agent/LLMAgent/Flow）
- 模块化设计，易于扩展
- 灵活的回调和处理器机制
- 自动处理复杂的多轮交互
