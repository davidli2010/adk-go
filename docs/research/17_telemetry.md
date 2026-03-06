# Telemetry 集成深入研究

## 概述

Telemetry 是 ADK Go 中实现**可观测性**的关键子系统。基于 OpenTelemetry 标准，提供分布式追踪（Tracing）和日志记录（Logging）能力，帮助开发者监控、调试和优化 Agent 应用。

---

## 核心架构

### 1. Providers 结构 (`telemetry/telemetry.go`)

```go
type Providers struct {
    genAICaptureMessageContent bool
    TracerProvider             *sdktrace.TracerProvider
    LoggerProvider             *sdklog.LoggerProvider
}
```

**核心组件**：
- `TracerProvider`: 分布式追踪提供者
- `LoggerProvider`: 日志记录提供者
- `genAICaptureMessageContent`: 控制是否记录消息内容（默认隐藏）

### 2. 配置选项 (`telemetry/config.go`)

```go
type config struct {
    oTelToCloud                bool
    genAICaptureMessageContent bool
    gcpResourceProject         string
    gcpQuotaProject            string
    googleCredentials          *google.Credentials
    resource                   *resource.Resource
    spanProcessors             []sdktrace.SpanProcessor
    logProcessors              []sdklog.Processor
    tracerProvider             *sdktrace.TracerProvider
    loggerProvider             *sdklog.LoggerProvider
}
```

**配置项**：
- `oTelToCloud`: 是否导出到 GCP Cloud Trace
- `genAICaptureMessageContent`: 是否记录消息内容
- `gcpResourceProject`: GCP 项目 ID
- `spanProcessors`: 自定义 Span 处理器
- `resource`: OpenTelemetry 资源定义

### 3. Option 模式

```go
type Option interface {
    apply(*config) error
}

func WithOtelToCloud(value bool) Option
func WithGcpResourceProject(project string) Option
func WithResource(r *resource.Resource) Option
func WithSpanProcessors(p ...sdktrace.SpanProcessor) Option
func WithTracerProvider(tp *sdktrace.TracerProvider) Option
func WithGenAICaptureMessageContent(capture bool) Option
```

**设计意图**：
- 灵活的配置方式
- 支持自定义 Provider
- 可选的消息内容记录

---

## 初始化流程

### 1. 创建 Providers

```go
func New(ctx context.Context, opts ...Option) (*Providers, error) {
    cfg, err := configure(ctx, opts...)
    if err != nil {
        return nil, err
    }
    return newInternal(cfg)
}
```

### 2. 配置解析

```go
func configure(ctx context.Context, opts ...Option) (*config, error) {
    cfg, err := configFromOpts(opts...)
    if err != nil {
        return nil, err
    }
    
    if cfg.oTelToCloud {
        // 加载 ADC 凭据
        if cfg.googleCredentials == nil {
            cfg.googleCredentials, err = google.FindDefaultCredentials(ctx, ...)
        }
        // 解析 GCP 项目 ID
        cfg.gcpQuotaProject, err = resolveGcpQuotaProject(cfg)
        cfg.gcpResourceProject, err = resolveGcpResourceProject(cfg)
    }
    
    // 解析资源
    cfg.resource, err = resolveResource(ctx, cfg)
    
    // 配置导出器
    spanProcessors, logProcessors, err := configureExporters(ctx, cfg)
    cfg.spanProcessors = append(cfg.spanProcessors, spanProcessors...)
    cfg.logProcessors = append(cfg.logProcessors, logProcessors...)
    
    return cfg, nil
}
```

### 3. 资源解析

```go
func resolveResource(ctx context.Context, cfg *config) (*resource.Resource, error) {
    r := resource.Default()
    
    opts := []resource.Option{
        resource.WithAttributes(
            attribute.Key("gcp.project_id").String(cfg.gcpResourceProject),
        ),
    }
    
    if cfg.oTelToCloud {
        // 添加 GCP 检测器
        opts = append(opts, resource.WithDetectors(gcp.NewDetector()))
    }
    
    gcpResource, err := resource.New(ctx, opts...)
    // 合并默认资源和 GCP 资源
    r, err = resource.Merge(r, gcpResource)
    
    // 合并配置资源
    if cfg.resource != nil {
        r, err = resource.Merge(r, cfg.resource)
    }
    
    return r, nil
}
```

**资源属性优先级**（后者覆盖前者）：
1. `resource.Default()` - 环境变量
2. GCP 属性 - 项目 ID、运行时信息
3. GCP Detector - GCE、GKE、CloudRun 等平台信息
4. 配置资源 - 自定义资源

### 4. 导出器配置

```go
func configureExporters(ctx context.Context, cfg *config) ([]sdktrace.SpanProcessor, []sdklog.Processor, error) {
    var spanProcessors []sdktrace.SpanProcessor
    var logProcessors []sdklog.Processor
    
    otelEndpointEnv := os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
    otelTracesEndpointEnv := os.Getenv("OTEL_EXPORTER_OTLP_TRACES_ENDPOINT")
    
    // OTLP HTTP 导出器
    if otelEndpointEnv != "" || otelTracesEndpointEnv != "" {
        exporter, err := otlptracehttp.New(ctx)
        spanProcessors = append(spanProcessors, sdktrace.NewBatchSpanProcessor(exporter))
    }
    
    // GCP Cloud Trace 导出器
    if cfg.oTelToCloud {
        spanExporter, err := newGcpSpanExporter(ctx, cfg)
        spanProcessors = append(spanProcessors, sdktrace.NewBatchSpanProcessor(spanExporter))
    }
    
    // 日志导出器（类似逻辑）
    // ...
    
    return spanProcessors, logProcessors, nil
}
```

**导出器选择**：
- 环境变量 `OTEL_EXPORTER_OTLP_ENDPOINT` → OTLP HTTP Exporter
- `oTelToCloud=true` → GCP Cloud Trace Exporter
- 支持同时配置多个导出器

### 5. GCP Span Exporter

```go
func newGcpSpanExporter(ctx context.Context, cfg *config) (sdktrace.SpanExporter, error) {
    client := oauth2.NewClient(ctx, cfg.googleCredentials.TokenSource)
    return otlptracehttp.New(ctx,
        otlptracehttp.WithHTTPClient(client),
        otlptracehttp.WithEndpointURL("https://telemetry.googleapis.com/v1/traces"),
        otlptracehttp.WithHeaders(map[string]string{
            "x-goog-user-project": cfg.gcpQuotaProject,
        }))
}
```

**GCP 集成**：
- 使用 ADC 认证
- 导出到 Cloud Trace API
- 设置 quota project 避免认证错误

---

## 分布式追踪（Tracing）

### 1. Tracer 初始化

```go
const systemName = "gcp.vertex.agent"

var tracer trace.Tracer = otel.GetTracerProvider().Tracer(
    systemName,
    trace.WithInstrumentationVersion(version.Version),
    trace.WithSchemaURL(semconv.SchemaURL),
)
```

**Tracer 信息**：
- 名称：`gcp.vertex.agent`
- 版本：ADK 版本号
- Schema URL：OpenTelemetry 语义约定

### 2. Span 类型

#### invoke_agent Span

```go
func StartInvokeAgentSpan(ctx context.Context, agent agent, sessionID, invocationID string) (context.Context, trace.Span) {
    agentName := agent.Name()
    spanCtx, span := tracer.Start(ctx, fmt.Sprintf("invoke_agent %s", agentName), trace.WithAttributes(
        gcpVertexAgentInvocationID.String(invocationID),
        semconv.GenAIOperationNameInvokeAgent,
        semconv.GenAIAgentDescription(agent.Description()),
        semconv.GenAIAgentName(agentName),
        semconv.GenAIConversationID(sessionID),
    ))
    return spanCtx, span
}
```

**属性**：
- `gcp.vertex.agent.invocation_id`: 调用 ID
- `gen_ai.operation_name`: `invoke_agent`
- `gen_ai.agent.description`: Agent 描述
- `gen_ai.agent.name`: Agent 名称
- `gen_ai.conversation_id`: 会话 ID

#### generate_content Span

```go
func StartGenerateContentSpan(ctx context.Context, params StartGenerateContentSpanParams) (context.Context, trace.Span) {
    modelName := params.ModelName
    spanCtx, span := tracer.Start(ctx, fmt.Sprintf("generate_content %s", modelName), trace.WithAttributes(
        gcpVertexAgentInvocationID.String(params.InvocationID),
        semconv.GenAIOperationNameGenerateContent,
        semconv.GenAIRequestModel(modelName),
    ))
    return spanCtx, span
}
```

**属性**：
- `gen_ai.operation_name`: `generate_content`
- `gen_ai.request.model`: 模型名称
- Token 使用量（响应后记录）

#### execute_tool Span

```go
func StartExecuteToolSpan(ctx context.Context, params StartExecuteToolSpanParams) (context.Context, trace.Span) {
    toolName := params.ToolName
    spanCtx, span := tracer.Start(ctx, fmt.Sprintf("execute_tool %s", toolName), trace.WithAttributes(
        semconv.GenAIOperationNameExecuteTool,
        semconv.GenAIToolName(toolName),
        gcpVertexAgentToolCallArgsName.String(safeSerialize(params.Args)),
    ))
    return spanCtx, span
}
```

**属性**：
- `gen_ai.operation_name`: `execute_tool`
- `gen_ai.tool.name`: 工具名称
- `gcp.vertex.agent.tool_call_args`: 工具参数（JSON 序列化）
- `gcp.vertex.agent.tool_response`: 工具响应（JSON 序列化）

### 3. 结果记录

#### Agent 结果

```go
func TraceAgentResult(span trace.Span, params TraceAgentResultParams) {
    recordErrorAndStatus(span, params.Error)
}

func recordErrorAndStatus(span trace.Span, err error) {
    if err == nil {
        return
    }
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
}
```

#### Generate Content 结果

```go
func TraceGenerateContentResult(span trace.Span, params TraceGenerateContentResultParams) {
    recordErrorAndStatus(span, params.Error)
    if params.Response == nil {
        return
    }
    span.SetAttributes(
        gcpVertexAgentEventID.String(params.EventID),
        semconv.GenAIResponseFinishReasons(string(params.Response.FinishReason)),
    )
    if params.Response.UsageMetadata != nil {
        span.SetAttributes(
            semconv.GenAIUsageInputTokens(int(params.Response.UsageMetadata.PromptTokenCount)),
            semconv.GenAIUsageOutputTokens(int(params.Response.UsageMetadata.CandidatesTokenCount)),
        )
    }
}
```

**记录内容**：
- Event ID
- Finish Reason
- Token 使用量（输入/输出）

#### Tool 结果

```go
func TraceToolResult(span trace.Span, params TraceToolResultParams) {
    recordErrorAndStatus(span, params.Error)
    
    attributes := []attribute.KeyValue{
        semconv.GenAIOperationNameKey.String(executeToolName),
        semconv.GenAIToolDescriptionKey.String(params.Description),
    }
    
    // 提取 FunctionResponse
    if params.ResponseEvent != nil {
        attributes = append(attributes, gcpVertexAgentEventID.String(params.ResponseEvent.ID))
        // 提取 tool_call_id 和 tool_response
        // ...
    }
    
    span.SetAttributes(attributes...)
}
```

### 4. WrapYield 辅助函数

```go
func WrapYield[T any](span trace.Span, yield func(T, error) bool, finalizeSpan func(trace.Span, T, error)) (wrapped func(T, error) bool, endSpan func()) {
    var val T
    var err error
    wrapped = func(v T, e error) bool {
        val = v
        err = e
        return yield(v, e)
    }
    endSpan = func() {
        finalizeSpan(span, val, err)
        span.End()
    }
    return wrapped, endSpan
}
```

**用途**：
- 包装迭代器的 yield 函数
- 自动记录最终值和错误
- 确保 span 被正确结束

**示例**：
```go
wrapped, endSpan := WrapYield(span, yield, func(span trace.Span, event *Event, err error) {
    TraceAgentResult(span, TraceAgentResultParams{
        ResponseEvent: event,
        Error:         err,
    })
})
defer endSpan()
```

---

## 日志记录（Logging）

### 1. Logger 初始化

```go
var otelLogger = global.GetLoggerProvider().Logger(
    systemName,
    log.WithSchemaURL(semconv.SchemaURL),
    log.WithInstrumentationVersion(version.Version),
)
```

### 2. 消息内容控制

```go
var genAICaptureMessageContent atomic.Bool

func SetGenAICaptureMessageContent(capture bool) {
    genAICaptureMessageContent.Store(capture)
}

func getGenAICaptureMessageContent() bool {
    return genAICaptureMessageContent.Load()
}

const elidedContent = "<elided>"
```

**安全考虑**：
- 默认隐藏消息内容（隐私保护）
- 通过环境变量 `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` 控制
- 运行时可通过 `SetGenAICaptureMessageContent` 修改

### 3. 日志事件类型

#### Request 日志

```go
func LogRequest(ctx context.Context, req *model.LLMRequest, backend genai.Backend) {
    genAISystem := variantToGenAISystem(backend)
    logSystemMessage(ctx, req, genAISystem)
    for _, content := range req.Contents {
        logUserMessage(ctx, content, genAISystem)
    }
}
```

**记录内容**：
- 系统消息（`gen_ai.system.message`）
- 用户消息（`gen_ai.user.message`）

#### System Message 日志

```go
func logSystemMessage(ctx context.Context, req *model.LLMRequest, genAISystem *log.KeyValue) {
    record := log.Record{}
    record.SetEventName("gen_ai.system.message")
    record.SetBody(log.MapValue(
        log.KeyValue{Key: "content", Value: extractSystemMessage(req)},
    ))
    if genAISystem != nil {
        record.AddAttributes(*genAISystem)
    }
    otelLogger.Emit(ctx, record)
}
```

**事件名称**：`gen_ai.system.message`

**Body 结构**：
```json
{
  "content": "You are a helpful assistant..."  // 或 "<elided>"
}
```

#### User Message 日志

```go
func logUserMessage(ctx context.Context, content *genai.Content, genAISystem *log.KeyValue) {
    record := log.Record{}
    record.SetEventName("gen_ai.user.message")
    record.SetBody(log.MapValue(
        log.KeyValue{Key: "content", Value: toLogValue(contentToJSONLikeValue(content))},
    ))
    if genAISystem != nil {
        record.AddAttributes(*genAISystem)
    }
    otelLogger.Emit(ctx, record)
}
```

**事件名称**：`gen_ai.user.message`

**Body 结构**：
```json
{
  "content": {
    "role": "user",
    "parts": [
      {"text": "Hello"}
    ]
  }
}
```

#### Response 日志

```go
func LogResponse(ctx context.Context, resp *model.LLMResponse, backend genai.Backend) {
    record := log.Record{}
    record.SetEventName("gen_ai.choice")
    
    kvs := []log.KeyValue{
        log.Int("index", 0),
        {Key: "content", Value: contentToLogValue(content)},
    }
    
    if finishReason != "" {
        kvs = append(kvs, log.String("finish_reason", finishReason))
    }
    record.SetBody(log.MapValue(kvs...))
    
    otelLogger.Emit(ctx, record)
}
```

**事件名称**：`gen_ai.choice`

**Body 结构**：
```json
{
  "index": 0,
  "content": {
    "role": "model",
    "parts": [
      {"text": "Hello! How can I help you?"}
    ]
  },
  "finish_reason": "STOP"
}
```

### 4. GenAI System 属性

```go
func variantToGenAISystem(variant genai.Backend) *log.KeyValue {
    if variant == genai.BackendVertexAI {
        val := log.KeyValueFromAttribute(semconv.GenAISystemGCPVertexAI)
        return &val
    }
    if variant == genai.BackendGeminiAPI {
        val := log.KeyValueFromAttribute(semconv.GenAISystemGCPGemini)
        return &val
    }
    return nil
}
```

**属性值**：
- `gcp.vertex_ai`: Vertex AI Backend
- `gcp.gemini`: Gemini API Backend

### 5. 内容转换

```go
func contentToJSONLikeValue(c *genai.Content) any {
    if !getGenAICaptureMessageContent() {
        return elidedContent
    }
    if c == nil {
        return nil
    }
    
    // 序列化为 JSON
    b, err := json.Marshal(c)
    if err != nil {
        return "<not_serializable>"
    }
    
    // 反序列化为 map
    var m map[string]any
    if err := json.Unmarshal(b, &m); err != nil {
        return "<not_serializable>"
    }
    return m
}
```

**转换逻辑**：
1. 检查是否允许记录内容
2. 序列化为 JSON（保留字段名）
3. 反序列化为 map（便于日志系统处理）

### 6. Log Value 转换器

```go
func toLogValue(v any) log.Value {
    switch val := v.(type) {
    case nil:
        return log.Value{}
    case string:
        return log.StringValue(val)
    case bool:
        return log.BoolValue(val)
    case float64:
        return log.Float64Value(val)
    case int:
        return log.IntValue(val)
    case []any:
        values := make([]log.Value, 0, len(val))
        for _, item := range val {
            values = append(values, toLogValue(item))
        }
        return log.SliceValue(values...)
    case map[string]any:
        kvs := make([]log.KeyValue, 0, len(val))
        for k, v := range val {
            kvs = append(kvs, log.KeyValue{Key: k, Value: toLogValue(v)})
        }
        return log.MapValue(kvs...)
    default:
        return log.StringValue(fmt.Sprintf("%v", val))
    }
}
```

**支持的类型**：
- `nil` → 空 Value
- `string` → StringValue
- `bool` → BoolValue
- `float64` → Float64Value
- `int` → IntValue
- `[]any` → SliceValue
- `map[string]any` → MapValue

---

## 集成与使用

### 基本使用

```go
import (
    "context"
    "log"
    "time"
    
    "go.opentelemetry.io/otel/sdk/resource"
    semconv "go.opentelemetry.io/otel/semconv/v1.36.0"
    "google.golang.org/adk/telemetry"
)

func main() {
    ctx := context.Background()
    
    // 创建资源
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceNameKey.String("my-adk-service"),
            semconv.ServiceVersionKey.String("1.0.0"),
        ),
    )
    if err != nil {
        log.Fatalf("failed to create resource: %v", err)
    }
    
    // 初始化 telemetry
    telemetryProviders, err := telemetry.New(ctx,
        telemetry.WithOtelToCloud(true),
        telemetry.WithResource(res),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer func() {
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        if err := telemetryProviders.Shutdown(shutdownCtx); err != nil {
            log.Printf("telemetry shutdown failed: %v", err)
        }
    }()
    
    // 设置全局 Provider
    telemetryProviders.SetGlobalOtelProviders()
    
    // 应用代码...
}
```

### 自定义 Span Processor

```go
telemetryProviders, err := telemetry.New(ctx,
    telemetry.WithSpanProcessors(
        sdktrace.NewBatchSpanProcessor(customExporter),
    ),
)
```

### 自定义 TracerProvider

```go
tp := sdktrace.NewTracerProvider(
    sdktrace.WithSampler(sdktrace.AlwaysSample()),
    sdktrace.WithResource(res),
)

telemetryProviders, err := telemetry.New(ctx,
    telemetry.WithTracerProvider(tp),
)
```

### 启用消息内容记录

```go
telemetryProviders, err := telemetry.New(ctx,
    telemetry.WithGenAICaptureMessageContent(true),
)
```

**或通过环境变量**：
```bash
export OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
```

---

## Span 层级结构

```
invoke_agent <agent_name>
├── generate_content <model_name>
│   ├── gen_ai.system.message (log)
│   ├── gen_ai.user.message (log)
│   └── gen_ai.choice (log)
├── execute_tool <tool_name>
│   └── ...
├── execute_tool <tool_name>
│   └── ...
└── generate_content <model_name>
    ├── gen_ai.system.message (log)
    ├── gen_ai.user.message (log)
    └── gen_ai.choice (log)
```

**层级关系**：
- `invoke_agent` 是顶层 span
- `generate_content` 和 `execute_tool` 是子 span
- 日志事件与最近的 span 关联

---

## 使用场景

### 场景 1：性能监控

```
invoke_agent assistant
├── generate_content gemini-2.0-flash-exp (1.2s)
├── execute_tool search (0.3s)
└── generate_content gemini-2.0-flash-exp (0.8s)
Total: 2.3s
```

**用途**：
- 识别性能瓶颈
- 优化慢查询和工具调用
- 监控 LLM 响应时间

### 场景 2：错误诊断

```
invoke_agent assistant
├── generate_content gemini-2.0-flash-exp (status: Error)
│   └── Error: rate limit exceeded
└── ...
```

**用途**：
- 追踪错误发生位置
- 记录错误详细信息
- 分析错误模式

### 场景 3：Token 使用分析

```
generate_content gemini-2.0-flash-exp
├── gen_ai.usage.input_tokens: 150
└── gen_ai.usage.output_tokens: 80
```

**用途**：
- 监控成本
- 优化提示词长度
- 分析 Token 消耗模式

### 场景 4：审计与合规

```
gen_ai.user.message
├── content: "Transfer $1000 to account XXX"
└── timestamp: "2024-01-15T10:30:00Z"

execute_tool transfer_money
├── tool_call_args: {"amount": 1000, "account": "XXX"}
└── tool_response: {"status": "success", "transaction_id": "TXN123"}
```

**用途**：
- 记录所有用户交互
- 审计工具调用
- 合规性检查

### 场景 5：调试与开发

```
[Trace ID: abc123]
invoke_agent my_agent
├── generate_content gemini-pro
│   ├── System Message: "You are a helpful assistant..."
│   ├── User Message: "What's the weather?"
│   └── Choice: "I need to call the weather tool..."
└── execute_tool get_weather
    ├── Args: {"location": "San Francisco"}
    └── Response: {"temp": 72, "condition": "sunny"}
```

**用途**：
- 查看完整的对话流程
- 分析 LLM 决策过程
- 调试工具调用问题

---

## 环境变量配置

### OpenTelemetry 标准

```bash
# 服务名称
export OTEL_SERVICE_NAME=my-adk-service

# OTLP Endpoint
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

# Traces Endpoint (覆盖)
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:4318/v1/traces

# Logs Endpoint (覆盖)
export OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://localhost:4318/v1/logs

# 资源属性
export OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.version=1.0.0
```

### GCP 特定

```bash
# GCP 项目 ID
export GOOGLE_CLOUD_PROJECT=my-gcp-project

# ADC 凭据文件
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json

# 消息内容记录
export OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
```

---

## 设计亮点

### 1. 标准化

- 基于 OpenTelemetry 标准
- 遵循语义约定（Semantic Conventions）
- 兼容主流可观测性平台

### 2. 隐私保护

- 默认隐藏消息内容
- 可通过配置启用
- 符合数据保护要求

### 3. 灵活配置

- 支持多种导出器
- 可自定义 Span Processor
- 可覆盖默认 Provider

### 4. GCP 集成

- 自动检测 GCP 项目
- 支持 Cloud Trace
- 支持 Cloud Logging（规划中）

### 5. 迭代器支持

- `WrapYield` 辅助函数
- 自动管理 Span 生命周期
- 支持流式响应

---

## 与其他组件的关系

### 与 Agent 的关系

- 在 `agent.Run()` 中创建 `invoke_agent` span
- 在 `LLMAgent.run()` 中创建 `generate_content` span
- 在工具执行中创建 `execute_tool` span

### 与 Model 的关系

- 记录 LLM 请求和响应
- 跟踪 Token 使用量
- 记录错误和重试

### 与 Tool 的关系

- 记录工具调用参数和响应
- 跟踪工具执行时间
- 记录工具错误

---

## 限制与改进方向

### 当前限制

1. **缺少 Metrics**：
   - 未实现 MeterProvider
   - 无法收集指标数据

2. **日志导出有限**：
   - Golang OTel exporter to CloudLogging 尚未可用
   - 仅支持 OTLP HTTP 导出

3. **采样策略**：
   - 默认采样所有 span
   - 缺少基于规则的采样

### 改进方向

1. **完善 Metrics**：
   - 实现 MeterProvider
   - 收集请求计数、延迟、错误率等指标

2. **增强日志**：
   - 支持 Cloud Logging
   - 支持日志级别过滤

3. **智能采样**：
   - 基于错误率的采样
   - 基于延迟的采样
   - 自定义采样策略

4. **性能优化**：
   - 批量导出优化
   - 异步处理
   - 缓存机制

---

## 总结

Telemetry 是 ADK Go 中实现可观测性的关键组件：

1. **核心功能**：分布式追踪、日志记录、指标收集（规划中）
2. **设计理念**：标准化、隐私保护、灵活配置、GCP 集成
3. **使用场景**：性能监控、错误诊断、成本分析、审计合规、调试开发
4. **扩展性**：支持多种导出器和自定义 Processor

通过 Telemetry，开发者可以全面监控 Agent 应用的运行状态，快速定位问题，优化性能，并满足合规要求。基于 OpenTelemetry 标准的设计确保了与主流可观测性平台的兼容性。