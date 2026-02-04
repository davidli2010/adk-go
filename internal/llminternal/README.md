# internal/llminternal 模块设计文档

## 概述

`internal/llminternal` 是 ADK Go 的核心模块，负责 LLM Agent 的运行时编排。该模块采用**管道模式**（Pipeline Pattern）与**回调钩子机制**（Callback Hooks）相结合的设计，实现了请求预处理、模型调用、响应后处理、工具执行等完整流程。

---

## 目录结构

```
internal/llminternal/
├── agent.go                      # Agent 状态定义
├── base_flow.go                   # Flow 核心编排引擎
├── basic_processor.go             # 基础配置处理器
├── tools_processor.go             # 工具处理器
├── contents_processor.go          # 会话历史处理器
├── instruction_processor.go       # 指令处理器
├── outputschema_processor.go      # 结构化输出处理器
├── agent_transfer.go              # 多 Agent 转移处理器
├── request_confirmation_processor.go  # 工具确认处理器
├── file_uploads_processor.go      # 文件上传处理器
├── stream_aggregator.go           # 流式响应聚合器
├── functions.go                  # 工具确认生成器
├── other_processors.go            # 待实现的处理器占位符
├── converters/
│   └── converters.go             # 响应类型转换
├── googlellm/
│   ├── variant.go                 # Gemini API 变体检测
│   └── variant_test.go
└── *_test.go                     # 单元测试文件
```

---

## 核心架构

### 1. 数据结构

#### State 结构体（agent.go:30-52）

```go
type State struct {
    Model model.LLM
    Tools    []tool.Tool
    Toolsets []tool.Toolset
    IncludeContents string
    GenerateContentConfig *genai.GenerateContentConfig
    Instruction               string
    InstructionProvider       InstructionProvider
    GlobalInstruction         string
    GlobalInstructionProvider InstructionProvider
    DisallowTransferToParent bool
    DisallowTransferToPeers  bool
    InputSchema  *genai.Schema
    OutputSchema *genai.Schema
    OutputKey string
}
```

#### Flow 结构体（base_flow.go:56-68）

```go
type Flow struct {
    Model model.LLM
    Tools []tool.Tool
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

---

## 处理管道设计

### 1. 请求处理链（Request Processing Pipeline）

`DefaultRequestProcessors`（base_flow.go:70-88）按顺序执行：

| 顺序 | 处理器 | 功能 |
|------|--------|------|
| 1 | `basicRequestProcessor` | 填充 LLM 生成配置 |
| 2 | `toolProcessor` | 加载 Agent 工具 |
| 3 | `authPreprocessor` | 认证预处理 |
| 4 | `RequestConfirmationRequestProcessor` | 请求确认 |
| 5 | `instructionsRequestProcessor` | 指令处理 |
| 6 | `identityRequestProcessor` | 身份处理 |
| 7 | `ContentsRequestProcessor` | 会话历史构建 |
| 8 | `nlPlanningRequestProcessor` | 自然语言规划 |
| 9 | `codeExecutionRequestProcessor` | 代码执行 |
| 10 | `outputSchemaRequestProcessor` | 输出模式处理 |
| 11 | `AgentTransferRequestProcessor` | Agent 转移 |
| 12 | `removeDisplayNameIfExists` | 清理显示名 |

### 2. 响应处理链（Response Processing Pipeline）

`DefaultResponseProcessors`（base_flow.go:89-92）：

- `nlPlanningResponseProcessor`: 处理规划响应
- `codeExecutionResponseProcessor`: 处理代码执行响应

### 3. 处理器接口设计

每个处理器遵循 `iter.Seq2[*session.Event, error]` 模式，支持：

- 生成中间事件（如认证确认）
- 返回错误中断流程
- 无事件直接继续处理

---

## 核心执行流程

### 主循环 Run()（base_flow.go:95-121）

```go
func (f *Flow) Run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error]
```

**执行流程**：
1. 调用 `runOneStep()` 执行单步推理
2. 检查是否收到最终响应（`IsFinalResponse()`）
3. 若为部分响应且非流式结束，返回错误
4. 循环直至收到最终响应

### 单步执行 runOneStep()（base_flow.go:123-243）

```text
┌─────────────────────────────────────────────────────────────┐
│                     runOneStep()                            │
├─────────────────────────────────────────────────────────────┤
│  1. 检查 Model 配置                                          │
│  2. 构建 LLMRequest                                          │
│  3. preprocess() → RequestProcessors                        │
│  4. callLLM() → BeforeModelCallbacks → Model.GenerateContent│
│     → AfterModelCallbacks / OnModelErrorCallbacks           │
│  5. postprocess() → ResponseProcessors                      │
│  6. 生成 ModelResponseEvent                                  │
│  7. handleFunctionCalls() → 工具执行                         │
│  8. 工具确认请求（若需要）                                     │
│  9. 结构化输出处理（OutputSchema）                            │
│ 10. Agent Transfer 处理                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 回调钩子系统

### 模型回调（base_flow.go:44-54）

| 回调类型 | 触发时机 | 用途 |
|---------|---------|------|
| `BeforeModelCallback` | 调用模型前 | 请求拦截、修改 |
| `AfterModelCallback` | 模型响应后 | 响应拦截、修改 |
| `OnModelErrorCallback` | 模型调用出错 | 错误恢复 |

### 工具回调（base_flow.go:50-54）

| 回调类型 | 触发时机 | 用途 |
|---------|---------|------|
| `BeforeToolCallback` | 工具执行前 | 参数验证、修改 |
| `AfterToolCallback` | 工具执行后 | 结果拦截、修改 |
| `OnToolErrorCallback` | 工具执行出错 | 错误恢复 |

---

## 处理器详细设计

### 1. 内容处理器 ContentsRequestProcessor（contents_processor.go）

**核心职责**：构建 LLM 请求的 `Contents` 字段（会话历史）

**处理策略**：
1. **事件过滤**：
   - 跳过无内容/角色的空事件
   - 过滤认证事件（`isAuthEvent`）
   - 过滤跨分支事件（`eventBelongsToBranch`）
   - 转换其他 Agent 的事件为用户上下文（`ConvertForeignEvent`）

2. **事件重排**：
   - `rearrangeEventsForLatestFunctionResponse`: 处理异步工具调用的响应合并
   - `rearrangeEventsForFunctionResponsesInHistory`: 清理整个事件历史

3. **IncludeContents 策略**：
   - `buildContentsDefault`: 完整历史
   - `buildContentsCurrentTurnContextOnly`: 仅当前轮次

### 2. 指令处理器 instructionsRequestProcessor（instruction_processor.go）

**功能**：
- 注入全局指令（`GlobalInstruction` / `GlobalInstructionProvider`）
- 注入 Agent 指令（`Instruction` / `InstructionProvider`）

**模板变量替换**（`InjectSessionState`）：
- 支持 `{variable}` 语法
- 支持 `{artifact.file_name}` 加载Artifacts
- 支持 `{app:name}`, `{user:name}`, `{temp:name}` 前缀
- 支持 `{variable?}` 可选变量

```go
// 示例指令
// You are {app:assistant_name}. Current user: {user:name}
```

### 3. 工具处理器 toolProcessor（tools_processor.go）

**职责**：从 Agent 配置中提取工具集（Tools + Toolsets）

```go
func toolProcessor(ctx agent.InvocationContext, req *model.LLMRequest, f *Flow) iter.Seq2[*session.Event, error]
```

### 4. 代理转移处理器 AgentTransferRequestProcessor（agent_transfer.go）

**职责**：为多 Agent 系统添加 `transfer_to_agent` 工具

**转移方向**：
1. Parent → Sub-agent
2. Sub-agent → Parent
3. Sub-agent → Peer agents（需满足条件）

```go
func transferTargets(agent, parent agent.Agent) []agent.Agent
```

### 5. 输出模式处理器 outputSchemaRequestProcessor（outputschema_processor.go）

**职责**：当模型不支持原生工具+OutputSchema组合时，提供 workaround

**检测逻辑**：
```go
func needOutputSchemaProcessor(state *State) bool {
    hasTools := len(state.Tools) > 0 || len(state.Toolsets) > 0
    canUseOutputSchemaWithTools := googlellm.CanGeminiModelUseOutputSchemaWithTools(state.Model.Name())
    return hasTools && !canUseOutputSchemaWithTools
}
```

**解决方案**：添加 `set_model_response` 工具，强制模型通过工具调用返回结构化数据。

### 6. 工具确认处理器 RequestConfirmationRequestProcessor（request_confirmation_processor.go）

**职责**：处理工具执行前的用户确认流程

**流程**：
1. 查找用户确认事件（`adk_request_credential`）
2. 提取确认信息
3. 重启被暂停的工具调用

### 7. 文件上传处理器 file_uploads_processor.go

**职责**：清理 Gemini API 不支持的 `displayName` 参数

```go
func removeDisplayNameIfExists(ctx agent.InvocationContext, req *model.LLMRequest, f *Flow) iter.Seq2[*session.Event, error]
```

---

## 流式响应聚合

### streamingResponseAggregator（stream_aggregator.go）

**设计目的**：处理 LLM 流式输出，将多个部分响应聚合成完整响应

**聚合策略**：
1. **文本累积**：持续累积 `text` 和 `thoughtText`
2. **边界检测**：当下个响应为空/音频时，生成聚合事件
3. **最终闭合**：`Close()` 方法在流结束时生成最终聚合

```go
type streamingResponseAggregator struct {
    text        string
    thoughtText string
    response    *model.LLMResponse
    role        string
}
```

---

## 工具执行系统

### 工具调用入口 handleFunctionCalls（base_flow.go:488-555）

**流程**：
1. 提取函数调用（`utils.FunctionCalls`）
2. 构建工具上下文（`toolinternal.NewToolContext`）
3. 查找工具（支持 `FunctionTool` 接口）
4. 执行 `callTool()` 方法
5. 合并并行响应（`mergeParallelFunctionResponseEvents`）

### 工具调用链 callTool（base_flow.go:568-613）

```text
callTool()
├── BeforeToolCallback（插件）
├── BeforeToolCallback（Flow）
├── tool.Run()
├── OnToolErrorCallback（插件）
├── OnToolErrorCallback（Flow）
├── AfterToolCallback（插件）
└── AfterToolCallback（Flow）
```

### 工具错误处理

**工具未找到错误**（`newToolNotFoundError`）：
```go
func newToolNotFoundError(toolName string, availableTools []string) error {
    return fmt.Errorf(`tool '%s' not found.
Available tools: %s
Possible causes:
  1. LLM hallucinated the function name
  2. Tool not registered
  3. Name mismatch`, toolName, joinedTools)
}
```

---

## 工具确认系统

### generateRequestConfirmationEvent（functions.go）

**触发场景**：工具执行前需要用户确认时

**流程**：
1. 从 `functionResponseEvent.Actions.RequestedToolConfirmations` 提取确认请求
2. 生成 `adk_request_confirmation` 函数调用
3. 返回确认请求事件

---

## 设计模式总结

| 模式 | 应用场景 | 示例 |
|------|---------|------|
| **管道模式** | 请求/响应处理链 | `RequestProcessors`, `ResponseProcessors` |
| **策略模式** | 不同处理策略 | `buildContentsDefault` vs `buildContentsCurrentTurnContextOnly` |
| **观察者模式** | 回调钩子 | `BeforeModelCallback`, `BeforeToolCallback` |
| **责任链模式** | 错误处理回调 | `invokeOnToolErrorCallbacks` |
| **迭代器模式** | 流式处理 | `iter.Seq2[*session.Event, error]` |
| **组合模式** | 事件合并 | `mergeParallelFunctionResponseEvents` |

---

## 外部依赖

- `google.golang.org/adk/agent`: Agent 接口
- `google.golang.org/adk/model`: LLM 请求/响应
- `google.golang.org/adk/session`: 事件模型
- `google.golang.org/adk/tool`: 工具接口
- `google.golang.org/genai`: Google AI 协议

---

## 设计亮点

1. **可扩展性**：通过 `RequestProcessors` 和 `Callbacks` 支持外部扩展
2. **可观测性**：集成 `telemetry` 追踪 LLM 调用和工具执行
3. **健壮性**：
   - 工具未找到时的详细错误提示
   - 异步工具调用的响应合并
   - 流式响应的边界处理
4. **一致性**：参考 adk-python 实现，保持跨语言行为一致
5. **多 Agent 支持**：完整的 Agent 转移机制

---

## 待实现功能（other_processors.go）

以下处理器当前为空实现，待后续开发：

- `identityRequestProcessor`: 身份处理器
- `nlPlanningRequestProcessor`: 自然语言规划请求处理器
- `codeExecutionRequestProcessor`: 代码执行请求处理器
- `authPreprocessor`: 认证预处理器
- `nlPlanningResponseProcessor`: 自然语言规划响应处理器
- `codeExecutionResponseProcessor`: 代码执行响应处理器
