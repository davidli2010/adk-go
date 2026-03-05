# ADK Go Flow 执行引擎详解

## 概述

Flow 是 ADK Go 的核心执行引擎，负责管理从 LLM 调用到工具执行的完整循环。本文档深入分析 `internal/llminternal/base_flow.go` 的实现细节。

---

## Flow 结构定义

**位置**: `internal/llminternal/base_flow.go:58-70`

```go
type Flow struct {
    Model model.LLM
    
    Tools                 []tool.Tool
    RequestProcessors     []func(ctx agent.InvocationContext, req *model.LLMRequest, f *Flow) iter.Seq2[*session.Event, error]
    ResponseProcessors    []func(ctx agent.InvocationContext, req *model.LLMRequest, resp *model.LLMResponse) error
    BeforeModelCallbacks  []BeforeModelCallback
    AfterModelCallbacks   []AfterModelCallback
    OnModelErrorCallbacks []OnModelErrorCallback
    BeforeToolCallbacks   []BeforeToolCallback
    AfterToolCallbacks    []AfterToolCallback
    OnToolErrorCallbacks  []OnToolErrorCallback
}
```

**核心字段**:
- `Model`: LLM 模型实例
- `Tools`: 可用工具列表
- `RequestProcessors`: 请求处理器链（11 个预定义）
- `ResponseProcessors`: 响应处理器链（2 个预定义）
- 回调链：Model/Tool 的前后置回调和错误处理

---

## Flow.Run() 主循环

**位置**: `internal/llminternal/base_flow.go:97-122`

### 方法签名

```go
func (f *Flow) Run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error]
```

### 循环逻辑

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
                    return  // 消费者提前终止
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
            // 否则继续循环（处理工具调用等）
        }
    }
}
```

### 循环流程图

```
Flow.Run()
    │
    ├─ for { 无限循环 }
    │   │
    │   ├─ runOneStep() → 事件流
    │   │   ├─ preprocess()
    │   │   ├─ callLLM()
    │   │   ├─ postprocess()
    │   │   └─ handleFunctionCalls()
    │   │
    │   ├─ yield 每个事件
    │   │
    │   └─ 检查退出条件
    │       ├─ lastEvent == nil → 退出
    │       ├─ IsFinalResponse() → 退出
    │       ├─ Partial → 错误退出
    │       └─ 否则 → 继续循环
    │
    └─ 返回最终结果
```

### IsFinalResponse() 判断

**位置**: `session/session.go:120-129`

```go
func (e *Event) IsFinalResponse() bool {
    if (e.Actions.SkipSummarization) || len(e.LongRunningToolIDs) > 0 {
        return true
    }
    return !hasFunctionCalls(&e.LLMResponse) && 
           !hasFunctionResponses(&e.LLMResponse) && 
           !e.LLMResponse.Partial && 
           !hasTrailingCodeExecutionResult(&e.LLMResponse)
}
```

**退出条件**（返回 true）:
- `SkipSummarization` 为 true
- 存在长运行工具调用
- 无函数调用
- 无函数响应
- 非部分响应
- 无尾部代码执行结果

---

## runOneStep() 步骤执行

**位置**: `internal/llminternal/base_flow.go:125-243`

### 方法签名

```go
func (f *Flow) runOneStep(ctx agent.InvocationContext) iter.Seq2[*session.Event, error]
```

### 完整执行流程

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
        if ctx.Ended() { return }  // 提前结束检查
        
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
            tools := make(map[string]tool.Tool)  // 转换工具字典
            modelResponseEvent := f.finalizeModelResponseEvent(ctx, resp, tools, stateDelta)
            if !yield(modelResponseEvent, nil) { return }
            
            // 7. 处理函数调用
            ev, err := f.handleFunctionCalls(ctx, tools, resp.LLMResponse, nil)
            if err != nil {
                yield(nil, err)
                return
            }
            if ev == nil { continue }  // 无函数调用
            
            // 8. 工具确认事件
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

---

## preprocess() 请求预处理

**位置**: `internal/llminternal/base_flow.go:245-266`

### 方法签名

```go
func (f *Flow) preprocess(ctx agent.InvocationContext, req *model.LLMRequest) iter.Seq2[*session.Event, error]
```

### 处理逻辑

```go
func (f *Flow) preprocess(ctx agent.InvocationContext, req *model.LLMRequest) iter.Seq2[*session.Event, error] {
    return func(yield func(*session.Event, error) bool) {
        // 1. 执行 RequestProcessors 链
        for _, processor := range f.RequestProcessors {
            for ev, err := range processor(ctx, req, f) {
                if err != nil {
                    yield(nil, err)
                    return
                }
                if ev != nil {
                    yield(ev, nil)
                }
            }
        }
        
        // 2. 工具预处理
        if f.Tools != nil {
            if err := toolPreprocess(ctx, req, f.Tools); err != nil {
                yield(nil, err)
            }
        }
    }
}
```

### RequestProcessors 链

**位置**: `internal/llminternal/base_flow.go:73-90`

```go
var DefaultRequestProcessors = []func(...){
    basicRequestProcessor,           // 1. 基础请求处理
    toolProcessor,                   // 2. 工具处理
    authPreprocessor,                // 3. 认证预处理
    RequestConfirmationRequestProcessor,  // 4. 请求确认
    instructionsRequestProcessor,    // 5. 指令处理
    identityRequestProcessor,        // 6. 身份处理
    ContentsRequestProcessor,        // 7. 内容处理（对话历史）
    nlPlanningRequestProcessor,      // 8. NL Planning
    codeExecutionRequestProcessor,   // 9. 代码执行
    outputSchemaRequestProcessor,    // 10. 输出模式
    AgentTransferRequestProcessor,   // 11. 代理转移
    removeDisplayNameIfExists,       // 12. 移除显示名称
}
```

### toolPreprocess() 工具预处理

**位置**: `internal/llminternal/base_flow.go:268-284`

```go
func toolPreprocess(ctx agent.InvocationContext, req *model.LLMRequest, tools []tool.Tool) error {
    for _, t := range tools {
        requestProcessor, ok := t.(toolinternal.RequestProcessor)
        if !ok {
            return fmt.Errorf("tool %q does not implement RequestProcessor() method", t.Name())
        }
        toolCtx := toolinternal.NewToolContext(ctx, "", &session.EventActions{}, nil)
        if err := requestProcessor.ProcessRequest(toolCtx, req); err != nil {
            return err
        }
    }
    return nil
}
```

**用途**: 调用每个工具的 `ProcessRequest()` 方法，允许工具修改请求。

---

## callLLM() LLM 调用

**位置**: `internal/llminternal/base_flow.go:290-367`

### 方法签名

```go
func (f *Flow) callLLM(ctx agent.InvocationContext, req *model.LLMRequest, stateDelta map[string]any) iter.Seq2[*responseWithEventID, error]
```

### 调用流程

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
                return  // 提前返回，跳过 LLM 调用
            }
        }
        
        // 2. 用户 BeforeModelCallbacks
        for _, callback := range f.BeforeModelCallbacks {
            cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
            callbackResponse, callbackErr := callback(cctx, req)
            if callbackResponse != nil || callbackErr != nil {
                yield(newResponseWithEventID(callbackResponse), callbackErr)
                return  // 提前返回，跳过 LLM 调用
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
                // 使用回调响应替换
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
            
            if !yield(resp, nil) {
                return
            }
        }
    }
}
```

### 回调执行顺序

```
1. PluginManager.RunBeforeModelCallback()
2. BeforeModelCallbacks (依次执行)
3. generateContent() - 实际 LLM 调用
4. OnModelErrorCallbacks (错误时)
5. AfterModelCallbacks (依次执行)
```

---

## generateContent() 实际调用

**位置**: `internal/llminternal/base_flow.go:377-422`

### 方法签名

```go
func generateContent(ctx agent.InvocationContext, m model.LLM, req *model.LLMRequest, useStream bool) iter.Seq2[*responseWithEventID, error]
```

### 调用逻辑

```go
func generateContent(ctx agent.InvocationContext, m model.LLM, req *model.LLMRequest, useStream bool) iter.Seq2[*responseWithEventID, error] {
    return func(yield func(*responseWithEventID, error) bool) {
        // 1. 启动追踪 Span
        spanCtx, span := telemetry.StartGenerateContentSpan(ctx, telemetry.StartGenerateContentSpanParams{
            ModelName:    m.Name(),
            InvocationID: ctx.InvocationID(),
        })
        ctx = ctx.WithContext(spanCtx)
        backend := googlellm.GetGoogleLLMVariant(m)
        
        // 2. 记录请求日志
        telemetry.LogRequest(ctx, req, backend)
        
        var lastResponse responseWithEventID
        var lastErr error
        spanEnded := false
        
        // 3. 结束 Span 的辅助函数
        endSpanAndTrackResult := func() {
            if spanEnded { return }
            telemetry.TraceGenerateContentResult(span, telemetry.TraceGenerateContentResultParams{
                Response: lastResponse.LLMResponse,
                EventID:  lastResponse.eventID,
                Error:    lastErr,
            })
            span.End()
            spanEnded = true
        }
        defer endSpanAndTrackResult()
        
        // 4. 调用 LLM.GenerateContent()
        for resp, err := range m.GenerateContent(ctx, req, useStream) {
            response := newResponseWithEventID(resp)
            lastResponse = *response
            lastErr = err
            
            // 5. 完成 Span
            if err != nil {
                endSpanAndTrackResult()
            } else if !resp.Partial {
                telemetry.LogResponse(ctx, resp, backend)
                endSpanAndTrackResult()
            }
            
            if !yield(response, err) {
                return
            }
        }
    }
}
```

**特点**:
- 完整的 Telemetry 追踪
- 支持流式和非流式模式
- 自动记录请求/响应日志
- Span 在最终响应时立即结束

---

## postprocess() 响应后处理

**位置**: `internal/llminternal/base_flow.go:469-477`

### 方法签名

```go
func (f *Flow) postprocess(ctx agent.InvocationContext, req *model.LLMRequest, resp *responseWithEventID) error
```

### 处理逻辑

```go
func (f *Flow) postprocess(ctx agent.InvocationContext, req *model.LLMRequest, resp *responseWithEventID) error {
    // 执行 ResponseProcessors 链
    for _, processor := range f.ResponseProcessors {
        if err := processor(ctx, req, resp.LLMResponse); err != nil {
            return err
        }
    }
    return nil
}
```

### ResponseProcessors 链

**位置**: `internal/llminternal/base_flow.go:91-94`

```go
var DefaultResponseProcessors = []func(...){
    nlPlanningResponseProcessor,     // 1. NL Planning 后处理
    codeExecutionResponseProcessor,  // 2. 代码执行后处理
}
```

---

## finalizeModelResponseEvent() 构建事件

**位置**: `internal/llminternal/base_flow.go:493-509`

### 方法签名

```go
func (f *Flow) finalizeModelResponseEvent(ctx agent.InvocationContext, resp *responseWithEventID, tools map[string]tool.Tool, stateDelta map[string]any) *session.Event
```

### 构建逻辑

```go
func (f *Flow) finalizeModelResponseEvent(ctx agent.InvocationContext, resp *responseWithEventID, tools map[string]tool.Tool, stateDelta map[string]any) *session.Event {
    // 1. 生成函数调用 ID（如缺失）
    utils.PopulateClientFunctionCallID(resp.Content)
    
    // 2. 创建事件
    ev := session.NewEvent(ctx.InvocationID())
    ev.ID = resp.eventID
    ev.Author = ctx.Agent().Name()
    ev.Branch = ctx.Branch()
    ev.LLMResponse = *resp.LLMResponse
    ev.Actions.StateDelta = stateDelta
    
    // 3. 标记长运行工具
    ev.LongRunningToolIDs = findLongRunningFunctionCallIDs(resp.Content, tools)
    
    return ev
}
```

### findLongRunningFunctionCallIDs()

**位置**: `internal/llminternal/base_flow.go:512-525`

```go
func findLongRunningFunctionCallIDs(c *genai.Content, tools map[string]tool.Tool) []string {
    set := make(map[string]struct{})
    for _, fc := range utils.FunctionCalls(c) {
        if tool, ok := tools[fc.Name]; ok && fc.ID != "" && tool.IsLongRunning() {
            set[fc.ID] = struct{}{}
        }
    }
    return slices.Collect(maps.Keys(set))
}
```

**用途**: 识别长运行工具调用，影响 `IsFinalResponse()` 判断。

---

## handleFunctionCalls() 工具调用处理

**位置**: `internal/llminternal/base_flow.go:559-652`

### 方法签名

```go
func (f *Flow) handleFunctionCalls(ctx agent.InvocationContext, toolsDict map[string]tool.Tool, resp *model.LLMResponse, toolConfirmations map[string]*toolconfirmation.ToolConfirmation) (mergedEvent *session.Event, err error)
```

### 完整流程

```go
func (f *Flow) handleFunctionCalls(...) (mergedEvent *session.Event, err error) {
    var fnResponseEvents []*session.Event
    fnCalls := utils.FunctionCalls(resp.Content)
    toolNames := slices.Collect(maps.Keys(toolsDict))
    var result map[string]any
    
    // 1. 创建合并 Span（多个工具调用时）
    if len(fnCalls) > 1 {
        mergedCtx, mergedToolCallSpan := telemetry.StartTrace(ctx, "execute_tool (merged)")
        ctx = ctx.WithContext(mergedCtx)
        defer func() {
            telemetry.TraceMergedToolCallsResult(mergedToolCallSpan, mergedEvent, err)
            mergedToolCallSpan.End()
        }()
    }
    
    // 2. 遍历每个函数调用
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
            
            // 3. 查找工具
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
                // 4. 调用工具
                result = f.callTool(toolCtx, funcTool, fnCall.Args)
            }
            
            // 5. 创建函数响应事件
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
            
            // 6. 记录工具执行结果
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

### 工具调用流程图

```
handleFunctionCalls()
    │
    ├─ 遍历 FunctionCalls
    │   │
    │   ├─ 启动工具执行 Span
    │   │
    │   ├─ 查找工具
    │   │   ├─ 未找到 → runOnToolErrorCallbacks()
    │   │   └─ 找到 → callTool()
    │   │
    │   ├─ callTool()
    │   │   ├─ BeforeToolCallbacks
    │   │   ├─ tool.Run()
    │   │   ├─ OnToolErrorCallbacks (错误时)
    │   │   └─ AfterToolCallbacks
    │   │
    │   ├─ 创建 FunctionResponse 事件
    │   │
    │   └─ 记录工具执行结果
    │
    └─ mergeParallelFunctionResponseEvents()
```

---

## callTool() 工具实际执行

**位置**: `internal/llminternal/base_flow.go:665-710`

### 方法签名

```go
func (f *Flow) callTool(toolCtx tool.Context, tool toolinternal.FunctionTool, fArgs map[string]any) map[string]any
```

### 执行流程

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

### 回调执行顺序

```
1. PluginManager.RunBeforeToolCallback()
2. BeforeToolCallbacks (依次执行)
3. tool.Run() - 实际工具执行
4. OnToolErrorCallbacks (错误时)
5. AfterToolCallbacks (依次执行)
```

---

## mergeParallelFunctionResponseEvents() 合并并行响应

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
    for _, ev := range events {
        if ev == nil || ev.LLMResponse.Content == nil {
            continue
        }
        parts = append(parts, ev.LLMResponse.Content.Parts...)
        actions = mergeEventActions(actions, &ev.Actions)
    }
    
    // 重用 events[0]
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

### mergeEventActions() 合并行为

**位置**: `internal/llminternal/base_flow.go:787-815`

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
        if base.RequestedToolConfirmations == nil {
            base.RequestedToolConfirmations = make(map[string]toolconfirmation.ToolConfirmation)
        }
        maps.Copy(base.RequestedToolConfirmations, other.RequestedToolConfirmations)
    }
    return base
}
```

### deepMergeMap() 深度合并

**位置**: `internal/llminternal/base_flow.go:817-831`

```go
func deepMergeMap(dst, src map[string]any) map[string]any {
    if dst == nil {
        dst = make(map[string]any)
    }
    for key, value := range src {
        if srcMap, ok := value.(map[string]any); ok {
            if dstMap, ok := dst[key].(map[string]any); ok {
                dst[key] = deepMergeMap(dstMap, srcMap)
                continue
            }
        }
        dst[key] = value
    }
    return dst
}
```

**特点**: 递归合并嵌套 map，支持深层状态合并。

---

## agentToRun() 查找目标 Agent

**位置**: `internal/llminternal/base_flow.go:479-490`

### 方法签名

```go
func (f *Flow) agentToRun(ctx agent.InvocationContext, agentName string) agent.Agent
```

### 查找逻辑

```go
func (f *Flow) agentToRun(ctx agent.InvocationContext, agentName string) agent.Agent {
    parents := parentmap.FromContext(ctx)
    // 获取可转移的 Agent 列表
    agents := transferTargets(ctx.Agent(), parents[ctx.Agent().Name()])
    
    // 查找目标 Agent
    for _, agent := range agents {
        if agent.Name() == agentName {
            return agent
        }
    }
    return nil
}
```

**用途**: 在代理转移时查找目标 Agent。

---

## 关键设计决策

### 1. 循环执行模式

**设计**: `for { runOneStep() }` 无限循环

**原因**:
- 支持工具调用→LLM 总结→更多工具调用的链式执行
- 通过 `IsFinalResponse()` 判断退出
- 自动处理多轮交互

### 2. 处理器链模式

**设计**: RequestProcessors (12 个) + ResponseProcessors (2 个)

**优势**:
- 模块化设计，每个处理器职责单一
- 清晰的执行顺序
- 易于扩展和定制

### 3. 回调优先级

**顺序**:
1. 插件回调（全局）
2. 用户回调（Agent 级别）
3. 实际执行

**优势**:
- 插件可全局拦截
- 用户可定制特定 Agent
- 层次清晰

### 4. 并行工具调用

**设计**: 多个工具调用创建合并 Span

**优势**:
- 支持并发执行（未来优化）
- 统一的追踪和日志
- 合并响应减少事件数量

### 5. 错误恢复机制

**机制**: OnModelErrorCallbacks + OnToolErrorCallbacks

**用途**:
- 错误降级处理
- 重试逻辑
- 自定义错误响应

### 6. 状态传递

**机制**: stateDelta 贯穿整个调用链

**流程**:
```
callLLM(stateDelta)
    ↓
finalizeModelResponseEvent(stateDelta)
    ↓
ev.Actions.StateDelta = stateDelta
    ↓
持久化到 Session
```

---

## Telemetry 集成

### Span 层级

```
Invocation (Agent.Run)
    │
    └─ GenerateContent (generateContent)
        │
        └─ execute_tool (handleFunctionCalls)
            │
            ├─ execute_tool (merged) - 多个工具时
            │   │
            │   └─ execute_tool (单个工具)
```

### 关键追踪点

1. **StartInvokeAgentSpan**: Agent 调用开始
2. **StartGenerateContentSpan**: LLM 调用开始
3. **StartExecuteToolSpan**: 工具执行开始
4. **TraceAgentResult**: Agent 结果
5. **TraceGenerateContentResult**: LLM 响应结果
6. **TraceToolResult**: 工具执行结果
7. **TraceMergedToolCallsResult**: 合并工具调用结果

---

## 总结

Flow 执行引擎是 ADK Go 的核心，具有以下特点：

1. **循环执行**: 多步循环，自动处理工具调用链
2. **处理器链**: 12 个 RequestProcessors + 2 个 ResponseProcessors
3. **回调机制**: 多层回调（插件/用户），灵活扩展
4. **工具调用**: 完整的 Before/After/OnError 回调链
5. **并行处理**: 支持多工具并行调用和响应合并
6. **Telemetry**: 完整的追踪和日志记录
7. **错误恢复**: OnError 回调支持降级处理

设计优势：
- 清晰的职责划分（preprocess/callLLM/postprocess/handleFunctionCalls）
- 模块化设计，易于扩展
- 完整的错误处理和恢复机制
- 自动处理复杂的多轮交互和代理转移
