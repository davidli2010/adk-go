# ADK Go 工具调用流程详解

## 概述

本文档深入分析 ADK Go 中工具调用的完整流程，包括 BeforeToolCallback 执行、tool.Run() 实际执行、AfterToolCallback 执行以及 FunctionResponse Event 创建机制。

---

## 工具调用流程概览

**位置**: `internal/llminternal/base_flow.go:559-652`

### 完整调用链

```
handleFunctionCalls()
    │
    ├─ 1. 获取 FunctionCalls 列表
    │
    ├─ 2. 创建合并 Span（多个工具时）
    │
    ├─ 3. 遍历每个 FunctionCall
    │   ├─ StartExecuteToolSpan
    │   ├─ 创建 toolCtx
    │   ├─ 查找工具
    │   ├─ callTool()
    │   │   ├─ BeforeToolCallbacks
    │   │   ├─ tool.Run()
    │   │   ├─ OnToolErrorCallbacks (错误时)
    │   │   └─ AfterToolCallbacks
    │   ├─ 创建 FunctionResponse Event
    │   └─ TraceToolResult
    │
    ├─ 4. 合并并行工具响应
    │
    └─ 返回合并的事件
```

---

## handleFunctionCalls() 主函数

**位置**: `internal/llminternal/base_flow.go:559-652`

### 方法签名

```go
func (f *Flow) handleFunctionCalls(
    ctx agent.InvocationContext,
    toolsDict map[string]tool.Tool,
    resp *model.LLMResponse,
    toolConfirmations map[string]*toolconfirmation.ToolConfirmation
) (mergedEvent *session.Event, err error)
```

### 执行流程

```go
func (f *Flow) handleFunctionCalls(ctx agent.InvocationContext, toolsDict map[string]tool.Tool, resp *model.LLMResponse, toolConfirmations map[string]*toolconfirmation.ToolConfirmation) (mergedEvent *session.Event, err error) {
    var fnResponseEvents []*session.Event
    
    // 1. 获取函数调用列表
    fnCalls := utils.FunctionCalls(resp.Content)
    toolNames := slices.Collect(maps.Keys(toolsDict))
    var result map[string]any
    
    // 2. 创建合并 Span（多个工具调用时）
    if len(fnCalls) > 1 {
        mergedCtx, mergedToolCallSpan := telemetry.StartTrace(ctx, "execute_tool (merged)")
        ctx = ctx.WithContext(mergedCtx)
        defer func() {
            telemetry.TraceMergedToolCallsResult(mergedToolCallSpan, mergedEvent, err)
            mergedToolCallSpan.End()
        }()
    }
    
    // 3. 遍历每个函数调用
    for _, fnCall := range fnCalls {
        func() {
            // 启动工具执行 Span
            sctx, span := telemetry.StartExecuteToolSpan(ctx, telemetry.StartExecuteToolSpanParams{
                ToolName: fnCall.Name,
                Args:     fnCall.Args,
            })
            defer span.End()
            toolCallCtx := ctx.WithContext(sctx)
            
            // 获取工具确认（如需）
            var confirmation *toolconfirmation.ToolConfirmation
            if toolConfirmations != nil {
                confirmation = toolConfirmations[fnCall.ID]
            }
            
            // 创建工具上下文
            toolCtx := toolinternal.NewToolContext(toolCallCtx, fnCall.ID, &session.EventActions{StateDelta: make(map[string]any)}, confirmation)
            
            // 4. 查找工具
            curTool, found := toolsDict[fnCall.Name]
            if !found {
                // 工具未找到
                err := newToolNotFoundError(fnCall.Name, toolNames)
                result, err = f.runOnToolErrorCallbacks(toolCtx, &fakeTool{name: fnCall.Name}, fnCall.Args, err)
                if err != nil {
                    result = map[string]any{"error": err.Error()}
                }
            } else if funcTool, ok := curTool.(toolinternal.FunctionTool); !ok {
                // 工具类型不匹配
                err := newToolNotFoundError(fnCall.Name, toolNames)
                result, err = f.runOnToolErrorCallbacks(toolCtx, &fakeTool{name: fnCall.Name}, fnCall.Args, err)
                if err != nil {
                    result = map[string]any{"error": err.Error()}
                }
            } else {
                // 5. 调用工具
                result = f.callTool(toolCtx, funcTool, fnCall.Args)
            }
            
            // 6. 创建 FunctionResponse 事件
            ev := session.NewEvent(ctx.InvocationID())
            ev.LLMResponse = model.LLMResponse{
                Content: &genai.Content{
                    Role: "user",
                    Parts: []*genai.Part{
                        {
                            FunctionResponse: &genai.FunctionResponse{
                                ID:       fnCall.ID,
                                Name:     fnCall.Name,
                                Response: result,
                            },
                        },
                    },
                },
            }
            ev.Author = ctx.Agent().Name()
            ev.Branch = ctx.Branch()
            ev.Actions = *toolCtx.Actions()
            
            // 7. 记录工具执行结果
            traceTool := curTool
            if traceTool == nil {
                traceTool = &fakeTool{name: fnCall.Name}
            }
            var toolErr error
            resultErr := result["error"]
            if resultErr != nil {
                if err, ok := resultErr.(error); ok {
                    toolErr = err
                } else if errStr, ok := resultErr.(string); ok {
                    toolErr = errors.New(errStr)
                }
            }
            telemetry.TraceToolResult(span, telemetry.TraceToolResultParams{
                Description:   traceTool.Description(),
                ResponseEvent: ev,
                Error:         toolErr,
            })
            
            fnResponseEvents = append(fnResponseEvents, ev)
        }()
    }
    
    // 7. 合并并行工具响应
    mergedEvent, err = mergeParallelFunctionResponseEvents(fnResponseEvents)
    return mergedEvent, err
}
```

### 并行工具调用处理

```go
// 创建合并 Span（多个工具调用时）
if len(fnCalls) > 1 {
    mergedCtx, mergedToolCallSpan := telemetry.StartTrace(ctx, "execute_tool (merged)")
    ctx = ctx.WithContext(mergedCtx)
    defer func() {
        telemetry.TraceMergedToolCallsResult(mergedToolCallSpan, mergedEvent, err)
        mergedToolCallSpan.End()
    }()
}
```

**设计意图**:
- 多个工具调用时创建合并 Span
- 统一追踪并行执行的工具
- 延迟结束 Span 直到所有工具完成

---

## callTool() 工具实际执行

**位置**: `internal/llminternal/base_flow.go:665-710`

### 方法签名

```go
func (f *Flow) callTool(toolCtx tool.Context, tool toolinternal.FunctionTool, fArgs map[string]any) map[string]any
```

### 完整执行流程

```go
func (f *Flow) callTool(toolCtx tool.Context, tool toolinternal.FunctionTool, fArgs map[string]any) map[string]any {
    var response map[string]any
    var err error
    pluginManager := pluginManagerFromContext(toolCtx)
    
    // 1. BeforeToolCallbacks
    if pluginManager != nil {
        response, err = pluginManager.RunBeforeToolCallback(toolCtx, tool, fArgs)
    }
    if response == nil && err == nil {
        response, err = f.invokeBeforeToolCallbacks(toolCtx, tool, fArgs)
    }
    
    // 2. 实际工具执行
    if response == nil && err == nil {
        response, err = tool.Run(toolCtx, fArgs)
    }
    
    // 3. OnToolErrorCallbacks (错误时)
    var errorResponse map[string]any
    var cbErr error
    if err != nil && pluginManager != nil {
        errorResponse, cbErr = pluginManager.RunOnToolErrorCallback(toolCtx, tool, fArgs, err)
    }
    if err != nil && errorResponse == nil && cbErr == nil {
        errorResponse, cbErr = f.invokeOnToolErrorCallbacks(toolCtx, tool, fArgs, err)
    }
    if errorResponse != nil || cbErr != nil {
        response = errorResponse
        err = cbErr
    }
    
    // 4. AfterToolCallbacks
    var alteredResponse map[string]any
    var alteredErr error
    if pluginManager != nil {
        alteredResponse, alteredErr = pluginManager.RunAfterToolCallback(toolCtx, tool, fArgs, response, err)
    }
    if alteredResponse == nil && alteredErr == nil {
        alteredResponse, alteredErr = f.invokeAfterToolCallbacks(toolCtx, tool, fArgs, response, err)
    }
    if alteredResponse != nil || alteredErr != nil {
        response = alteredResponse
        err = alteredErr
    }
    
    // 5. 错误处理
    if err != nil {
        return map[string]any{"error": err.Error()}
    }
    return response
}
```

### 回调执行顺序图

```
callTool()
    │
    ├─ 1. PluginManager.RunBeforeToolCallback()
    │   └─ 返回非 nil → 跳过工具执行
    │
    ├─ 2. BeforeToolCallbacks (依次)
    │   └─ 返回非 nil → 跳过工具执行
    │
    ├─ 3. tool.Run() - 实际执行
    │   └─ 可能返回错误
    │
    ├─ 4. OnToolErrorCallbacks (仅错误时)
    │   ├─ PluginManager
    │   └─ 用户回调
    │
    ├─ 5. AfterToolCallbacks (始终执行)
    │   ├─ PluginManager
    │   └─ 用户回调
    │
    └─ 6. 错误包装
```

---

## BeforeToolCallback 执行

**位置**: `internal/llminternal/base_flow.go:712-725`

### 方法签名

```go
type BeforeToolCallback func(ctx tool.Context, tool tool.Tool, args map[string]any) (map[string]any, error)
```

### 执行流程

```go
func (f *Flow) invokeBeforeToolCallbacks(toolCtx tool.Context, tool tool.Tool, fArgs map[string]any) (map[string]any, error) {
    for _, callback := range f.BeforeToolCallbacks {
        result, err := callback(toolCtx, tool, fArgs)
        if err != nil {
            return nil, err
        }
        // 返回非 nil 结果，跳过后续回调和工具执行
        if result != nil {
            return result, nil
        }
    }
    return nil, nil
}
```

### 回调参数

| 参数 | 类型 | 用途 |
|------|------|------|
| `ctx` | tool.Context | 工具上下文 |
| `tool` | tool.Tool | 工具实例 |
| `args` | map[string]any | 工具参数 |

### 返回行为

| 返回值 | 行为 |
|--------|------|
| `result != nil` | 使用返回结果，跳过工具执行 |
| `err != nil` | 返回错误，跳过工具执行 |
| `nil, nil` | 继续执行下一个回调或工具 |

### 使用场景

```go
// 参数验证
func validateArgsCallback(ctx tool.Context, tool tool.Tool, args map[string]any) (map[string]any, error) {
    if err := validate(args); err != nil {
        return nil, err
    }
    return nil, nil  // 继续执行
}

// 参数修改
func modifyArgsCallback(ctx tool.Context, tool tool.Tool, args map[string]any) (map[string]any, error) {
    args["processed"] = true
    return args, nil  // 修改参数但继续执行
}

// 缓存检查
func cacheCallback(ctx tool.Context, tool tool.Tool, args map[string]any) (map[string]any, error) {
    if cached, ok := cache.Get(tool.Name(), args); ok {
        return cached, nil  // 返回缓存，跳过执行
    }
    return nil, nil
}
```

---

## tool.Run() 实际执行

**位置**: `internal/llminternal/base_flow.go:676-678`

```go
if response == nil && err == nil {
    response, err = tool.Run(toolCtx, fArgs)
}
```

### 执行入口

- 实际调用 `tool.Run(toolCtx, fArgs)`
- 参数：工具上下文和参数映射
- 返回：工具执行结果 `map[string]any` 和错误

### 工具查找

**位置**: `internal/llminternal/base_flow.go:588-603`

```go
curTool, found := toolsDict[fnCall.Name]
if !found {
    // 工具未找到
    err := newToolNotFoundError(fnCall.Name, toolNames)
    result, err = f.runOnToolErrorCallbacks(toolCtx, &fakeTool{name: fnCall.Name}, fnCall.Args, err)
    if err != nil {
        result = map[string]any{"error": err.Error()}
    }
} else if funcTool, ok := curTool.(toolinternal.FunctionTool); !ok {
    // 工具类型不匹配
    err := newToolNotFoundError(fnCall.Name, toolNames)
    result, err = f.runOnToolErrorCallbacks(toolCtx, &fakeTool{name: fnCall.Name}, fnCall.Args, err)
    if err != nil {
        result = map[string]any{"error": err.Error()}
    }
} else {
    // 正常调用
    result = f.callTool(toolCtx, funcTool, fnCall.Args)
}
```

### 工具未找到错误

**位置**: `internal/llminternal/base_flow.go:537-553`

```go
func newToolNotFoundError(toolName string, availableTools []string) error {
    joinedTools := strings.Join(availableTools, ", ")
    
    return fmt.Errorf(`tool '%s' not found.
Available tools: %s

Possible causes:
  1. LLM hallucinated the function name - review agent instruction clarity
  2. Tool not registered - verify agent.tools list
  3. Name mismatch - check for typos

Suggested fixes:
  - Review agent instruction to ensure tool usage is clear
  - Verify tool is included in agent.tools list
  - Check for typos in function name`, toolName, joinedTools)
}
```

---

## OnToolErrorCallbacks 错误处理

**位置**: `internal/llminternal/base_flow.go:743-757`

### 方法签名

```go
type OnToolErrorCallback func(ctx tool.Context, tool tool.Tool, args map[string]any, err error) (map[string]any, error)
```

### 执行流程

```go
func (f *Flow) invokeOnToolErrorCallbacks(toolCtx tool.Context, tool tool.Tool, fArgs map[string]any, fErr error) (map[string]any, error) {
    for _, callback := range f.OnToolErrorCallbacks {
        result, err := callback(toolCtx, tool, fArgs, fErr)
        if err != nil {
            return nil, err
        }
        // 返回非 nil 结果/错误，替换原始错误
        if result != nil || err != nil {
            return result, err
        }
    }
    // 无干预，返回原始错误
    return nil, fErr
}
```

### 错误恢复场景

```go
// 重试回调
func retryCallback(ctx tool.Context, tool tool.Tool, args map[string]any, err error) (map[string]any, error) {
    if isRetryable(err) {
        return tool.Run(ctx, args)  // 重试
    }
    return nil, nil  // 不干预
}

// 错误转换
func transformErrorCallback(ctx tool.Context, tool tool.Tool, args map[string]any, err error) (map[string]any, error) {
    return map[string]any{"error": transformError(err)}, nil
}
```

---

## AfterToolCallback 执行

**位置**: `internal/llminternal/base_flow.go:727-741`

### 方法签名

```go
type AfterToolCallback func(ctx tool.Context, tool tool.Tool, args, result map[string]any, err error) (map[string]any, error)
```

### 执行流程

```go
func (f *Flow) invokeAfterToolCallbacks(toolCtx tool.Context, tool toolinternal.FunctionTool, fArgs, fResult map[string]any, fErr error) (map[string]any, error) {
    for _, callback := range f.AfterToolCallbacks {
        result, err := callback(toolCtx, tool, fArgs, fResult, fErr)
        if err != nil {
            return nil, err
        }
        // 返回非 nil 结果/错误，替换原始结果
        if result != nil || err != nil {
            return result, err
        }
    }
    // 无干预，返回原始结果
    return fResult, fErr
}
```

### 回调参数

| 参数 | 类型 | 用途 |
|------|------|------|
| `ctx` | tool.Context | 工具上下文 |
| `tool` | tool.Tool | 工具实例 |
| `args` | map[string]any | 原始参数 |
| `result` | map[string]any | 工具执行结果 |
| `err` | error | 工具执行错误 |

### 使用场景

```go
// 结果日志
func logResultCallback(ctx tool.Context, tool tool.Tool, args, result map[string]any, err error) (map[string]any, error) {
    log.Printf("Tool %s result: %+v, err: %v", tool.Name(), result, err)
    return nil, nil  // 不修改结果
}

// 结果转换
func transformResultCallback(ctx tool.Context, tool tool.Tool, args, result map[string]any, err error) (map[string]any, error) {
    if result != nil {
        result["processed"] = true
    }
    return result, err
}

// 错误处理
func handleErrorCallback(ctx tool.Context, tool tool.Tool, args, result map[string]any, err error) (map[string]any, error) {
    if err != nil {
        return map[string]any{"error": err.Error(), "recovered": true}, nil
    }
    return nil, nil
}
```

---

## FunctionResponse Event 创建

**位置**: `internal/llminternal/base_flow.go:605-623`

### 创建流程

```go
// 创建新事件
ev := session.NewEvent(ctx.InvocationID())

// 设置 LLM 响应内容
ev.LLMResponse = model.LLMResponse{
    Content: &genai.Content{
        Role: "user",
        Parts: []*genai.Part{
            {
                FunctionResponse: &genai.FunctionResponse{
                    ID:       fnCall.ID,      // 函数调用 ID
                    Name:     fnCall.Name,    // 函数名称
                    Response: result,         // 执行结果
                },
            },
        },
    },
}

// 设置事件元数据
ev.Author = ctx.Agent().Name()      // 当前 Agent 名称
ev.Branch = ctx.Branch()            // 分支路径

// 携带状态变更
ev.Actions = *toolCtx.Actions()
```

### Event 结构说明

```go
type Event struct {
    model.LLMResponse
    
    ID           string           // 事件唯一 ID
    Timestamp    time.Time        // 时间戳
    
    InvocationID string           // 调用 ID
    Branch       string           // 分支路径
    Author       string           // 作者（Agent 名称）
    
    Actions      EventActions     // 行为（状态变更等）
    LongRunningToolIDs []string   // 长运行工具 ID
}
```

### FunctionResponse 结构

```go
type FunctionResponse struct {
    ID       string      // 匹配 FunctionCall.ID
    Name     string      // 函数名称
    Response map[string]any  // 执行结果
}
```

---

## mergeParallelFunctionResponseEvents() 合并响应

**位置**: `internal/llminternal/base_flow.go:759-785`

### 方法签名

```go
func mergeParallelFunctionResponseEvents(events []*session.Event) (*session.Event, error)
```

### 合并逻辑

```go
func mergeParallelFunctionResponseEvents(events []*session.Event) (*session.Event, error) {
    switch len(events) {
    case 0:
        return nil, nil
    case 1:
        return events[0], nil
    }
    
    var parts []*genai.Part
    var actions *session.EventActions
    
    // 合并 Parts 和 Actions
    for _, ev := range events {
        if ev == nil || ev.LLMResponse.Content == nil {
            continue
        }
        parts = append(parts, ev.LLMResponse.Content.Parts...)
        actions = mergeEventActions(actions, &ev.Actions)
    }
    
    // 重用第一个事件
    ev := events[0]
    ev.LLMResponse = model.LLMResponse{
        Content: &genai.Content{
            Role:  "user",
            Parts: parts,
        },
    }
    ev.Actions = *actions
    return ev, nil
}
```

### 合并行为

```go
func mergeEventActions(base, other *session.EventActions) *session.EventActions {
    if other == nil { return base }
    if base == nil { return other }
    
    if other.SkipSummarization {
        base.SkipSummarization = true
    }
    if other.TransferToAgent != "" {
        base.TransferToAgent = other.TransferToAgent
    }
    if other.Escalate {
        base.Escalate = true
    }
    if other.StateDelta != nil {
        base.StateDelta = deepMergeMap(base.StateDelta, other.StateDelta)
    }
    if other.RequestedToolConfirmations != nil {
        // 合并确认请求
    }
    return base
}
```

---

## Telemetry 追踪

### 工具执行 Span 层级

```
Invocation (Agent.Run)
    │
    └─ GenerateContent (LLM 调用)
        │
        └─ execute_tool (handleFunctionCalls)
            │
            ├─ execute_tool (merged) - 多个工具时
            │   │
            │   └─ execute_tool (单个工具 1)
            │       └─ execute_tool (单个工具 2)
            │           └─ ...
```

### 关键追踪点

**StartExecuteToolSpan**
```go
sctx, span := telemetry.StartExecuteToolSpan(ctx, telemetry.StartExecuteToolSpanParams{
    ToolName: fnCall.Name,
    Args:     fnCall.Args,
})
defer span.End()
```

**TraceToolResult**
```go
telemetry.TraceToolResult(span, telemetry.TraceToolResultParams{
    Description:   traceTool.Description(),
    ResponseEvent: ev,
    Error:         toolErr,
})
```

---

## 工具上下文 (ToolContext)

**位置**: `toolinternal.NewToolContext`

### 创建

```go
toolCtx := toolinternal.NewToolContext(
    toolCallCtx,                          // 上下文
    fnCall.ID,                            // 函数调用 ID
    &session.EventActions{StateDelta: make(map[string]any)},  // 状态变更
    confirmation                          // 工具确认
)
```

### 作用

- 传递调用 ID 用于匹配
- 提供状态变更能力
- 支持工具确认机制

---

## 回调类型总结

### BeforeToolCallback

```go
func(ctx tool.Context, tool tool.Tool, args map[string]any) (map[string]any, error)
```
- 执行时机：工具执行前
- 用途：参数验证、修改、缓存
- 提前返回：返回非 nil 结果跳过工具执行

### AfterToolCallback

```go
func(ctx tool.Context, tool tool.Tool, args, result map[string]any, err error) (map[string]any, error)
```
- 执行时机：工具执行后（无论成功/失败）
- 用途：结果日志、转换、处理
- 始终执行

### OnToolErrorCallback

```go
func(ctx tool.Context, tool tool.Tool, args map[string]any, err error) (map[string]any, error)
```
- 执行时机：工具执行返回错误时
- 用途：错误恢复、重试、转换
- 仅错误时执行

---

## 关键设计决策

### 1. 回调执行顺序

```
插件 BeforeToolCallback → 用户 BeforeToolCallbacks → tool.Run() → 错误处理 → 插件 AfterToolCallback → 用户 AfterToolCallbacks
```

**设计意图**:
- 插件优先执行，可全局拦截
- 用户回调可定制特定行为
- 工具执行在中间
- After 回调始终执行（即使错误）

### 2. 错误处理策略

```go
if err != nil {
    // 1. 尝试错误回调恢复
    errorResponse, cbErr = f.invokeOnToolErrorCallbacks(...)
    // 2. 如无恢复，返回原始错误
    // 3. After 回调仍会执行
}
```

### 3. 状态变更传递

```go
// 创建上下文时初始化状态变更
toolCtx := toolinternal.NewToolContext(..., &session.EventActions{StateDelta: make(map[string]any)}, ...)

// 回调和工具可修改状态
toolCtx.Actions().StateDelta["key"] = value

// 最终携带到事件
ev.Actions = *toolCtx.Actions()
```

### 4. 并行工具合并

- 多个工具调用创建合并 Span
- 响应 Parts 简单拼接
- Actions 深度合并
- 保持事件语义统一

---

## 总结

ADK Go 的工具调用机制具有以下特点：

1. **多层回调**: BeforeTool → tool.Run() → OnError → AfterTool
2. **完整错误处理**: 工具未找到、类型错误、执行错误全面覆盖
3. **状态传递**: StateDelta 贯穿工具执行全程
4. **并行支持**: 多个工具调用自动合并
5. **Telemetry**: 完整的 Span 追踪层级
6. **提前返回**: Before 回调可跳过工具执行
7. **结果替换**: OnError/After 回调可修改结果

设计优势：
- 清晰的回调职责划分
- 灵活的错误恢复机制
- 完整的可观测性支持
- 统一的事件语义
