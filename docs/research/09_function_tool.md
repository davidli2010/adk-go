# ADK Go FunctionTool 实现详解

## 概述

FunctionTool 是 ADK Go 中将 Go 函数包装为可调用工具的核心实现。本文档深入分析 `tool/functiontool/functiontool.go` 的实现细节。

---

## 核心结构

### Config 配置结构

**位置**: `tool/functiontool/function.go:36-67`

```go
type Config struct {
    Name string
    Description string
    InputSchema *jsonschema.Schema
    OutputSchema *jsonschema.Schema
    IsLongRunning bool
    RequireConfirmation bool
    RequireConfirmationProvider any
}
```

**字段说明**:

| 字段 | 类型 | 用途 |
|------|------|------|
| `Name` | string | 工具名称 |
| `Description` | string | 工具描述，供 LLM 理解工具用途 |
| `InputSchema` | *jsonschema.Schema | 输入参数模式（可选，自动推断） |
| `OutputSchema` | *jsonschema.Schema | 输出结果模式（可选，自动推断） |
| `IsLongRunning` | bool | 标记为长运行操作 |
| `RequireConfirmation` | bool | 强制要求用户确认 |
| `RequireConfirmationProvider` | any | 动态确认提供者函数 |

### Func 类型定义

**位置**: `tool/functiontool/function.go:69-71`

```go
type Func[TArgs, TResults any] func(tool.Context, TArgs) (TResults, error)
```

**特点**:
- 泛型函数类型，支持任意输入输出类型
- 参数：`tool.Context` + `TArgs`
- 返回：`TResults` + `error`

### functionTool 结构

**位置**: `tool/functiontool/function.go:122-136`

```go
type functionTool[TArgs, TResults any] struct {
    cfg Config
    
    inputSchema *jsonschema.Resolved
    outputSchema *jsonschema.Resolved
    
    handler Func[TArgs, TResults]
    
    requireConfirmation bool
    requireConfirmationProvider func(TArgs) bool
}
```

---

## New() 构造函数

**位置**: `tool/functiontool/function.go:78-119`

### 方法签名

```go
func New[TArgs, TResults any](cfg Config, handler Func[TArgs, TResults]) (tool.Tool, error)
```

### 创建流程

```go
func New[TArgs, TResults any](cfg Config, handler Func[TArgs, TResults]) (tool.Tool, error) {
    // 1. 验证输入类型
    var zeroArgs TArgs
    argsType := reflect.TypeOf(zeroArgs)
    for argsType != nil && argsType.Kind() == reflect.Ptr {
        argsType = argsType.Elem()
    }
    if argsType == nil || (argsType.Kind() != reflect.Struct && argsType.Kind() != reflect.Map) {
        return nil, fmt.Errorf("input must be a struct or a map or a pointer to those types...")
    }
    
    // 2. 解析输入模式
    ischema, err := resolvedSchema[TArgs](cfg.InputSchema)
    if err != nil {
        return nil, fmt.Errorf("failed to infer input schema: %w", err)
    }
    
    // 3. 解析输出模式
    oschema, err := resolvedSchema[TResults](cfg.OutputSchema)
    if err != nil {
        return nil, fmt.Errorf("failed to infer output schema: %w", err)
    }
    
    // 4. 解析确认提供者
    var confirmWrapper func(TArgs) bool
    if cfg.RequireConfirmationProvider != nil {
        fn, ok := cfg.RequireConfirmationProvider.(func(TArgs) bool)
        if !ok {
            return nil, fmt.Errorf("error RequireConfirmationProvider must be a function...")
        }
        confirmWrapper = fn
    }
    
    // 5. 创建工具实例
    return &functionTool[TArgs, TResults]{
        cfg:                         cfg,
        inputSchema:                 ischema,
        outputSchema:                oschema,
        handler:                     handler,
        requireConfirmation:         cfg.RequireConfirmation,
        requireConfirmationProvider: confirmWrapper,
    }, nil
}
```

### 类型验证

```go
// 输入类型必须是 struct 或 map 或其指针
if argsType.Kind() != reflect.Struct && argsType.Kind() != reflect.Map {
    return nil, fmt.Errorf("input must be a struct or a map...")
}
```

---

## 工具接口实现

### Name() 名称

**位置**: `tool/functiontool/function.go:144-146`

```go
func (f *functionTool[TArgs, TResults]) Name() string {
    return f.cfg.Name
}
```

### Description() 描述

**位置**: `tool/functiontool/function.go:139-141`

```go
func (f *functionTool[TArgs, TResults]) Description() string {
    return f.cfg.Description
}
```

### IsLongRunning() 长运行标记

**位置**: `tool/functiontool/function.go:149-151`

```go
func (f *functionTool[TArgs, TResults]) IsLongRunning() bool {
    return f.cfg.IsLongRunning
}
```

### ProcessRequest() 请求处理

**位置**: `tool/functiontool/function.go:154-156`

```go
func (f *functionTool[TArgs, TResults]) ProcessRequest(ctx tool.Context, req *model.LLMRequest) error {
    return toolutils.PackTool(req, f)
}
```

**用途**: 将工具声明打包到 LLM 请求中

---

## Declaration() 工具声明

**位置**: `tool/functiontool/function.go:159-181`

### 方法签名

```go
func (f *functionTool[TArgs, TResults]) Declaration() *genai.FunctionDeclaration
```

### 实现逻辑

```go
func (f *functionTool[TArgs, TResults]) Declaration() *genai.FunctionDeclaration {
    decl := &genai.FunctionDeclaration{
        Name:        f.Name(),
        Description: f.Description(),
    }
    
    // 设置输入/输出 JSON Schema
    if f.inputSchema != nil {
        decl.ParametersJsonSchema = f.inputSchema.Schema()
    }
    if f.outputSchema != nil {
        decl.ResponseJsonSchema = f.outputSchema.Schema()
    }
    
    // 长运行操作说明
    if f.cfg.IsLongRunning {
        instruction := "NOTE: This is a long-running operation. Do not call this tool again if it has already returned some intermediate or pending status."
        if decl.Description != "" {
            decl.Description += "\n\n" + instruction
        } else {
            decl.Description = instruction
        }
    }
    
    return decl
}
```

### 输出结构

```go
type FunctionDeclaration struct {
    Name        string
    Description string
    ParametersJsonSchema map[string]any
    ResponseJsonSchema map[string]any
}
```

---

## Run() 工具执行

**位置**: `tool/functiontool/function.go:184-246`

### 方法签名

```go
func (f *functionTool[TArgs, TResults]) Run(ctx tool.Context, args any) (result map[string]any, err error)
```

### 完整执行流程

```go
func (f *functionTool[TArgs, TResults]) Run(ctx tool.Context, args any) (result map[string]any, err error) {
    // 1. Panic 恢复
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic in tool %q: %v\nstack: %s", f.Name(), r, debug.Stack())
        }
    }()
    
    // 2. 参数类型验证
    m, ok := args.(map[string]any)
    if !ok {
        return nil, fmt.Errorf("unexpected args type, got: %T", args)
    }
    
    // 3. 参数转换
    input, err := typeutil.ConvertToWithJSONSchema[map[string]any, TArgs](m, f.inputSchema)
    if err != nil {
        return nil, err
    }
    
    // 4. 用户确认检查
    if confirmation := ctx.ToolConfirmation(); confirmation != nil {
        if !confirmation.Confirmed {
            return nil, fmt.Errorf("error tool %q call is rejected", f.Name())
        }
    } else {
        requireConfirmation := f.requireConfirmation
        
        // 动态确认检查
        if f.requireConfirmationProvider != nil {
            requireConfirmation = f.requireConfirmationProvider(input)
        }
        
        if requireConfirmation {
            err := ctx.RequestConfirmation(
                fmt.Sprintf("Please approve or reject the tool call %s()...", f.Name()), nil)
            if err != nil {
                return nil, err
            }
            ctx.Actions().SkipSummarization = true
            return nil, fmt.Errorf("error tool %q requires confirmation...", f.Name())
        }
    }
    
    // 5. 执行处理函数
    output, err := f.handler(ctx, input)
    if err != nil {
        return nil, err
    }
    
    // 6. 结果转换
    resp, err := typeutil.ConvertToWithJSONSchema[TResults, map[string]any](output, f.outputSchema)
    if err == nil {
        return resp, nil
    }
    
    // 7. 结果包装（非字典类型）
    if f.outputSchema != nil {
        if err1 := f.outputSchema.Validate(output); err1 != nil {
            return resp, err
        }
    }
    wrappedOutput := map[string]any{"result": output}
    return wrappedOutput, nil
}
```

### 执行流程图

```
Run()
    │
    ├─ 1. Panic 恢复
    │
    ├─ 2. 参数类型验证
    │   └─ args 必须是 map[string]any
    │
    ├─ 3. 参数转换
    │   └─ map[string]any → TArgs
    │
    ├─ 4. 用户确认检查
    │   ├─ 已确认 → 继续
    │   ├─ 未确认 + RequireConfirmation → 请求确认
    │   └─ SkipSummarization = true
    │
    ├─ 5. 执行处理函数
    │   └─ f.handler(ctx, input)
    │
    └─ 6. 结果转换
        ├─ TResults → map[string]any
        ├─ 验证失败 → 包装为 {"result": output}
        └─ 返回
```

---

## 参数处理

### 类型验证

```go
m, ok := args.(map[string]any)
if !ok {
    return nil, fmt.Errorf("unexpected args type, got: %T", args)
}
```

### 类型转换

```go
input, err := typeutil.ConvertToWithJSONSchema[map[string]any, TArgs](m, f.inputSchema)
```

**功能**:
- 将 `map[string]any` 转换为目标类型 `TArgs`
- 支持 JSON Schema 验证
- 自动处理嵌套结构

---

## 用户确认机制

### 确认检查流程

```go
if confirmation := ctx.ToolConfirmation(); confirmation != nil {
    // 用户已响应确认请求
    if !confirmation.Confirmed {
        return nil, fmt.Errorf("error tool %q call is rejected", f.Name())
    }
} else {
    // 检查是否需要确认
    requireConfirmation := f.requireConfirmation
    
    // 动态确认提供者
    if f.requireConfirmationProvider != nil {
        requireConfirmation = f.requireConfirmationProvider(input)
    }
    
    if requireConfirmation {
        // 请求用户确认
        ctx.RequestConfirmation(...)
        ctx.Actions().SkipSummarization = true
        return nil, fmt.Errorf("error tool %q requires confirmation...")
    }
}
```

### 静态确认

```go
RequireConfirmation bool
```

- 工具级别强制要求确认

### 动态确认

```go
RequireConfirmationProvider func(TArgs) bool
```

- 根据输入参数动态决定
- 优先级高于静态配置

### SkipSummarization

```go
ctx.Actions().SkipSummarization = true
```

- 确认请求后设置
- 跳过 LLM 总结步骤

---

## 结果处理

### 类型转换

```go
resp, err := typeutil.ConvertToWithJSONSchema[TResults, map[string]any](output, f.outputSchema)
```

### 非字典类型包装

```go
if f.outputSchema != nil {
    if err1 := f.outputSchema.Validate(output); err1 != nil {
        return resp, err
    }
}
wrappedOutput := map[string]any{"result": output}
return wrappedOutput, nil
```

**原因**:
- LLM 要求结果为字典格式
- 非字典类型包装为 `{"result": value}`

---

## Schema 推断

### resolvedSchema() 函数

**位置**: `tool/functiontool/function.go:266-276`

```go
func resolvedSchema[T any](override *jsonschema.Schema) (*jsonschema.Resolved, error) {
    if override != nil {
        return override.Resolve(nil)
    }
    schema, err := jsonschema.For[T](nil)
    if err != nil {
        return nil, err
    }
    return schema.Resolve(nil)
}
```

### 推断逻辑

```
override != nil → 使用 override
override == nil → 使用 jsonschema.For[T]()
```

### 自动推断支持

- **Struct**: 字段名和类型自动映射
- **Map**: key/value 类型自动推断
- **指针**: 自动解引用

---

## Panic 恢复

**位置**: `tool/functiontool/function.go:186-190`

```go
defer func() {
    if r := recover(); r != nil {
        err = fmt.Errorf("panic in tool %q: %v\nstack: %s", f.Name(), r, debug.Stack())
    }
}()
```

**用途**:
- 捕获处理函数中的 panic
- 记录完整堆栈信息
- 转换为错误返回

---

## 使用示例

### 基本工具

```go
type SearchArgs struct {
    Query string `json:"query"`
}

type SearchResult struct {
    Results []string `json:"results"`
}

func searchHandler(ctx tool.Context, args SearchArgs) (SearchResult, error) {
    results, err := performSearch(args.Query)
    return SearchResult{Results: results}, err
}

tool, err := functiontool.New[SearchArgs, SearchResult](
    functiontool.Config{
        Name:        "search",
        Description: "Search for information",
    },
    searchHandler,
)
```

### 带确认的工具

```go
tool, err := functiontool.New[DeleteArgs, DeleteResult](
    functiontool.Config{
        Name:                   "delete",
        Description:            "Delete a file",
        RequireConfirmation:    true,
    },
    deleteHandler,
)
```

### 动态确认

```go
tool, err := functiontool.New[DeleteArgs, DeleteResult](
    functiontool.Config{
        Name: "delete",
        RequireConfirmationProvider: func(args DeleteArgs) bool {
            return args.Force  // 仅当 Force=true 时要求确认
        },
    },
    deleteHandler,
)
```

### 长运行工具

```go
tool, err := functiontool.New[ProcessArgs, ProcessResult](
    functiontool.Config{
        Name:         "process",
        IsLongRunning: true,
    },
    longRunningHandler,
)
```

---

## 与 tool.Tool 接口的关系

```go
type Tool interface {
    Name() string
    Description() string
    IsLongRunning() bool
    ProcessRequest(ctx Context, req *model.LLMRequest) error
    Run(ctx Context, args any) (map[string]any, error)
}
```

**实现对照**:

| 接口方法 | FunctionTool 实现 |
|----------|-------------------|
| Name() | `cfg.Name` |
| Description() | `cfg.Description` |
| IsLongRunning() | `cfg.IsLongRunning` |
| ProcessRequest() | `toolutils.PackTool()` |
| Run() | `functionTool.Run()` |

---

## 关键设计决策

### 1. 泛型函数类型

**选择**: `Func[TArgs, TResults any]`

**优势**:
- 编译时类型安全
- 自动类型推断
- 无需反射调用

### 2. Schema 自动推断

**设计**: 支持手动覆盖 + 自动推断

**优势**:
- 简单场景无需配置
- 复杂场景可精细控制

### 3. Panic 恢复

**设计**: defer + recover

**优势**:
- 防止单个工具崩溃整个系统
- 保留堆栈信息便于调试

### 4. 确认机制

**设计**: 静态 + 动态两种方式

**优势**:
- 简单场景使用静态配置
- 复杂场景使用动态判断

### 5. 结果包装

**设计**: 非字典结果包装为 `{"result": value}`

**原因**: LLM 要求统一字典格式

---

## 总结

FunctionTool 是 ADK Go 工具系统的核心，具有以下特点：

1. **泛型设计**: 编译时类型安全
2. **Schema 推断**: 自动 + 手动覆盖
3. **确认机制**: 静态/动态用户确认
4. **Panic 恢复**: 健壮的错误处理
5. **结果转换**: 自动格式转换和包装
6. **长运行支持**: 专门的状态标记

设计优势：
- 类型安全的函数包装
- 灵活的确认机制
- 完整的错误处理
- 简洁的 API 设计
