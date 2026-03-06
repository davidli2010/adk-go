# ADK Go LLM 调用详解

## 概述

本文档深入分析 ADK Go 中 LLM 调用的完整流程，包括 BeforeModelCallbacks 执行时机、generateContent 调用、AfterModelCallbacks 执行以及流式响应处理机制。

---

## LLM 调用流程概览

**位置**: `internal/llminternal/base_flow.go:290-367`

### 完整调用链

```
Flow.callLLM()
    │
    ├─ 1. PluginManager.RunBeforeModelCallback()
    │
    ├─ 2. BeforeModelCallbacks (依次执行)
    │
    ├─ 3. generateContent() - 实际 LLM 调用
    │   ├─ StartGenerateContentSpan
    │   ├─ LogRequest
    │   ├─ m.GenerateContent() 流式迭代
    │   ├─ LogResponse (仅最终响应)
    │   └─ TraceGenerateContentResult
    │
    ├─ 4. OnModelErrorCallbacks (错误时)
    │
    ├─ 5. utils.PopulateClientFunctionCallID()
    │
    └─ 6. AfterModelCallbacks (依次执行)
```

---

## BeforeModelCallbacks 执行时机

**位置**: `internal/llminternal/base_flow.go:290-310`

### 方法签名

```go
type BeforeModelCallback func(ctx agent.CallbackContext, llmRequest *model.LLMRequest) (*model.LLMResponse, error)
```

### 执行流程

```go
func (f *Flow) callLLM(ctx agent.InvocationContext, req *model.LLMRequest, stateDelta map[string]any) iter.Seq2[*responseWithEventID, error] {
    return func(yield func(*responseWithEventID, error) bool) {
        pluginManager := pluginManagerFromContext(ctx)
        
        // 1. 插件 BeforeModelCallback (优先执行)
        if pluginManager != nil {
            cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
            callbackResponse, callbackErr := pluginManager.RunBeforeModelCallback(cctx, req)
            if callbackResponse != nil || callbackErr != nil {
                yield(newResponseWithEventID(callbackResponse), callbackErr)
                return  // 提前返回，跳过 LLM 调用
            }
        }
        
        // 2. 用户 BeforeModelCallbacks (依次执行)
        for _, callback := range f.BeforeModelCallbacks {
            cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
            callbackResponse, callbackErr := callback(cctx, req)
            
            if callbackResponse != nil || callbackErr != nil {
                yield(newResponseWithEventID(callbackResponse), callbackErr)
                return  // 提前返回，跳过 LLM 调用
            }
        }
        
        // 3. 继续执行实际 LLM 调用...
    }
}
```

### 回调上下文

**位置**: `icontext.NewCallbackContextWithDelta`

```go
cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
```

**包含内容**:
- `ctx`: 当前 InvocationContext
- `stateDelta`: 状态变更映射（可读写）
- 访问 Session/Agent/Artifacts/Memory 等

### 提前返回机制

```go
if callbackResponse != nil || callbackErr != nil {
    yield(newResponseWithEventID(callbackResponse), callbackErr)
    return  // 关键：跳过实际 LLM 调用
}
```

**触发条件**:
- 返回非 nil `LLMResponse` → 使用回调响应
- 返回非 nil `error` → 使用回调错误

**用途**:
- **缓存**: 返回缓存的响应，跳过 LLM 调用
- **拦截**: 检查/修改请求参数
- **降级**: 返回预设响应（如限流时）
- **日志**: 记录请求详情

### 使用示例

```go
// 缓存回调
func cacheCallback(ctx agent.CallbackContext, req *model.LLMRequest) (*model.LLMResponse, error) {
    cacheKey := computeCacheKey(req)
    if cached, ok := cache.Get(cacheKey); ok {
        return cached, nil  // 返回缓存，跳过 LLM
    }
    return nil, nil  // 继续 LLM 调用
}

// 请求日志回调
func logCallback(ctx agent.CallbackContext, req *model.LLMRequest) (*model.LLMResponse, error) {
    log.Printf("LLM Request: %+v", req)
    return nil, nil  // 不干预，继续执行
}
```

---

## generateContent 调用流程

**位置**: `internal/llminternal/base_flow.go:377-422`

### 方法签名

```go
func generateContent(ctx agent.InvocationContext, m model.LLM, req *model.LLMRequest, useStream bool) iter.Seq2[*responseWithEventID, error]
```

### 完整实现

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
            if spanEnded {
                return  // 避免重复结束
            }
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
                endSpanAndTrackResult()  // 错误时立即结束
            } else if !resp.Partial {
                // 仅记录最终响应日志
                telemetry.LogResponse(ctx, resp, backend)
                endSpanAndTrackResult()
            }
            
            if !yield(response, err) {
                return  // 消费者提前终止
            }
        }
    }
}
```

### Telemetry 追踪

#### Span 创建

**位置**: `telemetry.StartGenerateContentSpan`

```go
spanCtx, span := telemetry.StartGenerateContentSpan(ctx, telemetry.StartGenerateContentSpanParams{
    ModelName:    m.Name(),
    InvocationID: ctx.InvocationID(),
})
```

**追踪内容**:
- Model 名称
- InvocationID (调用唯一标识)
- 开始时间戳

#### 请求日志

**位置**: `telemetry.LogRequest`

```go
telemetry.LogRequest(ctx, req, backend)
```

**记录内容**:
- 请求完整内容
- Model 配置
- Backend 类型

#### 响应日志

**位置**: `telemetry.LogResponse`

```go
if !resp.Partial {
    telemetry.LogResponse(ctx, resp, backend)
}
```

**特点**: 仅记录最终响应，不记录 Partial 响应

#### 结果追踪

**位置**: `telemetry.TraceGenerateContentResult`

```go
telemetry.TraceGenerateContentResult(span, telemetry.TraceGenerateContentResultParams{
    Response: lastResponse.LLMResponse,
    EventID:  lastResponse.eventID,
    Error:    lastErr,
})
```

**记录内容**:
- 完整响应内容
- Event ID
- 错误信息（如有）
- Token 使用量（如可用）

### Span 结束时机

```go
if err != nil {
    endSpanAndTrackResult()  // 错误时立即结束
} else if !resp.Partial {
    endSpanAndTrackResult()  // 最终响应时结束
}
```

**设计意图**:
- 错误时立即结束，记录错误信息
- Partial 响应不结束 Span，避免过早关闭
- 最终响应时结束，记录完整结果
- `defer` 保证异常情况下也能结束

---

## 流式响应处理

**位置**: `internal/llminternal/base_flow.go:318-366`

### 流式模式检测

```go
useStream := runconfig.FromContext(ctx).StreamingMode == runconfig.StreamingModeSSE
```

**配置选项**:
- `StreamingModeSSE`: 启用流式（Server-Sent Events）
- 其他：非流式模式

### 迭代器类型

```go
iter.Seq2[*responseWithEventID, error]
```

**Go 1.22+ 泛型迭代器**:
- 产出 `(response, error)` 对
- 支持 `for resp, err := range iterator` 语法
- 消费者可通过 `yield` 返回值控制迭代

### 流式处理流程

```go
for resp, err := range generateContent(ctx, f.Model, req, useStream) {
    // 1. 错误处理
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
    
    // 2. 填充函数调用 ID
    utils.PopulateClientFunctionCallID(resp.Content)
    
    // 3. AfterModelCallbacks
    callbackResp, callbackErr := f.runAfterModelCallbacks(ctx, resp.LLMResponse, stateDelta, err)
    if callbackErr != nil {
        yield(nil, callbackErr)
        return
    }
    
    if callbackResp != nil {
        resp = &responseWithEventID{LLMResponse: callbackResp, eventID: resp.eventID}
        if !yield(resp, nil) {
            return
        }
        continue
    }
    
    // 4. 错误检查
    if err != nil {
        yield(nil, err)
        return
    }
    
    // 5. 产出响应
    if !yield(resp, nil) {
        return
    }
}
```

### Partial 响应处理

**特点**:
- Partial 响应是流式中间片段
- 不触发日志记录
- Span 不结束
- 继续迭代下一个响应

```go
} else if !resp.Partial {
    // 仅记录最终响应日志
    telemetry.LogResponse(ctx, resp, backend)
    endSpanAndTrackResult()
}
```

### 消费者提前终止

```go
if !yield(resp, nil) {
    return  // 消费者返回 false，停止迭代
}
```

**场景**:
- 客户端断开连接
- 达到 token 限制
- 用户取消请求

---

## AfterModelCallbacks 执行

**位置**: `internal/llminternal/base_flow.go:425-445`

### 方法签名

```go
type AfterModelCallback func(ctx agent.CallbackContext, llmResponse *model.LLMResponse, llmResponseError error) (*model.LLMResponse, error)
```

### 执行流程

```go
func (f *Flow) runAfterModelCallbacks(ctx agent.InvocationContext, llmResp *model.LLMResponse, stateDelta map[string]any, llmErr error) (*model.LLMResponse, error) {
    pluginManager := pluginManagerFromContext(ctx)
    
    // 1. 插件 AfterModelCallback (优先执行)
    if pluginManager != nil {
        cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
        callbackResponse, callbackErr := pluginManager.RunAfterModelCallback(cctx, llmResp, llmErr)
        if callbackResponse != nil || callbackErr != nil {
            return callbackResponse, callbackErr
        }
    }
    
    // 2. 用户 AfterModelCallbacks (依次执行)
    for _, callback := range f.AfterModelCallbacks {
        cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
        callbackResponse, callbackErr := callback(cctx, llmResp, llmErr)
        
        if callbackResponse != nil || callbackErr != nil {
            return callbackResponse, callbackErr
        }
    }
    
    return nil, nil  // 无干预，返回原始响应
}
```

### 回调参数

| 参数 | 类型 | 用途 |
|------|------|------|
| `ctx` | CallbackContext | 回调上下文 |
| `llmResponse` | *model.LLMResponse | LLM 响应 |
| `llmResponseError` | error | LLM 错误（如有） |

### 返回行为

```go
if callbackResponse != nil || callbackErr != nil {
    return callbackResponse, callbackErr  // 替换原始响应/错误
}
```

**用途**:
- **修改响应**: 后处理响应内容
- **错误处理**: 转换或包装错误
- **日志记录**: 记录响应详情
- **Token 统计**: 收集使用量指标

### 使用示例

```go
// Token 统计回调
func tokenStatsCallback(ctx agent.CallbackContext, resp *model.LLMResponse, err error) (*model.LLMResponse, error) {
    if resp != nil && resp.UsageMetadata != nil {
        metrics.RecordTokenUsage(resp.UsageMetadata)
    }
    return nil, nil  // 不修改响应
}

// 响应过滤回调
func filterCallback(ctx agent.CallbackContext, resp *model.LLMResponse, err error) (*model.LLMResponse, error) {
    if resp != nil && resp.Content != nil {
        // 过滤敏感内容
        resp.Content = filterSensitiveContent(resp.Content)
    }
    return resp, nil  // 返回修改后的响应
}
```

---

## OnModelErrorCallbacks 错误处理

**位置**: `internal/llminternal/base_flow.go:447-467`

### 方法签名

```go
type OnModelErrorCallback func(ctx agent.CallbackContext, llmRequest *model.LLMRequest, llmResponseError error) (*model.LLMResponse, error)
```

### 执行流程

```go
func (f *Flow) runOnModelErrorCallbacks(ctx agent.InvocationContext, llmReq *model.LLMRequest, stateDelta map[string]any, llmErr error) (*model.LLMResponse, error) {
    pluginManager := pluginManagerFromContext(ctx)
    
    // 1. 插件 OnModelErrorCallback (优先执行)
    if pluginManager != nil {
        cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
        callbackResponse, callbackErr := pluginManager.RunOnModelErrorCallback(cctx, llmReq, llmErr)
        if callbackResponse != nil || callbackErr != nil {
            return callbackResponse, callbackErr
        }
    }
    
    // 2. 用户 OnModelErrorCallbacks (依次执行)
    for _, callback := range f.OnModelErrorCallbacks {
        cctx := icontext.NewCallbackContextWithDelta(ctx, stateDelta)
        callbackResponse, callbackErr := callback(cctx, llmReq, llmErr)
        
        if callbackResponse != nil || callbackErr != nil {
            return callbackResponse, callbackErr
        }
    }
    
    return nil, nil  // 无干预，返回原始错误
}
```

### 回调参数

| 参数 | 类型 | 用途 |
|------|------|------|
| `ctx` | CallbackContext | 回调上下文 |
| `llmRequest` | *model.LLMRequest | 原始请求 |
| `llmResponseError` | error | LLM 错误 |

### 错误恢复场景

```go
// 重试回调
func retryCallback(ctx agent.CallbackContext, req *model.LLMRequest, err error) (*model.LLMResponse, error) {
    if isRetryableError(err) {
        // 执行重试逻辑
        return retryLLMCall(ctx, req)
    }
    return nil, nil  // 不干预
}

// 降级回调
func fallbackCallback(ctx agent.CallbackContext, req *model.LLMRequest, err error) (*model.LLMResponse, error) {
    // 返回预设的降级响应
    return &model.LLMResponse{
        Content: &genai.Content{
            Parts: []*genai.Part{{Text: "服务暂时不可用，请稍后重试"}},
        },
    }, nil
}
```

---

## utils.PopulateClientFunctionCallID()

**位置**: `internal/utils/` (函数调用 ID 填充)

### 用途

```go
utils.PopulateClientFunctionCallID(resp.Content)
```

**目的**: 为函数调用生成客户端 ID（如模型未提供）

**原因**:
- GenAI API 中函数调用 ID 是可选的
- 某些模型不使用该字段
- 后续处理需要唯一 ID 匹配调用和响应

### 实现逻辑（推测）

```go
func PopulateClientFunctionCallID(content *genai.Content) {
    if content == nil {
        return
    }
    for _, part := range content.Parts {
        if part.FunctionCall != nil && part.FunctionCall.ID == "" {
            part.FunctionCall.ID = uuid.New().String()
        }
    }
}
```

---

## 回调执行顺序总结

### 完整调用链

```
callLLM()
    │
    ├─ 1. PluginManager.RunBeforeModelCallback()
    │   └─ 可提前返回，跳过 LLM
    │
    ├─ 2. BeforeModelCallbacks (依次)
    │   └─ 任一返回非 nil，跳过后续和 LLM
    │
    ├─ 3. generateContent()
    │   ├─ StartGenerateContentSpan
    │   ├─ LogRequest
    │   ├─ for resp, err := range m.GenerateContent()
    │   │   ├─ 错误 → OnModelErrorCallbacks
    │   │   ├─ PopulateClientFunctionCallID
    │   │   └─ AfterModelCallbacks
    │   ├─ LogResponse (仅最终响应)
    │   └─ TraceGenerateContentResult
    │
    └─ yield 每个响应
```

### 优先级层次

```
插件回调 (全局)
    ↓
用户回调 (Agent 级别)
    ↓
实际执行 (LLM/Tool)
```

---

## 错误处理策略

### LLM 调用错误

```go
for resp, err := range generateContent(ctx, f.Model, req, useStream) {
    if err != nil {
        // 1. 执行 OnModelErrorCallbacks
        cbResp, cbErr := f.runOnModelErrorCallbacks(ctx, req, stateDelta, err)
        if cbErr != nil {
            yield(nil, cbErr)  // 回调也失败
            return
        }
        if cbResp == nil {
            yield(nil, err)  // 无恢复响应，返回原始错误
            return
        }
        // 使用回调响应替换
        resp = &responseWithEventID{LLMResponse: cbResp, eventID: resp.eventID}
    }
    // ...
}
```

### 策略总结

1. **错误捕获**: 立即捕获 generateContent 错误
2. **回调恢复**: 执行 OnModelErrorCallbacks 尝试恢复
3. **降级响应**: 回调可返回替代响应
4. **错误传递**: 无法恢复时返回原始错误

---

## 流式 vs 非流式

### 流式模式 (`useStream = true`)

```
m.GenerateContent() → 迭代器
    │
    ├─ yield Partial 响应 1
    ├─ yield Partial 响应 2
    ├─ yield Partial 响应 3
    └─ yield 最终响应
```

**特点**:
- 多个响应依次产出
- Partial 响应不结束 Span
- 最终响应触发日志和 Span 结束
- 支持实时显示

### 非流式模式 (`useStream = false`)

```
m.GenerateContent() → 单次响应
    │
    └─ yield 最终响应
```

**特点**:
- 单次响应
- 立即触发日志和 Span 结束
- 简单快速

---

## 关键设计决策

### 1. 迭代器返回类型

**选择**: `iter.Seq2[*responseWithEventID, error]`

**优势**:
- 原生 Go 1.22+ 语法支持
- 流式和非流式统一接口
- 消费者可控制迭代终止
- 错误和响应同时处理

### 2. Span 立即结束策略

**设计**: 最终响应时立即结束 Span

**原因**:
- 避免捕获上游 yield 处理时间
- 准确记录 LLM 调用耗时
- 减少 Span 内存占用

### 3. Partial 响应不记录日志

**设计**: 仅记录最终响应日志

**原因**:
- 减少日志量
- Partial 响应不完整
- 最终响应包含完整信息

### 4. 回调可替换响应

**设计**: 回调返回非 nil 响应时替换原始响应

**用途**:
- 缓存命中时返回缓存
- 错误恢复时返回降级响应
- 后处理修改响应内容

### 5. stateDelta 贯穿调用链

**设计**: stateDelta 作为参数传递给所有回调

**用途**:
- 回调可读写状态变更
- 状态变更累积到事件
- 支持跨回调状态共享

---

## 性能优化建议

### 1. 缓存实现

```go
func cacheCallback(ctx agent.CallbackContext, req *model.LLMRequest) (*model.LLMResponse, error) {
    cacheKey := computeCacheKey(req)
    if cached, ok := cache.Get(cacheKey); ok {
        return cached, nil  // 缓存命中，跳过 LLM
    }
    return nil, nil  // 缓存未命中，继续调用
}
```

### 2. 请求去重

```go
func dedupCallback(ctx agent.CallbackContext, req *model.LLMRequest) (*model.LLMResponse, error) {
    key := computeRequestKey(req)
    if pending, ok := pendingRequests.Get(key); ok {
        return <-pending, nil  // 等待并行请求结果
    }
    return nil, nil
}
```

### 3. 限流降级

```go
func rateLimitCallback(ctx agent.CallbackContext, req *model.LLMRequest) (*model.LLMResponse, error) {
    if !rateLimiter.Allow() {
        return &model.LLMResponse{
            Content: &genai.Content{
                Parts: []*genai.Part{{Text: "请求过多，请稍后重试"}},
            },
        }, nil
    }
    return nil, nil
}
```

---

## 总结

ADK Go 的 LLM 调用机制具有以下特点：

1. **多层回调**: Before/After/OnError 三层回调，灵活扩展
2. **流式支持**: 统一迭代器接口，支持流式和非流式
3. **完整追踪**: Telemetry 集成，Span/日志/指标全覆盖
4. **错误恢复**: OnModelErrorCallbacks 支持降级和重试
5. **提前返回**: BeforeModelCallback 可跳过 LLM 调用
6. **响应替换**: AfterModelCallback 可修改响应内容
7. **状态传递**: stateDelta 贯穿调用链，支持状态共享

设计优势：
- 清晰的回调层次和执行顺序
- 灵活的错误处理和恢复机制
- 完整的可观测性支持
- 统一的流式/非流式接口
