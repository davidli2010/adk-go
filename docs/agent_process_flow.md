# ADK Go Agent 完整处理流程

## 流程图

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           用户请求入口 (Runner.Run)                                    │
│                           runner/runner.go:114                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 1. Session获取                                                                       │
│    runner/runner.go:119 - sessionService.Get()                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 2. Agent查找 (findAgentToRun)                                                        │
│    runner/runner.go:131                                                              │
│    - 检查是否有函数调用响应 → 返回对应的subAgent                                       │
│    - 从会话历史中查找最后一个可转移的agent                                            │
│    - 若无则返回rootAgent                                                             │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 3. 上下文构建                                                                        │
│    runner/runner.go:137-170                                                          │
│    - parentmap.ToContext() - 注入agent树关系                                          │
│    - runconfig.ToContext() - 注入运行配置                                            │
│    - plugininternal.ToContext() - 注入插件管理器                                     │
│    - 创建Artifacts/Memory服务实例                                                    │
│    - NewInvocationContext() - 创建InvocationContext                                  │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 4. 用户消息处理 (appendMessageToSession)                                            │
│    runner/runner.go:171-287                                                          │
│    - pluginManager.RunOnUserMessageCallback() - 用户消息回调                        │
│    - 保存输入Blob为Artifact (可选)                                                   │
│    - 创建Event并Append到Session                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 5. Plugin BeforeRunCallback                                                          │
│    runner/runner.go:183-196                                                          │
│    - 若返回非空结果则提前退出                                                         │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 6. Agent.Run (agent/agent.go:160)                                                   │
│    ┌─────────────────────────────────────────────────────────────────────────────┐  │
│    │ 6.1 遥测span启动 (StartInvokeAgentSpan)                                    │  │
│    │ 6.2 runBeforeAgentCallbacks (agent/agent.go:184)                          │  │
│    │     - pluginManager.RunBeforeAgentCallback()                              │  │
│    │     - agent配置的BeforeAgentCallbacks                                     │  │
│    │     - 若返回非空则创建Event并返回                                          │  │
│    │ 6.3 检查Ended标志                                                          │  │
│    │ 6.4 执行agent.run() - 即LLMAgent的run方法                                 │  │
│    │ 6.5 runAfterAgentCallbacks (agent/agent.go:208)                          │  │
│    │     - pluginManager.RunAfterAgentCallback()                               │  │
│    │     - agent配置的AfterAgentCallbacks                                      │  │
│    └─────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 7. LLMAgent.run (agent/llmagent/llmagent.go:347)                                    │
│    - icontext.NewInvocationContext() - 创建新的InvocationContext                    │
│    - 创建Flow对象 (llminternal.Flow)                                                │
│    - 返回Flow.Run()结果                                                              │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 8. Flow.Run (internal/llminternal/base_flow.go:97)                                │
│    ┌─────────────────────────────────────────────────────────────────────────────┐  │
│    │ 循环执行 runOneStep() 直到返回最终响应                                       │  │
│    │ - 若lastEvent.IsFinalResponse() = true 则退出循环                          │  │
│    └─────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 9. Flow.runOneStep (internal/llminternal/base_flow.go:125)                         │
│    ┌─────────────────────────────────────────────────────────────────────────────┐  │
│    │ 9.1 preprocess (base_flow.go:245)                                        │  │
│    │     - 按顺序执行RequestProcessors:                                        │  │
│    │       basicRequestProcessor → toolProcessor → authPreprocessor →         │  │
│    │       RequestConfirmationRequestProcessor → instructionsRequestProcessor │  │
│    │       → identityRequestProcessor → ContentsRequestProcessor →            │  │
│    │       nlPlanningRequestProcessor → codeExecutionRequestProcessor →      │  │
│    │       outputSchemaRequestProcessor → AgentTransferRequestProcessor       │  │
│    │ 9.2 检查Ended标志                                                          │  │
│    │ 9.3 callLLM (base_flow.go:290)                                          │  │
│    │     - BeforeModelCallbacks (可跳过实际LLM调用)                            │  │
│    │     - generateContent() → 调用model.GenerateContent()                    │  │
│    │     - AfterModelCallbacks                                                 │  │
│    │ 9.4 postprocess (base_flow.go:469)                                       │  │
│    │     - 执行ResponseProcessors: nlPlanningResponseProcessor,               │  │
│    │       codeExecutionResponseProcessor                                     │  │
│    │ 9.5 finalizeModelResponseEvent - 创建模型响应Event                       │  │
│    │ 9.6 handleFunctionCalls (base_flow.go:559)                                │  │
│    │     - 若有工具调用则执行工具                                               │  │
│    │     - 工具确认处理                                                         │  │
│    │ 9.7 Agent Transfer处理                                                    │  │
│    │     - 若Actions.TransferToAgent非空, 切换到目标Agent继续执行               │  │
│    └─────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 10. 工具调用处理 (handleFunctionCalls - base_flow.go:559)                           │
│     ┌─────────────────────────────────────────────────────────────────────────────┐ │
│     │ 10.1 为每个工具调用:                                                       │ │
│     │     - BeforeToolCallback (可修改参数或跳过执行)                            │ │
│     │     - tool.Run() - 实际执行工具                                            │ │
│     │     - OnToolErrorCallback (错误处理)                                      │ │
│     │     - AfterToolCallback (后处理)                                          │ │
│     │ 10.2 创建FunctionResponse Event                                          │ │
│     │ 10.3 mergeParallelFunctionResponseEvents - 合并并行工具结果              │ │
│     └─────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ 11. Event输出与存储                                                                 │
│     runner/runner.go:220-226                                                         │
│     - sessionService.AppendEvent() - 持久化Event到Session                            │
│     - yield事件给上层调用者                                                          │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

## 关键模块/函数说明

| 层级 | 模块/文件 | 关键函数 | 职责 |
|------|----------|---------|------|
| **入口** | `runner/runner.go` | `Run()` | 用户请求入口，协调整个执行流程 |
| **会话** | `session/session.go` | `Session`接口 | 管理对话状态和事件序列 |
| **Agent** | `agent/agent.go` | `Agent.Run()` | Agent基础接口，执行before/after callbacks |
| **LLM** | `agent/llmagent/llmagent.go` | `llmAgent.run()` | LLM智能代理实现 |
| **Flow** | `internal/llminternal/base_flow.go` | `Flow.Run()` | 核心执行引擎，控制LLM调用循环 |
| **LLM调用** | `internal/llminternal/base_flow.go` | `callLLM()` | 执行模型调用和处理回调 |
| **工具** | `internal/llminternal/base_flow.go` | `handleFunctionCalls()` | 执行工具调用并处理结果 |
| **插件** | `plugin/plugin.go` | 各类Callback | 扩展点：消息、事件、agent、model、tool回调 |

## 执行循环说明

Flow采用**多步循环执行**模式 (`base_flow.go:97-123`)：

```
┌─────────────────────────────────────────┐
│          Flow.Run() 循环                │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │     runOneStep() 执行一次          │  │
│  │  1. preprocess (请求预处理)        │  │
│  │  2. callLLM (调用LLM)             │  │
│  │  3. postprocess (响应后处理)       │  │
│  │  4. handleFunctionCalls (工具调用) │  │
│  │  5. Agent Transfer (代理转移)      │  │
│  └───────────────────────────────────┘  │
│                  │                       │
│                  ▼                       │
│      ┌───────────────────────┐          │
│      │ lastEvent.IsFinalRes- │          │
│      │ ponse() ?             │          │
│      └───────────┬───────────┘          │
│                  │                       │
│         YES      │      NO               │
│          ┌──────┴──────┐                │
│          │             │                │
│          ▼             ▼                │
│       退出         继续循环              │
└─────────────────────────────────────────┘
```

### 循环退出条件

只有当LLM返回**最终响应**（无FunctionCall、无FunctionResponse、非Partial）时，循环才会结束。

### 工具调用流程

若有工具调用，会先执行工具，然后将工具结果合并到下一轮LLM调用中：
1. LLM返回带FunctionCall的响应
2. handleFunctionCalls执行每个工具
3. 创建FunctionResponse Event
4. 下一轮循环将FunctionResponse发给LLM
5. LLM基于工具结果生成最终响应
