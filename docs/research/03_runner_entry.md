# ADK Go Runner 入口分析

## 概述

Runner 是 ADK Go 的用户请求入口，负责协调会话管理、Agent 查找、上下文构建和事件输出。本文档深入分析 `runner/runner.go` 的核心实现。

---

## Runner 结构

### 结构定义

**位置**: `runner/runner.go:100-109`

```go
type Runner struct {
    appName         string
    rootAgent       agent.Agent
    sessionService  session.Service
    artifactService artifact.Service
    memoryService   memory.Service
    
    parents       parentmap.Map           // 代理父子关系映射
    pluginManager *plugininternal.PluginManager  // 插件管理器
}
```

**职责**:
- 管理 Agent 执行生命周期
- 协调会话、资源、记忆服务
- 处理插件回调
- 产出事件流

### Config 配置

**位置**: `runner/runner.go:44-61`

```go
type Config struct {
    AppName         string
    Agent           agent.Agent        // 根 Agent
    SessionService  session.Service    // 必需
    
    ArtifactService artifact.Service   // 可选
    MemoryService   memory.Service     // 可选
    PluginConfig    PluginConfig       // 可选
}
```

---

## Runner.Run() 核心流程

**位置**: `runner/runner.go:114-233`

### 方法签名

```go
func (r *Runner) Run(
    ctx context.Context, 
    userID, sessionID string, 
    msg *genai.Content, 
    cfg agent.RunConfig
) iter.Seq2[*session.Event, error]
```

**返回类型**: `iter.Seq2[*session.Event, error]` - Go 1.22+ 泛型迭代器，产出事件和错误对。

### 完整流程图

```
Runner.Run()
    │
    ├─ 1. 获取会话
    │   └─ sessionService.Get()
    │
    ├─ 2. 查找 Agent
    │   └─ findAgentToRun()
    │
    ├─ 3. 构建上下文
    │   ├─ parentmap.ToContext()
    │   ├─ runconfig.ToContext()
    │   ├─ plugininternal.ToContext()
    │   └─ icontext.NewInvocationContext()
    │
    ├─ 4. 初始化服务包装
    │   ├─ Artifacts (可选)
    │   └─ Memory (可选)
    │
    ├─ 5. 追加用户消息到会话
    │   └─ appendMessageToSession()
    │
    ├─ 6. 执行插件回调
    │   ├─ BeforeRunCallback
    │   └─ defer AfterRunCallback
    │
    └─ 7. 执行 Agent.Run()
        ├─ 遍历事件流
        ├─ OnEventCallback 处理
        ├─ AppendEvent 持久化
        └─ yield 产出事件
```

---

## 详细步骤分析

### 步骤 1：获取会话

**位置**: `runner/runner.go:119-129`

```go
resp, err := r.sessionService.Get(ctx, &session.GetRequest{
    AppName:   r.appName,
    UserID:    userID,
    SessionID: sessionID,
})
if err != nil {
    yield(nil, err)
    return
}

storedSession := resp.Session
```

**关键点**:
- 使用 `sessionService.Get()` 获取会话
- 错误直接通过 `yield(nil, err)` 返回
- 会话是后续所有操作的基础

---

### 步骤 2：查找 Agent

**位置**: `runner/runner.go:131-135`

```go
agentToRun, err := r.findAgentToRun(storedSession, msg)
if err != nil {
    yield(nil, err)
    return
}
```

**职责**: 根据会话历史确定哪个 Agent 应该处理当前请求。

**查找逻辑** (详见 `findAgentToRun()`):
1. 优先处理用户函数调用响应
2. 从后向前遍历事件，找到最后一个非用户 Agent
3. 检查 Agent 是否可转移
4. 回退到根 Agent

---

### 步骤 3：构建上下文

**位置**: `runner/runner.go:137-142`

```go
ctx = parentmap.ToContext(ctx, r.parents)
ctx = runconfig.ToContext(ctx, &runconfig.RunConfig{
    StreamingMode: runconfig.StreamingMode(cfg.StreamingMode),
})
ctx = plugininternal.ToContext(ctx, r.pluginManager)
```

**上下文层级**:
1. **parentmap**: 代理父子关系，用于代理转移
2. **runconfig**: 运行配置（流式模式等）
3. **plugininternal**: 插件管理器

---

### 步骤 4：初始化服务包装

**位置**: `runner/runner.go:144-161`

```go
var artifacts agent.Artifacts
if r.artifactService != nil {
    artifacts = &artifactinternal.Artifacts{
        Service:   r.artifactService,
        SessionID: storedSession.ID(),
        AppName:   storedSession.AppName(),
        UserID:    storedSession.UserID(),
    }
}

var memoryImpl agent.Memory = nil
if r.memoryService != nil {
    memoryImpl = &imemory.Memory{
        Service:   r.memoryService,
        SessionID: storedSession.ID(),
        UserID:    storedSession.UserID(),
        AppName:   storedSession.AppName(),
    }
}
```

**设计特点**:
- 服务包装器注入会话元数据
- 可选服务，为 nil 时不使用
- 统一接口，便于 Agent 调用

---

### 步骤 5：创建 InvocationContext

**位置**: `runner/runner.go:163-170`

```go
ctx := icontext.NewInvocationContext(ctx, icontext.InvocationContextParams{
    Artifacts:   artifacts,
    Memory:      memoryImpl,
    Session:     storedSession,
    Agent:       agentToRun,
    UserContent: msg,
    RunConfig:   &cfg,
})
```

**InvocationContext 包含**:
| 字段 | 用途 |
|------|------|
| `Artifacts` | 资源文件服务 |
| `Memory` | 长期记忆服务 |
| `Session` | 会话状态 |
| `Agent` | 当前执行 Agent |
| `UserContent` | 用户输入内容 |
| `RunConfig` | 运行配置 |
| `InvocationID` | 调用唯一标识（自动生成） |

---

### 步骤 6：追加用户消息到会话

**位置**: `runner/runner.go:171-175`

```go
ctx, err = r.appendMessageToSession(ctx, storedSession, msg, cfg.SaveInputBlobsAsArtifacts, r.pluginManager)
if err != nil {
    yield(nil, err)
    return
}
```

**详细分析** (见 `appendMessageToSession()`):

#### 子步骤 6.1：OnUserMessageCallback

**位置**: `runner/runner.go:239-257`

```go
if pluginManager != nil {
    modifiedMsg, err := pluginManager.RunOnUserMessageCallback(ctx, msg)
    if err != nil {
        return ctx, fmt.Errorf("error running on run user message callback : %w", err)
    }
    if modifiedMsg != nil {
        msg = modifiedMsg
        // 更新上下文中的用户消息
    }
}
```

#### 子步骤 6.2：保存输入为资源文件

**位置**: `runner/runner.go:259-273`

```go
if artifactsService != nil && saveInputBlobsAsArtifacts {
    for i, part := range msg.Parts {
        if part.InlineData == nil {
            continue
        }
        fileName := fmt.Sprintf("artifact_%s_%d", ctx.InvocationID(), i)
        if _, err := artifactsService.Save(ctx, fileName, part); err != nil {
            return ctx, fmt.Errorf("failed to save artifact %s: %w", fileName, err)
        }
        // 替换为文本占位符
        msg.Parts[i] = &genai.Part{
            Text: fmt.Sprintf("Uploaded file: %s...", fileName),
        }
    }
}
```

**用途**: 将用户上传的文件（图片、文档等）保存到资源服务，LLM 只看到文本引用。

#### 子步骤 6.3：创建用户消息事件

**位置**: `runner/runner.go:275-285`

```go
event := session.NewEvent(ctx.InvocationID())
event.Author = "user"
event.LLMResponse = model.LLMResponse{
    Content: msg,
}

if err := r.sessionService.AppendEvent(ctx, storedSession, event); err != nil {
    return ctx, fmt.Errorf("failed to append event to sessionService: %w", err)
}
```

---

### 步骤 7：执行插件回调

**位置**: `runner/runner.go:177-197`

```go
pluginManager := r.pluginManager
if pluginManager != nil {
    // 延迟执行 AfterRunCallback
    defer pluginManager.RunAfterRunCallback(ctx)
    
    // 执行 BeforeRunCallback
    earlyExitResult, err := pluginManager.RunBeforeRunCallback(ctx)
    if earlyExitResult != nil || err != nil {
        // 提前退出处理
        earlyExitEvent := session.NewEvent(ctx.InvocationID())
        earlyExitEvent.Author = "user"
        earlyExitEvent.LLMResponse = model.LLMResponse{
            Content: msg,
        }
        if err := r.sessionService.AppendEvent(ctx, storedSession, earlyExitEvent); err != nil {
            yield(nil, fmt.Errorf("failed to add event to session: %w", err))
            return
        }
        yield(earlyExitEvent, err)
        return
    }
}
```

**回调时机**:
- **BeforeRunCallback**: Agent 执行前，可修改上下文或提前退出
- **AfterRunCallback**: 延迟执行，清理和收尾工作

---

### 步骤 8：执行 Agent.Run()

**位置**: `runner/runner.go:199-232`

```go
for event, err := range agentToRun.Run(ctx) {
    if err != nil {
        if !yield(event, err) {
            return
        }
        continue
    }
    
    // OnEventCallback 处理
    if pluginManager != nil {
        modifiedEvent, err := pluginManager.RunOnEventCallback(ctx, event)
        if err != nil {
            if !yield(nil, err) {
                return
            }
            continue
        }
        if modifiedEvent != nil {
            event = modifiedEvent
        }
    }
    
    // 持久化非 Partial 事件
    if !event.LLMResponse.Partial {
        if err := r.sessionService.AppendEvent(ctx, storedSession, event); err != nil {
            yield(nil, fmt.Errorf("failed to add event to session: %w", err))
            return
        }
    }
    
    if !yield(event, nil) {
        return
    }
}
```

**关键处理**:
1. **错误处理**: 错误事件也产出，继续处理后续事件
2. **OnEventCallback**: 插件可修改事件
3. **Partial 事件**: 不持久化，只产出给调用者
4. **yield**: 控制迭代器产出，支持消费者提前终止

---

## findAgentToRun() 详解

**位置**: `runner/runner.go:289-321`

### 方法签名

```go
func (r *Runner) findAgentToRun(session session.Session, msg *genai.Content) (agent.Agent, error)
```

### 查找逻辑

```go
// 1. 优先处理用户函数调用响应
if event := handleUserFunctionCallResponse(session.Events(), msg); event != nil {
    subAgent := findAgent(r.rootAgent, event.Author)
    if subAgent != nil {
        return subAgent, nil
    }
}

// 2. 从后向前遍历事件
events := session.Events()
for i := events.Len() - 1; i >= 0; i-- {
    event := events.At(i)
    
    if event.Author == "user" {
        continue
    }
    
    subAgent := findAgent(r.rootAgent, event.Author)
    if subAgent == nil {
        continue
    }
    
    // 3. 检查可转移性
    if r.isTransferableAcrossAgentTree(subAgent) {
        return subAgent, nil
    }
}

// 4. 回退到根 Agent
return r.rootAgent, nil
```

### 设计意图

1. **函数调用优先**: 用户响应函数调用时，回到原 Agent
2. **最近对话优先**: 从后向前找，找到最后一个活跃的 Agent
3. **可转移检查**: 确保 Agent 链允许跨树转移
4. **根 Agent 兜底**: 新对话或无合适 Agent 时，使用根 Agent

---

## handleUserFunctionCallResponse() 详解

**位置**: `runner/runner.go:323-349`

### 方法签名

```go
func handleUserFunctionCallResponse(events session.Events, msg *genai.Content) *session.Event
```

### 处理逻辑

```go
// 1. 检查是否有函数响应
functionResponses := utils.FunctionResponses(msg)
if len(functionResponses) == 0 {
    return nil
}

// 2. 获取第一个函数调用 ID
callID := functionResponses[0].ID

// 3. 从后向前查找匹配的函数调用事件
for i := events.Len() - 1; i >= 0; i-- {
    event := events.At(i)
    for _, part := range utils.FunctionCalls(event.Content) {
        if part.ID == callID {
            return event  // 找到匹配的调用事件
        }
    }
}
return nil
```

### 用途

- 用户返回工具调用结果时，找到原始调用事件
- 确定应该由哪个 Agent 处理响应
- 假设同一调用的所有函数响应来自同一 Agent

---

## isTransferableAcrossAgentTree() 详解

**位置**: `runner/runner.go:351-364`

### 方法签名

```go
func (r *Runner) isTransferableAcrossAgentTree(agentToRun agent.Agent) bool
```

### 检查逻辑

```go
for curAgent := agentToRun; curAgent != nil; curAgent = r.parents[curAgent.Name()] {
    llmAgent, ok := curAgent.(llminternal.Agent)
    if !ok {
        return false
    }
    
    if llminternal.Reveal(llmAgent).DisallowTransferToParent {
        return false
    }
}
return true
```

### 检查内容

- 遍历 Agent 到根节点的整条路径
- 每个节点必须是 LLMAgent
- 检查 `DisallowTransferToParent` 标志
- 任一节点禁止转移则返回 false

---

## findAgent() 递归查找

**位置**: `runner/runner.go:366-376`

```go
func findAgent(curAgent agent.Agent, targetName string) agent.Agent {
    if curAgent == nil || curName() == targetName {
        return curAgent
    }
    
    for _, subAgent := range curAgent.SubAgents() {
        if agent := findAgent(subAgent, targetName); agent != nil {
            return agent
        }
    }
    return nil
}
```

**算法**: DFS 深度优先搜索，遍历 Agent 树查找目标名称。

---

## 关键设计决策

### 1. 迭代器返回类型

**选择**: `iter.Seq2[*session.Event, error]`

**优势**:
- 流式输出，支持实时响应
- 错误和事件同时处理
- 消费者可提前终止（`yield` 返回 false）
- 符合 Go 1.22+ 最佳实践

### 2. 会话获取策略

**策略**: 每次 Run() 都重新获取会话

**原因**:
- 保证数据一致性
- 支持并发请求
- 避免状态过期

### 3. 事件持久化时机

**策略**: Agent 执行后持久化，Partial 事件除外

**原因**:
- Partial 事件是流式中间状态
- 最终事件才需要持久化
- 减少存储压力

### 4. 插件回调链

**顺序**:
1. OnUserMessageCallback (用户消息处理)
2. BeforeRunCallback (执行前)
3. OnEventCallback (每个事件)
4. AfterRunCallback (延迟执行)

### 5. Agent 查找优先级

1. 函数调用响应匹配（最高优先级）
2. 最近对话的 Agent
3. 根 Agent（兜底）

---

## 与 Flow 的关系

```
Runner.Run()
    │
    └─ agentToRun.Run(ctx)  ← LLMAgent.run()
        │
        └─ Flow.Run()  ← internal/llminternal/base_flow.go
            │
            └─ runOneStep() 循环
                ├─ preprocess()
                ├─ callLLM()
                ├─ postprocess()
                └─ handleFunctionCalls()
```

**Runner 职责**:
- 会话和上下文管理
- Agent 选择和调度
- 事件持久化和输出

**Flow 职责**:
- LLM 调用循环
- 工具调用处理
- 响应生成

---

## 错误处理策略

### 1. 会话错误

```go
if err != nil {
    yield(nil, err)
    return
}
```

### 2. Agent 执行错误

```go
if err != nil {
    if !yield(event, err) {
        return
    }
    continue  // 继续处理后续事件
}
```

### 3. 事件持久化错误

```go
if err := r.sessionService.AppendEvent(ctx, storedSession, event); err != nil {
    yield(nil, fmt.Errorf("failed to add event to session: %w", err))
    return
}
```

**策略总结**:
- 关键错误（会话获取、持久化）直接返回
- Agent 执行错误产出后继续
- 所有错误包装清晰的上下文信息

---

## 总结

Runner 作为 ADK Go 的请求入口，承担以下核心职责：

1. **会话管理**: 获取、更新会话状态
2. **Agent 调度**: 智能选择执行 Agent
3. **上下文构建**: 注入服务、配置、元数据
4. **事件流处理**: 迭代器输出、持久化、插件回调
5. **错误处理**: 分层错误处理策略

设计特点：
- 流式迭代器返回，支持实时响应
- 插件系统深度集成，支持扩展
- 智能 Agent 查找，支持复杂工作流
- 清晰的分层职责，便于维护和扩展
