# ADK Go 模型集成详解

## 概述

本文档深入分析 ADK Go 中模型集成的实现，包括 Gemini 模型实现、流式响应处理机制。

---

## 核心结构

### geminiModel 结构

**位置**: `model/gemini/gemini.go:36-40`

```go
type geminiModel struct {
    client             *genai.Client
    name               string
    versionHeaderValue string
}
```

**字段说明**:

| 字段 | 类型 | 用途 |
|------|------|------|
| `client` | *genai.Client | Gemini API 客户端 |
| `name` | string | 模型名称（如 gemini-2.5-flash） |
| `versionHeaderValue` | string | 版本头信息（用于遥测） |

---

## NewModel() 构造函数

**位置**: `model/gemini/gemini.go:42-64`

### 方法签名

```go
func NewModel(ctx context.Context, modelName string, cfg *genai.ClientConfig) (model.LLM, error)
```

### 创建流程

```go
func NewModel(ctx context.Context, modelName string, cfg *genai.ClientConfig) (model.LLM, error) {
    // 1. 创建 genai.Client
    client, err := genai.NewClient(ctx, cfg)
    if err != nil {
        return nil, err
    }
    
    // 2. 构建版本头信息
    headerValue := fmt.Sprintf("google-adk/%s gl-go/%s", 
        version.Version,
        strings.TrimPrefix(runtime.Version(), "go"))
    
    // 3. 返回模型实例
    return &geminiModel{
        name:               modelName,
        client:             client,
        versionHeaderValue: headerValue,
    }, nil
}
```

### 版本头信息

```go
headerValue = "google-adk/{version} gl-go/{go-version}"
```

**用途**:
- 标识 ADK Go 版本
- 标识 Go 运行时版本
- 用于 API 遥测和分析

---

## model.LLM 接口

**位置**: `model/llm.go`

### 接口定义

```go
type LLM interface {
    Name() string
    GenerateContent(ctx context.Context, req *LLMRequest, stream bool) iter.Seq2[*LLMResponse, error]
}
```

### Gemini 实现

```go
func (m *geminiModel) Name() string {
    return m.name
}

func (m *geminiModel) GenerateContent(ctx context.Context, req *model.LLMRequest, stream bool) iter.Seq2[*model.LLMResponse, error] {
    // ...
}
```

---

## GenerateContent() 主方法

**位置**: `model/gemini/gemini.go:71-92`

### 方法签名

```go
func (m *geminiModel) GenerateContent(ctx context.Context, req *model.LLMRequest, stream bool) iter.Seq2[*model.LLMResponse, error]
```

### 完整实现

```go
func (m *geminiModel) GenerateContent(ctx context.Context, req *model.LLMRequest, stream bool) iter.Seq2[*model.LLMResponse, error] {
    // 1. 可能追加用户内容
    m.maybeAppendUserContent(req)
    
    // 2. 初始化配置
    if req.Config == nil {
        req.Config = &genai.GenerateContentConfig{}
    }
    if req.Config.HTTPOptions == nil {
        req.Config.HTTPOptions = &genai.HTTPOptions{}
    }
    if req.Config.HTTPOptions.Headers == nil {
        req.Config.HTTPOptions.Headers = make(http.Header)
    }
    
    // 3. 添加版本头
    m.addHeaders(req.Config.HTTPOptions.Headers)
    
    // 4. 根据 stream 参数选择调用方式
    if stream {
        return m.generateStream(ctx, req)
    }
    
    return func(yield func(*model.LLMResponse, error) bool) {
        resp, err := m.generate(ctx, req)
        yield(resp, err)
    }
}
```

### 执行流程图

```
GenerateContent()
    │
    ├─ 1. maybeAppendUserContent()
    │   └─ 确保有用户内容可继续
    │
    ├─ 2. 初始化配置
    │   ├─ req.Config
    │   └─ req.Config.HTTPOptions
    │
    ├─ 3. addHeaders()
    │   └─ 设置 x-goog-api-client 和 user-agent
    │
    └─ 4. stream ?
        ├─ true  → generateStream()
        └─ false → generate()
```

---

## maybeAppendUserContent() 用户内容处理

**位置**: `model/gemini/gemini.go:145-154`

### 方法签名

```go
func (m *geminiModel) maybeAppendUserContent(req *model.LLMRequest)
```

### 实现逻辑

```go
func (m *geminiModel) maybeAppendUserContent(req *model.LLMRequest) {
    // 1. 空内容时添加默认用户内容
    if len(req.Contents) == 0 {
        req.Contents = append(req.Contents, 
            genai.NewContentFromText("Handle the requests as specified in the System Instruction.", "user"))
    }
    
    // 2. 最后内容不是用户时追加继续提示
    if last := req.Contents[len(req.Contents)-1]; last != nil && last.Role != "user" {
        req.Contents = append(req.Contents, 
            genai.NewContentFromText("Continue processing previous requests as instructed. Exit or provide a summary if no more outputs are needed.", "user"))
    }
}
```

**用途**:
- 确保模型有内容可处理
- 处理非用户角色的最后内容
- 支持多轮对话继续

---

## addHeaders() 头信息添加

**位置**: `model/gemini/gemini.go:94-98`

### 实现逻辑

```go
func (m *geminiModel) addHeaders(headers http.Header) {
    headers.Set("x-goog-api-client", m.versionHeaderValue)
    headers.Set("user-agent", m.versionHeaderValue)
}
```

**设置的头**:
- `x-goog-api-client`: 标识 ADK 和 Go 版本
- `user-agent`: 同样的版本信息

---

## modelName() 模型名称

**位置**: `model/gemini/gemini.go:100-108`

### 方法签名

```go
func (m *geminiModel) modelName(req *model.LLMRequest) string
```

### 实现逻辑

```go
func (m *geminiModel) modelName(req *model.LLMRequest) string {
    // 优先使用请求中的模型名称
    if req.Model != "" {
        return req.Model
    }
    // 回退到构造时的模型名称
    return m.name
}
```

**用途**:
- 支持 BeforeModelCallback 动态修改模型
- 保持向后兼容

---

## generate() 同步调用

**位置**: `model/gemini/gemini.go:110-121`

### 方法签名

```go
func (m *geminiModel) generate(ctx context.Context, req *model.LLMRequest) (*model.LLMResponse, error)
```

### 实现逻辑

```go
func (m *geminiModel) generate(ctx context.Context, req *model.LLMRequest) (*model.LLMResponse, error) {
    // 调用 Gemini API
    resp, err := m.client.Models.GenerateContent(ctx, m.modelName(req), req.Contents, req.Config)
    if err != nil {
        return nil, fmt.Errorf("failed to call model: %w", err)
    }
    
    // 检查响应候选
    if len(resp.Candidates) == 0 {
        return nil, fmt.Errorf("empty response")
    }
    
    // 转换响应格式
    return converters.Genai2LLMResponse(resp), nil
}
```

### 返回值

- 成功：返回 `*model.LLMResponse`
- 失败：返回 `nil, error`

---

## generateStream() 流式调用

**位置**: `model/gemini/gemini.go:123-143`

### 方法签名

```go
func (m *geminiModel) generateStream(ctx context.Context, req *model.LLMRequest) iter.Seq2[*model.LLMResponse, error]
```

### 实现逻辑

```go
func (m *geminiModel) generateStream(ctx context.Context, req *model.LLMRequest) iter.Seq2[*model.LLMResponse, error] {
    // 1. 创建流式响应聚合器
    aggregator := llminternal.NewStreamingResponseAggregator()
    
    return func(yield func(*model.LLMResponse, error) bool) {
        // 2. 调用流式 API
        for resp, err := range m.client.Models.GenerateContentStream(ctx, m.modelName(req), req.Contents, req.Config) {
            if err != nil {
                yield(nil, err)
                return
            }
            
            // 3. 处理每个响应片段
            for llmResponse, err := range aggregator.ProcessResponse(ctx, resp) {
                if !yield(llmResponse, err) {
                    return // 消费者停止
                }
            }
        }
        
        // 4. 关闭聚合器，生成最终聚合响应
        if closeResult := aggregator.Close(); closeResult != nil {
            yield(closeResult, nil)
        }
    }
}
```

### 流式处理流程图

```
generateStream()
    │
    ├─ 1. 创建聚合器
    │   └─ NewStreamingResponseAggregator()
    │
    ├─ 2. 流式迭代
    │   └─ for resp, err := range GenerateContentStream()
    │
    ├─ 3. 处理响应片段
    │   └─ aggregator.ProcessResponse()
    │       ├─ 转换响应格式
    │       ├─ 聚合文本
    │       └─ yield 每个片段
    │
    └─ 4. 关闭聚合器
        └─ aggregator.Close()
            └─ yield 最终聚合响应
```

---

## StreamingResponseAggregator 流式响应聚合器

**位置**: `internal/llminternal/stream_aggregator.go`

### 结构定义

```go
type streamingResponseAggregator struct {
    text        string
    thoughtText string
    response    *model.LLMResponse
    role        string
}
```

**字段说明**:

| 字段 | 类型 | 用途 |
|------|------|------|
| `text` | string | 累积的普通文本 |
| `thoughtText` | string | 累积的思考文本 |
| `response` | *model.LLMResponse | 当前响应引用 |
| `role` | string | 内容角色 |

### ProcessResponse() 响应处理

**位置**: `internal/llminternal/stream_aggregator.go:46-67`

```go
func (s *streamingResponseAggregator) ProcessResponse(ctx context.Context, genResp *genai.GenerateContentResponse) iter.Seq2[*model.LLMResponse, error] {
    return func(yield func(*model.LLMResponse, error) bool) {
        if len(genResp.Candidates) == 0 {
            yield(nil, fmt.Errorf("empty response"))
            return
        }
        
        // 转换响应
        resp := converters.Genai2LLMResponse(genResp)
        resp.TurnComplete = candidate.FinishReason != ""
        
        // 聚合并 yield 中间事件
        if aggrResp := s.aggregateResponse(resp); aggrResp != nil {
            if !yield(aggrResp, nil) {
                return
            }
        }
        
        // Yield 处理后的响应
        if !yield(resp, nil) {
            return
        }
    }
}
```

### aggregateResponse() 聚合逻辑

**位置**: `internal/llminternal/stream_aggregator.go:71-107`

```go
func (s *streamingResponseAggregator) aggregateResponse(llmResponse *model.LLMResponse) *model.LLMResponse {
    s.response = llmResponse
    
    var part0 *genai.Part
    if llmResponse.Content != nil && len(llmResponse.Content.Parts) > 0 {
        part0 = llmResponse.Content.Parts[0]
        s.role = llmResponse.Content.Role
    }
    
    // 1. 如果是文本，追加到累积
    if part0 != nil && part0.Text != "" {
        if part0.Thought {
            s.thoughtText += part0.Text
        } else {
            s.text += part0.Text
        }
        llmResponse.Partial = true
        return nil  // 不 yield 中间响应
    }
    
    // 2. 过滤空部分
    if part0 != nil && reflect.ValueOf(*part0).IsZero() {
        llmResponse.Partial = true
        return nil
    }
    
    // 3. 有累积文本但当前无内容，返回聚合响应
    if (s.thoughtText != "" || s.text != "") &&
        (llmResponse.Content == nil ||
            len(llmResponse.Content.Parts) == 0 ||
            (len(llmResponse.Content.Parts) > 0 && llmResponse.Content.Parts[0].InlineData == nil)) {
        return s.createAggregateResponse()
    }
    
    return nil
}
```

### Close() 关闭聚合器

**位置**: `internal/llminternal/stream_aggregator.go:111-113`

```go
func (s *streamingResponseAggregator) Close() *model.LLMResponse {
    return s.createAggregateResponse()
}
```

**用途**: 流式结束后生成最终聚合响应

### createAggregateResponse() 创建聚合响应

**位置**: `internal/llminternal/stream_aggregator.go:115-139`

```go
func (s *streamingResponseAggregator) createAggregateResponse() *model.LLMResponse {
    if (s.text != "" || s.thoughtText != "") && s.response != nil {
        var parts []*genai.Part
        if s.thoughtText != "" {
            parts = append(parts, &genai.Part{Text: s.thoughtText, Thought: true})
        }
        if s.text != "" {
            parts = append(parts, &genai.Part{Text: s.text, Thought: false})
        }
        
        response := &model.LLMResponse{
            Content:            &genai.Content{Parts: parts, Role: s.role},
            ErrorCode:          s.response.ErrorCode,
            ErrorMessage:       s.response.ErrorMessage,
            UsageMetadata:      s.response.UsageMetadata,
            GroundingMetadata:  s.response.GroundingMetadata,
            CitationMetadata:   s.response.CitationMetadata,
            FinishReason:       s.response.FinishReason,
        }
        s.clear()
        return response
    }
    s.clear()
    return nil
}
```

### clear() 清理状态

```go
func (s *streamingResponseAggregator) clear() {
    s.response = nil
    s.text = ""
    s.thoughtText = ""
    s.role = ""
}
```

---

## 流式响应处理机制

### 处理流程

```
Gemini API 流式响应
    │
    ├─ Chunk 1: "Hello" (Partial)
    │   ├─ 累积到 s.text
    │   └─ yield resp (Partial=true)
    │
    ├─ Chunk 2: " world" (Partial)
    │   ├─ 累积到 s.text
    │   └─ yield resp (Partial=true)
    │
    └─ Chunk 3: "" (Final)
        ├─ 检测到空内容
        ├─ createAggregateResponse()
        ├─ yield 聚合响应
        └─ clear()
```

### 关键机制

1. **文本累积**: 多个文本片段累积到 `text`
2. **部分标记**: `Partial = true` 标记中间响应
3. **聚合响应**: 空内容时生成最终聚合响应
4. **思考文本**: 支持 Thought 标记的文本

---

## 与 model.LLM 接口的关系

```go
type LLM interface {
    Name() string
    GenerateContent(ctx context.Context, req *LLMRequest, stream bool) iter.Seq2[*LLMResponse, error]
}
```

**实现对照**:

| 接口方法 | Gemini 实现 |
|----------|------------|
| Name() | `m.name` |
| GenerateContent() | `geminiModel.GenerateContent()` |

---

## 关键设计决策

### 1. 迭代器返回类型

**选择**: `iter.Seq2[*model.LLMResponse, error]`

**优势**:
- 原生 Go 1.22+ 支持
- 流式和非流式统一接口
- 消费者可控迭代

### 2. 流式/非流式分支

**设计**: `if stream { ... } else { ... }`

**原因**:
- 非流式：简单直接返回
- 流式：需要聚合器处理

### 3. StreamingResponseAggregator

**设计**: 累积文本，生成聚合响应

**优势**:
- 处理部分响应
- 支持思考模式
- 统一输出格式

### 4. 版本头信息

**设计**: `google-adk/{version} gl-go/{go-version}`

**用途**:
- 服务端遥测
- 调试和问题追踪

### 5. maybeAppendUserContent

**设计**: 自动确保有用户内容

**原因**:
- Gemini API 要求有用户内容
- 简化用户使用

---

## 使用示例

### 创建模型

```go
model, err := gemini.NewModel(ctx, "gemini-2.5-flash", &genai.ClientConfig{
    APIKey: os.Getenv("GEMINI_API_KEY"),
})
```

### 非流式调用

```go
for resp, err := range model.GenerateContent(ctx, req, false) {
    if err != nil {
        // 处理错误
    }
    // 处理响应
}
```

### 流式调用

```go
for resp, err := range model.GenerateContent(ctx, req, true) {
    if err != nil {
        // 处理错误
    }
    if resp.Partial {
        // 处理部分响应
    } else {
        // 处理最终响应
    }
}
```

---

## 总结

ADK Go 的模型集成具有以下特点：

1. **统一接口**: model.LLM 接口，适配多种模型
2. **Gemini 实现**: 完整的 Gemini API 封装
3. **流式支持**: StreamingResponseAggregator 处理流式响应
4. **版本追踪**: 版本头信息用于遥测
5. **自动处理**: maybeAppendUserContent 自动补充内容

设计优势：
- 简洁的 API 设计
- 灵活的流式/非流式切换
- 完整的响应聚合机制
- 易于扩展支持其他模型
