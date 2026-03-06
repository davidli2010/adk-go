# ADK Go 部署服务详解

## 概述

本文档深入分析 ADK Go 的两种部署服务：REST API 服务器和 A2A 协议服务器。

---

## 1. REST API 服务器

### 概述

REST API 服务器提供 HTTP 接口，用于与 ADK Agent 进行交互。

### 核心文件

```
server/adkrest/
├── handler.go          # 主处理器
├── controllers/       # 控制器
│   ├── runtime.go     # 运行时 API
│   ├── sessions.go    # 会话 API
│   ├── apps.go        # 应用 API
│   ├── artifacts.go   # 资源 API
│   └── debug.go       # 调试 API
├── internal/
│   ├── routers/       # 路由
│   ├── services/       # 服务
│   └── models/        # 模型
```

### 创建 Handler

**位置**: `server/adkrest/handler.go:31-48`

```go
func NewHandler(config *launcher.Config, sseWriteTimeout time.Duration) http.Handler {
    // 1. 创建调试遥测
    debugTelemetry := services.NewDebugTelemetry()
    
    // 2. 配置遥测
    config.TelemetryOptions = append(config.TelemetryOptions, 
        telemetry.WithSpanProcessors(debugTelemetry.SpanProcessor()))
    config.TelemetryOptions = append(config.TelemetryOptions,
        telemetry.WithLogRecordProcessors(debugTelemetry.LogProcessor()))
    
    // 3. 创建路由器
    router := mux.NewRouter().StrictSlash(true)
    
    // 4. 设置子路由
    setupRouter(router,
        // 会话 API
        routers.NewSessionsAPIRouter(controllers.NewSessionsAPIController(config.SessionService)),
        // 运行时 API
        routers.NewRuntimeAPIRouter(controllers.NewRuntimeAPIController(
            config.SessionService, 
            config.MemoryService, 
            config.AgentLoader, 
            config.ArtifactService, 
            sseWriteTimeout, 
            config.PluginConfig,
        )),
        // 应用 API
        routers.NewAppsAPIRouter(controllers.NewAppsAPIController(config.AgentLoader)),
        // 调试 API
        routers.NewDebugAPIRouter(controllers.NewDebugAPIController(
            config.SessionService, 
            config.AgentLoader, 
            debugTelemetry,
        )),
        // 资源 API
        routers.NewArtifactsAPIRouter(controllers.NewArtifactsAPIController(config.ArtifactService)),
        // 评估 API
        &routers.EvalAPIRouter{},
    )
    
    return router
}
```

### API 路由

| 路由 | 控制器 | 功能 |
|------|---------|------|
| `/sessions/*` | SessionsAPIController | 会话管理 |
| `/runtime/*` | RuntimeAPIController | Agent 运行 |
| `/apps/*` | AppsAPIController | 应用管理 |
| `/artifacts/*` | ArtifactsAPIController | 资源管理 |
| `/debug/*` | DebugAPIController | 调试接口 |

### 运行时 API (runtime.go)

```go
type RuntimeAPIController struct {
    sessionService  session.Service
    memoryService   memory.Service
    agentLoader     launcher.AgentLoader
    artifactService artifact.Service
    
    sseWriteTimeout time.Duration
    pluginConfig    runner.PluginConfig
}
```

**核心功能**:
- `POST /runtime/{app_name}/users/{user_id}/sessions/{session_id}/run`: 运行 Agent
- SSE 流式返回事件

### 会话 API (sessions.go)

```go
type SessionsAPIController struct {
    sessionService session.Service
}
```

**核心功能**:
- `POST /sessions`: 创建会话
- `GET /sessions/{session_id}`: 获取会话
- `GET /sessions`: 列出会话
- `DELETE /sessions/{session_id}`: 删除会话
- `POST /sessions/{session_id}/events`: 追加事件

### 资源 API (artifacts.go)

```go
type ArtifactsAPIController struct {
    artifactService artifact.Service
}
```

**核心功能**:
- `POST /artifacts`: 保存资源
- `GET /artifacts/{artifact_name}`: 加载资源
- `GET /artifacts`: 列出资源
- `GET /artifacts/{artifact_name}/versions`: 列出版本

---

## 2. A2A 协议服务器

### 概述

A2A (Agent-to-Agent) 协议服务器实现 A2A 标准，支持 Agent 之间的互操作。

### 核心文件

```
server/adka2a/
├── processor.go        # 事件处理器
├── executor.go       # 执行器
├── events.go         # 事件处理
├── agent_card.go     # Agent 卡片
├── metadata.go       # 元数据
└── input_required.go # 输入请求处理
```

### eventProcessor 事件处理器

**位置**: `server/adka2a/processor.go:34-54`

```go
type eventProcessor struct {
    reqCtx        *a2asrv.RequestContext
    meta          invocationMeta
    partConverter GenAIPartConverter
    
    // 跟踪升级和代理转移
    terminalActions session.EventActions
    
    // 失败事件（延迟发送）
    failedEvent *a2a.TaskStatusUpdateEvent
    
    // 输入请求处理器
    inputRequiredProcessor *inputRequiredProcessor
    
    // 事件到资源的转换
    eventToArtifact eventToArtifactTransform
}
```

### process() 处理流程

**位置**: `server/adka2a/processor.go:71-113`

```go
func (p *eventProcessor) process(ctx context.Context, event *session.Event) (*a2a.TaskArtifactUpdateEvent, error) {
    if event == nil {
        return nil, nil
    }
    
    // 1. 更新终端操作
    p.updateTerminalActions(event)
    
    // 2. 获取事件元数据
    eventMeta, err := toEventMeta(p.meta, event)
    
    // 3. 处理错误响应
    resp := event.LLMResponse
    if resp.ErrorCode != "" || resp.ErrorMessage != "" {
        if p.failedEvent == nil {
            p.failedEvent = toTaskFailedUpdateEvent(...)
        }
    }
    
    // 4. 处理输入请求
    event, err = p.inputRequiredProcessor.process(event)
    
    // 5. 转换 Parts
    parts, err := p.convertParts(ctx, event)
    if len(parts) == 0 {
        return nil, nil
    }
    
    // 6. 转换为 A2A 资源
    result, err := p.eventToArtifact.transform(event, parts, eventMeta)
    
    return result, nil
}
```

### A2A 事件转换

```
ADK Event
    │
    ├─ LLMResponse
    │   ├─ Content.Parts
    │   │   ├─ Text
    │   │   ├─ FunctionCall
    │   │   ├─ FunctionResponse
    │   │   └─ ...
    │   │
    │   └─ Partial
    │
    └─ Actions
        ├─ Escalate
        ├─ TransferToAgent
        └─ StateDelta

↓ 转换

A2A TaskEvent
    ├─ TaskState
    │   ├─ working
    │   ├─ input_required
    │   ├─ completed
    │   └─ failed
    │
    ├─ Parts
    │   ├─ TextPart
    │   ├─ DataPart
    │   └─ ...
    │
    └─ Metadata
        ├─ escalation
        ├─ agent_transfer
        └─ ...
```

### 终端操作处理

```go
func (p *eventProcessor) updateTerminalActions(event *session.Event) {
    p.terminalActions.Escalate = p.terminalActions.Escalate || event.Actions.Escalate
    if event.Actions.TransferToAgent != "" {
        p.terminalActions.TransferToAgent = event.Actions.TransferToAgent
    }
}
```

### 最终状态更新

```go
func (p *eventProcessor) makeFinalStatusUpdate() *a2a.TaskStatusUpdateEvent {
    // 1. 检查失败事件
    for _, event := range []*a2a.TaskStatusUpdateEvent{p.failedEvent, p.inputRequiredProcessor.event} {
        if event != nil {
            event.Metadata = setActionsMeta(event.Metadata, p.terminalActions)
            return event
        }
    }
    
    // 2. 返回完成状态
    ev := a2a.NewStatusUpdateEvent(p.reqCtx, a2a.TaskStateCompleted, nil)
    ev.Final = true
    ev.Metadata = setActionsMeta(p.meta.eventMeta, p.terminalActions)
    return ev
}
```

---

## 3. 对比分析

### REST API vs A2A

| 特性 | REST API | A2A 协议 |
|------|----------|----------|
| 协议 | HTTP + SSE | A2A 标准 |
| 场景 | 客户端-服务器 | Agent-Agent |
| 事件格式 | ADK Event | A2A TaskEvent |
| 流式 | Server-Sent Events | A2A 流 |
| 状态管理 | Session Service | Task State |

### 使用场景

```go
// REST API 场景
// 客户端应用直接调用 Agent
http.Handle("/runtime/myapp/users/{uid}/sessions/{sid}/run", handler)

// A2A 场景
// Agent 之间相互调用
// Agent A → Agent B (通过 A2A 协议)
```

---

## 4. 事件流处理

### REST API SSE 流

```
HTTP GET /runtime/{app}/users/{user}/sessions/{session}/run
    │
    ├─ 200 OK
    ├─ Content-Type: text/event-stream
    │
    └─ Event: {"event": {...}}
        ├─ data: {"content": "..."}
        ├─ data: {"content": "..."}
        └─ data: {"content": "...", "is_final": true}
```

### A2A 任务流

```
A2A Client → TaskRequest
    │
    └─ A2A Server → Agent.Run()
        │
        ├─ TaskArtifactUpdateEvent (流式)
        │   ├─ working
        │   ├─ input_required
        │   └─ ...
        │
        └─ TaskStatusUpdateEvent (最终)
            ├─ completed
            ├─ failed
            └─ input_required (等待输入)
```

---

## 5. 核心组件

### AgentCard

Agent 能力描述，用于服务发现：

```go
type AgentCard struct {
    Name        string
    Description string
    URL         string
    Capabilities []string
}
```

### invocationMeta 调用元数据

```go
type invocationMeta struct {
    appName     string
    userID      string
    sessionID   string
    taskID      string
}
```

---

## 6. 错误处理

### REST API 错误

```go
// HTTP 状态码
400 // 请求错误
401 // 未授权
404 // 资源不存在
500 // 服务器错误
```

### A2A 错误

```go
// TaskStatusUpdateEvent with TaskStateFailed
{
    "status": {
        "state": "failed",
        "message": {
            "role": "agent",
            "parts": [{"type": "text", "text": "error message"}]
        }
    }
}
```

---

## 7. 集成点

### 与 Runner 集成

```go
// REST API
runner, _ := runner.New(runner.Config{...})
for event, err := range runner.Run(ctx, userID, sessionID, msg, config) {
    // SSE 写出
}

// A2A Server
agent := adka2a.New(agentConfig)
agentServer := a2asrv.New(agentAdapter)
```

### 与 Session Service 集成

```go
// 两种服务都使用 Session Service
sessionService.Get(ctx, &session.GetRequest{...})
sessionService.AppendEvent(ctx, session, event)
```

---

## 8. 部署示例

### 启动 REST API 服务器

```go
import "google.golang.org/adk/server/adkrest"

handler := adkrest.NewHandler(&launcher.Config{
    AppName:         "my_app",
    Agent:          myAgent,
    SessionService:  sessionService,
    ArtifactService: artifactService,
    MemoryService:   memoryService,
}, 30*time.Second)

http.ListenAndServe(":8080", handler)
```

### 启动 A2A 服务器

```go
import "github.com/a2aproject/a2a-go/a2asrv"

agentAdapter := adka2a.NewAgentAdapter(myAgent, sessionService)
server := a2asrv.New(agentAdapter)

server.ListenAndServe(":8081")
```

---

## 总结

ADK Go 的部署服务具有以下特点：

1. **REST API 服务器**
   - 基于 Gorilla Mux 路由器
   - SSE 流式事件
   - 完整的 CRUD 操作

2. **A2A 协议服务器**
   - 实现 A2A 标准
   - Agent 间互操作
   - 任务状态管理

3. **共同特点**
   - 复用 Session Service
   - 事件流处理
   - Telemetry 集成
