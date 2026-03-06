# ADK Go 插件系统详解

## 概述

本文档深入分析 ADK Go 的插件系统，包括插件配置、回调类型和插件管理器实现。

---

## 核心结构

### Config 配置结构

**位置**: `plugin/plugin.go:26-48`

```go
type Config struct {
    Name string
    
    // 用户消息回调
    OnUserMessageCallback OnUserMessageCallback
    
    // 事件回调
    OnEventCallback OnEventCallback
    
    // 运行回调
    BeforeRunCallback BeforeRunCallback
    AfterRunCallback  AfterRunCallback
    
    // Agent 回调
    BeforeAgentCallback agent.BeforeAgentCallback
    AfterAgentCallback  agent.AfterAgentCallback
    
    // Model 回调
    BeforeModelCallback  llmagent.BeforeModelCallback
    AfterModelCallback   llmagent.AfterModelCallback
    OnModelErrorCallback llmagent.OnModelErrorCallback
    
    // Tool 回调
    BeforeToolCallback  llmagent.BeforeToolCallback
    AfterToolCallback   llmagent.AfterToolCallback
    OnToolErrorCallback llmagent.OnToolErrorCallback
    
    // 关闭函数
    CloseFunc func() error
}
```

---

## 回调类型定义

### 用户消息回调

**位置**: `plugin/plugin.go:161`

```go
type OnUserMessageCallback func(agent.InvocationContext, *genai.Content) (*genai.Content, error)
```

**用途**: 用户消息预处理，可修改或拦截用户输入

### 运行回调

**位置**: `plugin/plugin.go:163-165`

```go
type BeforeRunCallback func(agent.InvocationContext) (*genai.Content, error)
type AfterRunCallback func(agent.InvocationContext)
```

**用途**:
- BeforeRunCallback: Agent 执行前，可提前返回响应
- AfterRunCallback: Agent 执行后，清理和收尾

### 事件回调

**位置**: `plugin/plugin.go:167`

```go
type OnEventCallback func(agent.InvocationContext, *session.Event) (*session.Event, error)
```

**用途**: 事件后处理，可修改或拦截事件

---

## Plugin 结构

**位置**: `plugin/plugin.go:78-99`

```go
type Plugin struct {
    name string
    
    onUserMessageCallback OnUserMessageCallback
    onEventCallback       OnEventCallback
    
    beforeRunCallback BeforeRunCallback
    afterRunCallback  AfterRunCallback
    
    beforeAgentCallback agent.BeforeAgentCallback
    afterAgentCallback  agent.AfterAgentCallback
    
    beforeModelCallback  llmagent.BeforeModelCallback
    afterModelCallback   llmagent.AfterModelCallback
    onModelErrorCallback llmagent.OnModelErrorCallback
    
    beforeToolCallback  llmagent.BeforeToolCallback
    afterToolCallback   llmagent.AfterToolCallback
    onToolErrorCallback llmagent.OnToolErrorCallback
    
    closeFunc func() error
}
```

---

## 回调执行顺序

### 完整执行链

```
Runner.Run()
    │
    ├─ 1. OnUserMessageCallback
    │   └─ 追加用户消息到会话前
    │
    ├─ 2. BeforeRunCallback
    │   └─ Agent 执行前
    │
    └─ Agent.Run()
        │
        ├─ 3. BeforeAgentCallback
        │   └─ Agent 执行前
        │
        ├─ 4. LLM 调用
        │   ├─ 4.1 BeforeModelCallback
        │   ├─ 4.2 generateContent()
        │   ├─ 4.3 OnModelErrorCallback
        │   └─ 4.4 AfterModelCallback
        │
        ├─ 5. Tool 调用
        │   ├─ 5.1 BeforeToolCallback
        │   ├─ 5.2 tool.Run()
        │   ├─ 5.3 OnToolErrorCallback
        │   └─ 5.4 AfterToolCallback
        │
        └─ 6. AfterAgentCallback
            └─ Agent 执行后
    
    └─ 7. AfterRunCallback
        └─ Runner 执行后（延迟执行）
    
    └─ 8. OnEventCallback
        └─ 每个事件产出后
```

---

## PluginManager 插件管理器

### 结构定义

**位置**: `internal/plugininternal/plugin_manager.go:38-41`

```go
type PluginManager struct {
    plugins      []*plugin.Plugin
    closeTimeout time.Duration
}
```

### 创建 PluginManager

**位置**: `internal/plugininternal/plugin_manager.go:44-59`

```go
func NewPluginManager(cfg PluginConfig) (*PluginManager, error) {
    pm := &PluginManager{
        closeTimeout: cfg.CloseTimeout,
        plugins:      make([]*plugin.Plugin, 0, len(cfg.Plugins)),
    }
    
    // 注册插件
    for _, p := range cfg.Plugins {
        err := pm.registerPlugin(p)
        if err != nil {
            return nil, err
        }
    }
    
    return pm, nil
}
```

### 注册插件

**位置**: `internal/plugininternal/plugin_manager.go:62-73`

```go
func (pm *PluginManager) registerPlugin(plugin *plugin.Plugin) error {
    if plugin == nil {
        return fmt.Errorf("cannot register nil plugin")
    }
    for _, p := range pm.plugins {
        if p.Name() == plugin.Name() {
            return fmt.Errorf("plugin with name '%s' already registered", plugin.Name())
        }
    }
    pm.plugins = append(pm.plugins, plugin)
    return nil
}
```

**验证**:
- 不能注册 nil 插件
- 插件名称不能重复

---

## 回调执行机制

### 顺序执行

所有插件回调按注册顺序依次执行：

```go
func (pm *PluginManager) RunBeforeRunCallback(cctx agent.InvocationContext) (*genai.Content, error) {
    for _, plugin := range pm.plugins {
        callback := plugin.BeforeRunCallback()
        if callback != nil {
            newContent, err := callback(cctx)
            if err != nil {
                return nil, err
            }
            if newContent != nil {
                return newContent, nil // 提前退出
            }
        }
    }
    return nil, nil
}
```

### 提前退出机制

```go
if newContent != nil {
    return newContent, nil // 提前退出，跳过后续插件
}
```

**触发条件**:
- 回调返回非 nil 内容 → 跳过后续插件
- 回调返回错误 → 立即返回错误

**用途**:
- 缓存命中时返回缓存响应
- 权限检查失败时拦截
- 限流时返回降级响应

---

## 回调执行方法汇总

### 用户消息

```go
func (pm *PluginManager) RunOnUserMessageCallback(cctx agent.InvocationContext, userMessage *genai.Content) (*genai.Content, error)
```

### 运行

```go
func (pm *PluginManager) RunBeforeRunCallback(cctx agent.InvocationContext) (*genai.Content, error)
func (pm *PluginManager) RunAfterRunCallback(cctx agent.InvocationContext)
```

### 事件

```go
func (pm *PluginManager) RunOnEventCallback(cctx agent.InvocationContext, event *session.Event) (*session.Event, error)
```

### Agent

```go
func (pm *PluginManager) RunBeforeAgentCallback(cctx agent.CallbackContext) (*genai.Content, error)
func (pm *PluginManager) RunAfterAgentCallback(cctx agent.CallbackContext) (*genai.Content, error)
```

### Model

```go
func (pm *PluginManager) RunBeforeModelCallback(cctx agent.CallbackContext, llmRequest *model.LLMRequest) (*model.LLMResponse, error)
func (pm *PluginManager) RunAfterModelCallback(cctx agent.CallbackContext, llmResponse *model.LLMResponse, llmResponseError error) (*model.LLMResponse, error)
func (pm *PluginManager) RunOnModelErrorCallback(cctx agent.CallbackContext, llmRequest *model.LLMRequest, llmResponseError error) (*model.LLMResponse, error)
```

### Tool

```go
func (pm *PluginManager) RunBeforeToolCallback(ctx tool.Context, tool tool.Tool, args map[string]any) (map[string]any, error)
func (pm *PluginManager) RunAfterToolCallback(ctx tool.Context, tool tool.Tool, args, result map[string]any, err error) (map[string]any, error)
func (pm *PluginManager) RunOnToolErrorCallback(ctx tool.Context, tool tool.Tool, args map[string]any, err error) (map[string]any, error)
```

---

## 插件关闭

**位置**: `internal/plugininternal/plugin_manager.go:272-284`

```go
func (pm *PluginManager) Close() error {
    var errors []error
    for _, plugin := range pm.plugins {
        if err := plugin.Close(); err != nil {
            errors = append(errors, fmt.Errorf("error closing plugin '%s': %w", plugin.Name(), err))
        }
    }
    if len(errors) > 0 {
        return fmt.Errorf("failed to close plugins: %v", errors)
    }
    return nil
}
```

**特点**:
- 依次关闭所有插件
- 收集所有关闭错误
- 返回合并的错误信息

---

## 上下文传递

**位置**: `internal/plugininternal/plugin_manager.go:286-288`

```go
func ToContext(ctx context.Context, cfg *PluginManager) context.Context {
    return context.WithValue(ctx, plugincontext.PluginManagerCtxKey, cfg)
}
```

**用途**: 将 PluginManager 注入到 Context 中

---

## 使用示例

### 创建插件

```go
myPlugin, err := plugin.New(plugin.Config{
    Name: "my_plugin",
    
    BeforeModelCallback: func(ctx agent.CallbackContext, req *model.LLMRequest) (*model.LLMResponse, error) {
        log.Printf("Before LLM call: %s", req.Model)
        return nil, nil  // 不干预
    },
    
    AfterModelCallback: func(ctx agent.CallbackContext, resp *model.LLMResponse, err error) (*model.LLMResponse, error) {
        if resp != nil && resp.UsageMetadata != nil {
            log.Printf("Token usage: %d", resp.UsageMetadata.TotalTokenCount)
        }
        return nil, nil  // 不干预
    },
    
    OnToolCallback: func(ctx tool.Context, tool tool.Tool, args map[string]any) (map[string]any, error) {
        log.Printf("Tool %s called with args: %v", tool.Name(), args)
        return nil, nil
    },
    
    CloseFunc: func() error {
        log.Println("Plugin closed")
        return nil
    },
})
```

### 注册到 Runner

```go
runner, err := runner.New(runner.Config{
    AppName: "my_app",
    Agent:   myAgent,
    SessionService: sessionService,
    PluginConfig: runner.PluginConfig{
        Plugins:      []*plugin.Plugin{myPlugin},
        CloseTimeout: 10 * time.Second,
    },
})
```

### 缓存插件示例

```go
cachePlugin, err := plugin.New(plugin.Config{
    Name: "cache",
    
    BeforeModelCallback: func(ctx agent.CallbackContext, req *model.LLMRequest) (*model.LLMResponse, error) {
        key := hashRequest(req)
        if cached, ok := cache.Get(key); ok {
            return cached, nil  // 返回缓存，跳过 LLM
        }
        return nil, nil
    },
    
    AfterModelCallback: func(ctx agent.CallbackContext, resp *model.LLMResponse, err error) (*model.LLMResponse, error) {
        if err == nil && resp != nil {
            key := hashRequest(ctx)  // 需要保存请求
            cache.Set(key, resp)
        }
        return nil, nil
    },
})
```

### 限流插件示例

```go
rateLimitPlugin, err := plugin.New(plugin.Config{
    Name: "rate_limiter",
    
    BeforeRunCallback: func(ctx agent.InvocationContext) (*genai.Content, error) {
        if !limiter.Allow() {
            return &genai.Content{
                Parts: []*genai.Part{{Text: "Rate limit exceeded. Please try again later."}},
            }, nil
        }
        return nil, nil
    },
})
```

---

## 与 Runner 的集成

### Runner 初始化

```go
// 创建 PluginManager
pluginManager, err := plugininternal.NewPluginManager(plugininternal.PluginConfig{
    Plugins:      cfg.PluginConfig.Plugins,
    CloseTimeout: cfg.PluginConfig.CloseTimeout,
})

// 注入到 Context
ctx = plugininternal.ToContext(ctx, pluginManager)

// 延迟关闭
defer pluginManager.Close()
```

### 回调调用位置

| 回调 | 调用位置 |
|------|----------|
| OnUserMessageCallback | `runner.appendMessageToSession()` |
| BeforeRunCallback | `runner.Run()` |
| AfterRunCallback | `runner.Run()` (defer) |
| OnEventCallback | `runner.Run()` |
| BeforeAgentCallback | `agent.Run()` |
| AfterAgentCallback | `agent.Run()` |
| BeforeModelCallback | `base_flow.callLLM()` |
| AfterModelCallback | `base_flow.callLLM()` |
| OnModelErrorCallback | `base_flow.callLLM()` |
| BeforeToolCallback | `base_flow.callTool()` |
| AfterToolCallback | `base_flow.callTool()` |
| OnToolErrorCallback | `base_flow.callTool()` |

---

## 关键设计决策

### 1. 回调链设计

**选择**: 顺序执行 + 提前退出

**优势**:
- 插件可按序协作
- 早期退出优化性能
- 错误可立即传播

### 2. 插件名称唯一性

**选择**: 注册时检查名称重复

**原因**:
- 避免回调执行歧义
- 便于调试和追踪
- 清晰的错误信息

### 3. 空回调处理

**选择**: 检查 callback != nil 后执行

```go
callback := plugin.BeforeRunCallback()
if callback != nil {
    // 执行
}
```

**优势**:
- 允许部分实现
- 无需提供所有回调

### 4. CloseFunc 默认值

**选择**: nil 时提供默认空函数

```go
if p.closeFunc == nil {
    p.closeFunc = func() error { return nil }
}
```

**原因**: 防止 Close() panic

### 5. Context 传递

**选择**: 注入到 context.Value

**优势**:
- 隐式传递，无需修改接口
- 运行时获取，无需构造函数参数

---

## 总结

ADK Go 的插件系统具有以下特点：

1. **全面回调**: 覆盖从用户消息到工具执行的全生命周期
2. **顺序执行**: 按注册顺序执行，支持协作
3. **提前退出**: 回调可中断后续执行
4. **错误处理**: 错误立即传播，跳过后续插件
5. **资源管理**: 提供 Close 方法清理资源
6. **线程安全**: 无状态设计，天然线程安全

设计优势：
- 灵活的扩展机制
- 清晰的执行顺序
- 完整的生命周期管理
- 易于调试和追踪
