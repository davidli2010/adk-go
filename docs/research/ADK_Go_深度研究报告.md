# ADK Go 深度研究报告

## 概述

ADK Go（Agent Development Kit for Go）是 Google 推出的 Go 语言版 Agent 开发框架，为构建智能 Agent 应用提供了完整的工具和接口。本报告基于对 ADK Go 源代码的深入分析，系统性地阐述其核心架构、关键实现和设计理念。

ADK Go 采用模块化设计，核心组件包括：Agent 执行引擎、Flow 循环执行器、LLM 调用管理、工具系统，会话状态管理、插件系统等。该框架大量使用 Go 1.22+ 的泛型迭代器（iter.Seq、iter.Seq2）实现流式响应处理，支持同步和异步执行模式。

---

## 第一部分：核心架构与基础接口

### 1.1 Agent 接口体系

Agent 是 ADK Go 的核心抽象，所有代理实现都遵循统一的接口定义。Agent 接口位于 `agent/agent.go:43`，定义了代理的基本行为和能力。

```go
type Agent interface {
    Name() string
    Description() string
    Run(InvocationContext) iter.Seq2[*session.Event, error]
    SubAgents() []Agent
    internal() *agent
}
```

接口方法说明如下：Name() 方法返回代理的唯一标识名称；Description() 方法返回代理能力描述，LLM 利用此信息决定是否委派任务；Run() 方法是执行入口，返回事件序列和错误；SubAgents() 方法返回子代理列表，支持任务委派和层级结构。

Agent 配置结构（`agent/agent.go:74`）包含名称、描述，子代理列表以及前后置回调函数。Config 的 Run 字段是核心执行函数，类型为 `func(InvocationContext) iter.Seq2[*session.Event, error]`，这使得用户可以自定义 Agent 的执行逻辑。

### 1.2 InvocationContext 调用上下文

InvocationContext 表示代理调用的上下文，贯穿整个请求的生命周期。该接口定义在 `agent/context.go`，继承自 context.Context 并扩展了 Agent 相关的方法。

```go
type InvocationContext interface {
    context.Context
    
    Agent() Agent
    Artifacts() Artifacts
    Memory() Memory
    Session() session.Session
    InvocationID() string
    Branch() string
    UserContent() *genai.Content
    RunConfig() *RunConfig
    EndInvocation()
    Ended() bool
    WithContext(ctx context.Context) InvocationContext
}
```

InvocationContext 的核心方法包括：Agent() 返回当前执行的代理；Session() 返回会话状态；Artifacts() 返回资源文件服务；Memory() 返回长期记忆服务；InvocationID() 返回调用唯一标识；Branch() 返回调用分支路径，格式为 `agent1.agent2.agent3`；UserContent() 返回用户输入内容；RunConfig() 返回运行时配置。

调用层级结构体现了 ADK Go 的设计理念：Invocation（从用户消息开始到最终响应结束）包含多个 Agent Call（由 agent.Run() 处理），每个 Agent Call 包含多个 Step（单次 LLM 调用加上可能的工具调用）。

### 1.3 Session 与 Event 事件溯源

ADK Go 采用事件溯源（Event Sourcing）模式记录所有用户与代理之间的交互。每次交互都生成独立的事件，支持完整的对话历史追踪，回放和审计。

Session 接口（`session/session.go:32`）表示用户与代理之间的对话系列，包含 ID()、AppName()、UserID()、State()、Events() 和 LastUpdateTime() 方法。

State 接口（`session/session.go:51`）定义状态管理能力，提供 Get()、Set() 和 All() 方法。状态键使用前缀区分作用域：`app:` 表示应用级别状态（所有用户和会话共享），`user:` 表示用户级别状态（同一用户的会话共享），`temp:` 表示临时状态（当前调用内有效）。

Event 结构（`session/session.go:92`）记录对话中的每一次交互，包含 LLMResponse 嵌入（继承模型响应）、ID（事件唯一标识）、Timestamp（时间戳）、InvocationID（调用标识）、Branch（分支路径）、Author（事件作者）、Actions（行为记录）和 LongRunningToolIDs（长运行工具 ID 列表）。

EventActions 结构（`session/session.go:142`）包含 StateDelta（会话状态变更）、ArtifactDelta（文件资源版本变更）、RequestedToolConfirmations（需要用户确认的工具）、SkipSummarization（跳过总结标志）、TransferToAgent（代理转移目标名称）和 Escalate（升级到上级代理标志）。

IsFinalResponse() 方法（`session/session.go:120`）判断事件是否为最终响应。满足以下任一条件时返回 true：SkipSummarization 为 true；存在 LongRunningToolIDs；没有 FunctionCall；没有 FunctionResponse；不是 Partial；没有尾随代码执行结果。该方法是 Flow 循环执行的退出条件判断依据。

### 1.4 LLM 模型接口

LLM 接口（`model/llm.go:26`）提供对底层大语言模型的访问。

```go
type LLM interface {
    Name() string
    GenerateContent(ctx context.Context, req *LLMRequest, stream bool) iter.Seq2[*LLMResponse, error]
}
```

LLMRequest 结构（`model/llm.go:31`）包含 Model（模型名称）、Contents（消息内容数组）、Config（生成配置）和 Tools（可用工具映射）。

LLMResponse 结构（`model/llm.go:41`）包含 Content（响应内容）、CitationMetadata（引用元数据）、GroundingMetadata（接地元数据）、UsageMetadata（使用量元数据）、CustomMetadata（自定义元数据）、LogprobsResult（概率结果）、Partial（流式模式下的部分内容标志）、TurnComplete（响应完成标志）、Interrupted（中断标志）、ErrorCode（错误代码）、ErrorMessage（错误消息）、FinishReason（完成原因）和 AvgLogprobs（平均对数概率）。

### 1.5 Tool 工具接口

Tool 接口（`tool/tool.go:30`）定义可调用工具的抽象。

```go
type Tool interface {
    Name() string
    Description() string
    IsLongRunning() bool
}
```

Tool Context 接口（`tool/tool.go:45`）继承 CallbackContext 并扩展了工具相关的方法，包括 FunctionCallID()（函数调用 ID）、Actions()（事件行为）、SearchMemory()（搜索记忆）和 ToolConfirmation()（工具确认）等。

Toolset 接口（`tool/tool.go:97`）表示工具集，可以动态提供工具列表。RunnableTool 接口（`tool/tool.go:225`）继承 Tool 并添加了 Declaration() 方法（返回函数声明）和 Run() 方法（执行工具）。

---

## 第二部分：执行流程与核心引擎

### 2.1 Runner 入口机制

Runner 是 ADK Go 的用户请求入口，负责协调会话管理、Agent 查找、上下文构建和事件输出。Runner 结构（`runner/runner.go:100`）包含 appName、rootAgent、sessionService、artifactService、memoryService、parents（代理父子关系映射）和 pluginManager（插件管理器）。

Runner.Run() 方法（`runner/runner.go:114`）的完整流程如下：首先获取会话，通过 sessionService.Get() 方法获取现有会话或创建新会话；然后查找 Agent，通过 findAgentToRun() 方法确定哪个 Agent 应该处理当前请求，查找逻辑优先处理用户函数调用响应，然后从后向前遍历事件找到最后一个非用户 Agent，检查可转移性，最后回退到根 Agent；接着构建上下文，依次注入 parentmap（代理父子关系）、runconfig（运行配置）和 plugininternal（插件管理器）；随后初始化服务包装，包装 Artifacts 和 Memory 服务并注入会话元数据；之后创建 InvocationContext，包含所有服务和配置信息；然后追加用户消息到会话，执行 OnUserMessageCallback，保存输入为资源文件（可选），创建用户消息事件并追加到会话；再执行插件回调，包括 BeforeRunCallback（Agent 执行前，可修改上下文或提前退出）和 AfterRunCallback（延迟执行，清理和收尾工作）；最后执行 Agent.Run()，遍历事件流，执行 OnEventCallback，持久化非 Partial 事件，yield 产出事件。

findAgentToRun() 方法的查找逻辑体现了 ADK Go 的智能调度能力：优先处理用户函数调用响应（当用户响应工具调用时，回到原 Agent）；从后向前遍历事件，找到最后一个活跃的 Agent；检查 Agent 是否允许跨树转移；最后回退到根 Agent 作为兜底。

### 2.2 Agent 执行入口

Agent.Run() 方法（`agent/agent.go:160`）是所有 Agent 实现的统一执行入口。执行流程包括：启动 Telemetry 追踪（创建 OpenTelemetry Span）；构建 invocationContext（包装上下文，注入追踪）；执行前置回调（runBeforeAgentCallbacks，包括插件回调和用户定义回调）；检查是否提前结束（ctx.Ended()）；执行实际 run 函数（遍历事件流，自动设置 Author）；再次检查结束；执行后置回调（runAfterAgentCallbacks）。

runBeforeAgentCallbacks() 方法（`agent/agent.go:229`）的执行顺序为：插件回调优先，用户回调依次执行，检查状态变更（有状态变更也创建事件）。提前退出机制通过返回非 nil 内容实现，此时会调用 ctx.EndInvocation() 标记调用结束。

callbackContext 结构（`agent/agent.go:352`）实现 CallbackContext 接口，提供状态读写、Artifacts 访问、调用信息获取等能力。callbackContextState 嵌套结构（`agent/agent.go:404`）实现了 State 接口，回调中的状态变更先写入 StateDelta，事件创建时携带状态变更记录，后续持久化到 Session。

### 2.3 LLMAgent 核心实现

LLMAgent 是 ADK Go 中最核心的 Agent 实现，负责与 LLM 模型交互、工具调用和响应生成。LLMAgent 结构（`agent/llmagent/llmagent.go:341`）嵌入 agent.Agent 接口和 llminternal.State，组合了模型、回调、指令等配置。

New() 构造函数（`agent/llmagent/llmagent.go:34`）的创建流程包括：转换回调类型（用户定义转换为内部类型）；创建 llmAgent 实例并设置各种配置；创建基础 Agent 并注入 run 方法；设置内部状态。

Config 配置结构（`agent/llmagent/llmagent.go:115`）包含丰富的配置选项。核心配置字段包括 Name、Description、SubAgents、Model、Tools、Toolsets、Instruction、InstructionProvider、GlobalInstruction、InputSchema、OutputSchema、OutputKey、GenerateContentConfig、DisallowTransferToParent、DisallowTransferToPeers 和 IncludeContents。回调配置字段包括 BeforeAgentCallbacks、AfterAgentCallbacks、BeforeModelCallbacks、AfterModelCallbacks、OnModelErrorCallbacks、BeforeToolCallbacks、AfterToolCallbacks 和 OnToolErrorCallbacks。

LLMAgent.run() 方法（`agent/llmagent/llmagent.go:347`）的执行流程为：更新上下文（确保 Agent 字段指向当前 llmAgent）；创建 Flow 对象（注入 Model、回调、处理器等配置）；执行 Flow.Run() 并产出事件流；可选地将输出保存到 Session State（通过 maybeSaveOutputToState() 方法）。

### 2.4 Flow 执行引擎

Flow 是 ADK Go 的核心执行引擎，负责管理从 LLM 调用到工具执行的完整循环。Flow 结构（`internal/llminternal/base_flow.go:58`）包含 Model、Tools、RequestProcessors、ResponseProcessors 和各种回调链。

Flow.Run() 方法（`internal/llminternal/base_flow.go:97`）实现无限循环执行模式：执行 runOneStep() 获取事件流，yield 每个事件并记录 lastEvent，检查退出条件（lastEvent == nil 或 IsFinalResponse() 返回 true 或 Partial 标志为 true），满足退出条件则退出循环，否则继续执行下一步。

runOneStep() 方法（`internal/llminternal/base_flow.go:125`）的完整执行流程包括：检查 Model 配置；预处理（preprocess）执行 RequestProcessors 链；调用 LLM（callLLM）；后处理（postprocess）执行 ResponseProcessors 链；跳过空响应（代码执行器触发）；构建事件（finalizeModelResponseEvent）；处理函数调用（handleFunctionCalls）；工具确认事件（如需用户确认）；结构化响应处理（OutputSchema）；代理转移处理（TransferToAgent）。

RequestProcessors 链（`internal/llminternal/base_flow.go:73`）包含 12 个预定义处理器，依次执行：basicRequestProcessor（基础请求处理）、toolProcessor（工具处理）、authPreprocessor（认证预处理）、RequestConfirmationRequestProcessor（请求确认）、instructionsRequestProcessor（指令处理）、identityRequestProcessor（身份处理）、 ContentsRequestProcessor（内容处理，对话历史）、nlPlanningRequestProcessor（NL Planning）、codeExecutionRequestProcessor（代码执行）、outputSchemaRequestProcessor（输出模式）、AgentTransferRequestProcessor（代理转移）和 removeDisplayNameIfExists（移除显示名称）。

ResponseProcessors 链包含 2 个预定义处理器：nlPlanningResponseProcessor（NL Planning 后处理）和 codeExecutionResponseProcessor（代码执行后处理）。

### 2.5 callLLM LLM 调用流程

callLLM() 方法（`internal/llminternal/base_flow.go:290`）的调用流程为：执行插件 BeforeModelCallback（如有，返回非 nil 则跳过 LLM）；执行用户 BeforeModelCallbacks（依次执行，任一返回非 nil 则跳过 LLM）；调用 generateContent() 实际调用 LLM；错误时执行 OnModelErrorCallbacks；填充函数调用 ID；执行 AfterModelCallbacks。

generateContent() 函数（`internal/llminternal/base_flow.go:377`）的实现包含：启动追踪 Span；记录请求日志；调用 LLM.GenerateContent()（流式或非流式）；记录响应日志（仅最终响应）；追踪结果并结束 Span。

流式响应处理机制体现了 ADK Go 的设计特点：使用 StreamingResponseAggregator 聚合器（`internal/llminternal/stream_aggregator.go`）；文本累积到 text 或 thoughtText；Partial = true 标记中间响应；空内容时生成最终聚合响应。

### 2.6 工具调用处理

handleFunctionCalls() 方法（`internal/llminternal/base_flow.go:559`）处理 LLM 返回的函数调用。执行流程包括：获取 FunctionCalls 列表；创建合并 Span（多个工具时）；遍历每个 FunctionCall（启动工具执行 Span、创建 toolCtx、查找工具、调用 callTool()、创建 FunctionResponse Event、记录工具执行结果）；合并并行工具响应。

callTool() 方法（`internal/llminternal/base_flow.go:665`）的实际执行流程为：执行插件 BeforeToolCallback；执行用户 BeforeToolCallbacks；执行 tool.Run()（实际工具执行）；错误时执行 OnToolErrorCallbacks；执行 AfterToolCallbacks（始终执行，无论成功/失败）；错误包装。

工具未找到时（`internal/llminternal/base_flow.go:537`）会返回详细错误信息，包括可能的幻觉原因、工具注册问题和修复建议。

mergeParallelFunctionResponseEvents() 函数（`internal/llminternal/base_flow.go:759`）合并多个并行工具调用响应，包括 Parts 合并和 Actions 深度合并。

---

## 第三部分：关键子系统

### 3.1 FunctionTool 工具实现

FunctionTool 是 ADK Go 中将 Go 函数包装为可调用工具的核心实现。Config 配置结构（`tool/functiontool/function.go:36`）包含 Name、Description、InputSchema、OutputSchema、IsLongRunning、RequireConfirmation 和 RequireConfirmationProvider。

泛型函数类型（`tool/functiontool/function.go:69`）使用 Go 泛型实现类型安全的函数包装。

```go
type Func[TArgs, TResults any] func(tool.Context, TArgs) (TResults, error)
```

New() 构造函数（`tool/functiontool/function.go:78`）的创建流程包括：验证输入类型（必须是 struct 或 map 或其指针）；解析输入/输出 Schema（支持自动推断）；解析确认提供者；创建工具实例。

Run() 方法（`tool/functiontool/function.go:184`）的执行流程为：Panic 恢复（捕获处理函数中的 panic 并转换为错误）；参数类型验证（args 必须是 map[string]any）；参数转换（map[string]any → TArgs）；用户确认检查（静态/动态）；执行处理函数；结果转换（TResults → map[string]any）。

用户确认机制支持两种方式：静态确认（RequireConfirmation bool）和动态确认（RequireConfirmationProvider func(TArgs) bool）。确认请求后设置 SkipSummarization = true，跳过 LLM 总结步骤。

Schema 推断（`tool/functiontool/function.go:266`）支持手动覆盖和自动推断，自动推断基于 Go 类型的反射分析。

### 3.2 模型集成

Gemini 模型实现（`model/gemini/gemini.go`）提供了对 Google Gemini API 的访问。

geminiModel 结构（`model/gemini/gemini.go:36`）包含 client、name 和 versionHeaderValue。版本头信息格式为 `google-adk/{version} gl-go/{go-version}`，用于服务端遥测和分析。

GenerateContent() 方法（`model/gemini/gemini.go:71`）的执行流程包括：maybeAppendUserContent() 确保有用户内容可处理；初始化配置（Config、HTTPOptions）；addHeaders() 设置版本头；根据 stream 参数选择调用方式（generateStream 或 generate）。

maybeAppendUserContent() 方法（`model/gemini/gemini.go:145`）的逻辑为：空内容时添加默认用户内容；最后内容不是用户时追加继续提示。

流式响应处理（generateStream）使用 StreamingResponseAggregator 聚合器，包括 ProcessResponse()（处理每个响应片段）、aggregateResponse()（聚合逻辑）和 Close()（关闭聚合器生成最终响应）。

### 3.3 会话状态管理

会话状态管理采用四层作用域设计：应用级（app:）、用户级（user:）、会话级（无前缀）和临时级（temp:）。状态优先级为 sessionState > userState > appState，后定义的值覆盖先定义的。

inMemoryService 结构（`session/inmemory.go:39`）使用三级索引存储：sessions（appName+userID+sessionID → session）、userState（appName → userID → state）、appState（appName → state）。

状态合并机制（mergeStates）按优先级合并：appState → userState → sessionState。appendEvent 时通过 ExtractStateDeltas() 提取不同级别的状态变更，updateSessionState() 更新会话状态，trimTempDeltaState() 移除临时状态（temp: 前缀）。

线程安全设计使用 RWMutex 保护并发访问，All() 方法返回克隆副本避免持锁遍历。

---

## 第四部分：扩展系统

### 4.1 插件系统

插件系统提供从用户消息到工具执行的全生命周期扩展能力。Plugin Config 配置结构（`plugin/plugin.go:26`）包含丰富回调类型：OnUserMessageCallback、OnEventCallback、BeforeRunCallback、AfterRunCallback、BeforeAgentCallback、AfterAgentCallback、BeforeModelCallback、AfterModelCallback、OnModelErrorCallback、BeforeToolCallback、AfterToolCallback 和 OnToolErrorCallback。

PluginManager 插件管理器（`internal/plugininternal/plugin_manager.go:38`）管理多个插件的注册和回调执行。回调执行采用顺序执行 + 提前退出机制：所有插件回调按注册顺序依次执行；回调返回非 nil 内容时跳过后续插件；回调返回错误时立即返回错误。

与 Runner 的集成体现在 OnUserMessageCallback（追加用户消息到会话前）、BeforeRunCallback（Agent 执行前）、AfterRunCallback（Runner 执行后延迟执行）和 OnEventCallback（每个事件产出后）。

### 4.2 工作流代理

ADK Go 提供三种工作流代理实现。

SequentialAgent（顺序执行代理，`agent/workflowagents/sequentialagent/agent.go`）按顺序依次执行子代理，执行流程为遍历 SubAgents，依次执行每个 subAgent.Run(ctx)，yield 每个事件。

ParallelAgent（并行执行代理，`agent/workflowagents/parallelagent/agent.go`）并行执行子代理，每个子代理在独立分支中运行。关键设计包括：创建 errgroup 管理并发；创建独立分支（Branch）隔离对话历史；同步确认机制（等待消费者处理完事件后再继续）。

LoopAgent（循环执行代理，`agent/workflowagents/loopagent/agent.go`）循环执行子代理直到达到最大迭代次数或子代理升级。退出条件包括：MaxIterations 达到；Escalate 标志设置；消费者终止。

三种代理对比：SequentialAgent 顺序执行无并发无分支隔离；ParallelAgent 并行执行有并发有分支隔离；LoopAgent 循环执行无并发无分支隔离。

### 4.3 REST API 服务器

REST API 服务器（`server/adkrest/`）提供 HTTP 接口与 ADK Agent 交互。核心路由包括：/sessions/*（会话管理）、/runtime/*（Agent 运行，SSE 流式返回事件）、/apps/*（应用管理）、/artifacts/*（资源管理）和 /debug/*（调试接口）。

RuntimeAPIController 的核心功能是 POST /runtime/{app_name}/users/{user_id}/sessions/{session_id}/run，运行 Agent 并 SSE 流式返回事件。

### 4.4 A2A 协议服务器

A2A 协议服务器（`server/adka2a/`）实现 Agent-to-Agent 标准，支持 Agent 之间的互操作。

eventProcessor 事件处理器（`server/adka2a/processor.go:34`）的处理流程包括：updateTerminalActions() 更新终端操作（Escalate、TransferToAgent）；处理错误响应；inputRequiredProcessor.process() 处理输入请求；convertParts() 转换 Parts；eventToArtifact.transform() 转换为 A2A 资源。

A2A 事件转换将 ADK Event 转换为 A2A TaskEvent，包括 TaskState（working、input_required、completed、failed）、Parts（TextPart、DataPart 等）和 Metadata（escalation、agent_transfer 等）。

### 4.5 Memory 服务

Memory 服务实现跨会话记忆能力，允许 Agent 访问用户历史会话中的信息。

核心接口（`memory/service.go`）包括 AddSession() 和 Search() 方法。Agent 层面的 Memory 接口（`agent/agent.go`）简化了底层接口，自动注入 AppName 和 UserID。

内存实现（`memory/inmemory.go`）的存储结构为三级索引：AppName + UserID → SessionID → []Event。关键词提取使用简单的空格分词和小写化。搜索算法提取查询关键词，遍历所有会话的所有事件，检查关键词交集，返回匹配的事件。

使用场景包括：用户偏好记忆（跨会话记住用户喜好）、项目上下文记忆（记住用户正在做的事情）和 Tool 中访问记忆（在工具执行中获取历史上下文）。

### 4.6 Artifact 服务

Artifact 服务实现文件资源管理，支持存储、加载、删除、版本控制和双重作用域。

核心接口（`artifact/service.go`）包括 Save()、Load()、Delete()、List() 和 Versions() 方法。

内存实现（`artifact/inmemory.go`）使用有序存储（omap.Map）和复合键编码。键编码机制使用 ordered.Encode 将结构体编码为字符串键，版本号使用 ordered.Rev 反转确保最新版本在前。

双重作用域包括会话作用域（文件仅在特定会话中可见）和用户作用域（文件在用户的所有会话中可见），通过文件名前缀 user: 区分。

版本控制实现包括：自动版本号递增；支持加载任意历史版本；支持列出所有版本；支持删除特定版本或全部版本。

### 4.7 Telemetry 集成

Telemetry 集成基于 OpenTelemetry 标准，提供分布式追踪和日志记录能力。

Providers 结构（`telemetry/telemetry.go`）包含 TracerProvider 和 LoggerProvider。配置选项使用 Option 模式，支持 WithOtelToCloud、WithGcpResourceProject、WithResource、WithSpanProcessors、WithTracerProvider 和 WithGenAICaptureMessageContent。

Span 类型包括：invoke_agent（Agent 调用）、generate_content（LLM 调用）和 execute_tool（工具执行）。关键属性使用 OpenTelemetry 语义约定（semconv）。

日志事件类型包括：gen_ai.system.message（系统消息）、gen_ai.user.message（用户消息）和 gen_ai.choice（响应）。

消息内容控制通过 genAICaptureMessageContent 标志实现，默认隐藏消息内容以保护隐私，可通过环境变量 OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT 启用。

---

## 第五部分：设计模式与关键技术

### 5.1 泛型迭代器模式

ADK Go 大量使用 Go 1.22+ 的泛型迭代器（iter.Seq 和 iter.Seq2）实现流式处理。典型应用包括：iter.Seq2[*session.Event, error]（事件序列）、iter.Seq2[string, any]（键值对迭代）、iter.Seq[*Event]（只读事件序列）。

迭代器模式的优势包括：原生 Go 语法支持；流式和非流式统一接口；消费者可控迭代终止；错误和响应同时处理。

### 5.2 回调链机制

回调机制贯穿整个执行流程，形成了完整的钩子链。回调类型包括 Before（执行前）、After（执行后）和 OnError（错误时）。

回调执行顺序为：插件回调（全局）→ 用户回调（Agent 级别）→ 实际执行。这种层次设计允许插件全局拦截和修改，同时用户可定制特定 Agent 行为。

提前退出机制通过返回非 nil 内容实现，跳过后续回调和实际执行。用途包括：缓存命中返回缓存响应；权限检查失败时拦截；限流时返回降级响应。

### 5.3 事件溯源模式

事件溯源是 ADK Go 的核心设计模式。每次交互生成独立事件（Event），Event 继承 LLMResponse，语义明确。Partial 事件不存储，减少存储压力。状态变更通过 StateDelta 记录，支持完整的历史追踪。

Branch 分支字段支持多代理分支隔离，子代理看不到同级代理的对话历史，适用于复杂的工作流场景。

### 5.4 状态分层管理

四层状态作用域设计支持灵活的状态共享策略。应用级状态所有用户和会话共享；用户级状态同一用户的会话共享；会话级状态当前会话专用；临时级状态当前调用专用，调用结束后丢弃。

状态前缀约定：`app:` 应用级、`user:` 用户级、`temp:` 临时级。

### 5.5 资源管理

Artifact 服务的双重作用域设计：会话作用域文件仅在当前会话可见；用户作用域文件在用户所有会话可见。

Memory 服务的预计算优化：添加会话时预计算关键词集合；搜索时无需重复解析文本；以空间换时间提高搜索效率。

---

## 第六部分：部署与集成

### 6.1 基本使用流程

基本使用流程包括：创建 LLM 模型实例；创建 Agent（LLMAgent 或自定义）；创建 Session Service；配置 Runner；调用 Runner.Run() 遍历事件流。

```go
// 创建模型
model, _ := gemini.NewModel(ctx, "gemini-2.5-flash", &genai.ClientConfig{APIKey: "..."})

// 创建 Agent
agent, _ := llmagent.New(llmagent.Config{
    Name: "my_agent",
    Model: model,
    Instruction: "You are a helpful assistant.",
    Tools: []tool.Tool{myTool},
})

// 创建 Runner
runner, _ := runner.New(runner.Config{
    AppName: "my_app",
    Agent: agent,
    SessionService: sessionService,
})

// 执行
for event, err := range runner.Run(ctx, userID, sessionID, msg, config) {
    // 处理事件
}
```

### 6.2 插件开发

插件开发示例：创建 Plugin 并配置各种回调；注册到 Runner 的 PluginConfig。

```go
myPlugin, _ := plugin.New(plugin.Config{
    Name: "my_plugin",
    BeforeModelCallback: func(ctx agent.CallbackContext, req *model.LLMRequest) (*model.LLMResponse, error) {
        // 缓存检查
        return nil, nil
    },
    AfterModelCallback: func(ctx agent.CallbackContext, resp *model.LLMResponse, err error) (*model.LLMResponse, error) {
        // Token 统计
        return nil, nil
    },
})
```

### 6.3 工具开发

工具开发示例：定义输入/输出结构体；创建处理函数；使用 functiontool.New 创建工具。

```go
type SearchArgs struct {
    Query string `json:"query"`
}
type SearchResult struct {
    Results []string `json:"results"`
}

tool, _ := functiontool.New[SearchArgs, SearchResult](
    functiontool.Config{
        Name: "search",
        Description: "Search for information",
    },
    func(ctx tool.Context, args SearchArgs) (SearchResult, error) {
        // 实现搜索逻辑
        return SearchResult{Results: []string{}}, nil
    },
)
```

### 6.4 部署选项

REST API 服务器部署示例：

```go
handler := adkrest.NewHandler(&launcher.Config{
    AppName: "my_app",
    Agent: myAgent,
    SessionService: sessionService,
    ArtifactService: artifactService,
    MemoryService: memoryService,
}, 30*time.Second)
http.ListenAndServe(":8080", handler)
```

A2A 服务器部署需要创建 AgentAdapter 并启动 A2A Server。

---

## 总结

ADK Go 是一个设计精良的 Go 语言 Agent 开发框架，具有以下核心特点：

架构设计方面，采用统一的 Agent 接口和事件溯源模式，支持灵活的扩展机制。核心执行流程清晰：Runner 协调整个执行，LLMAgent 封装 LLM 逻辑，Flow 引擎管理循环调用，工具系统支持丰富的能力扩展。

技术选型方面，大量使用 Go 1.22+ 泛型迭代器实现流式处理，基于 OpenTelemetry 的可观测性支持，基于 Google GenAI 库的模型集成。

扩展能力方面，插件系统支持全生命周期回调，工作流代理支持顺序/并行/循环执行，Memory 和 Artifact 服务支持跨会话记忆和文件管理。

生态系统方面，REST API 服务器和 A2A 协议服务器支持多种部署场景，遥测系统支持性能监控和错误诊断。

ADK Go 与 adk-python 项目保持 API 一致性，为 Go 开发者提供了构建智能 Agent 应用的强大工具集。

---

*本报告基于 ADK Go 源代码分析生成，涵盖核心架构、关键实现和设计理念。*
