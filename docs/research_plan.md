# ADK Go 深入研究计划

根据项目文档和代码结构，制定以下深入研究计划，聚焦关键技术，忽略无关紧要的问题和模块。

## 研究目标

深入理解 ADK Go 项目的核心架构和关键技术实现，掌握从请求入口到事件输出的完整流程。

---

## 第一阶段：核心架构理解

**优先级**：高

### 研究重点

| 模块 | 关键文件 | 行号 | 研究重点 |
|------|---------|------|---------|
| **Agent核心** | `agent/agent.go` | :41 | Agent接口定义 |
| **Agent核心** | `agent/llmagent/llmagent.go` | :347 | LLMAgent.run() 实现 |
| **执行引擎** | `internal/llminternal/base_flow.go` | :97 | Flow.Run() 循环执行 |
| **执行引擎** | `internal/llminternal/base_flow.go` | :125 | runOneStep() 步骤执行 |
| **执行引擎** | `internal/llminternal/base_flow.go` | :290 | callLLM() 模型调用 |
| **执行引擎** | `internal/llminternal/base_flow.go` | :559 | handleFunctionCalls() 工具调用 |
| **运行时** | `runner/runner.go` | :114 | Runner.Run() 入口 |

### 核心理解

- Runner 作为用户请求入口，协调整个执行流程
- Flow 采用多步循环执行模式：preprocess → callLLM → postprocess → handleFunctionCalls
- 循环退出条件：LLM 返回最终响应（无FunctionCall、无FunctionResponse、非Partial）

---

## 第二阶段：关键子系统

**优先级**：高

### 研究重点

| 模块 | 关键文件 | 行号 | 研究重点 |
|------|---------|------|---------|
| **工具系统** | `tool/tool.go` | :30 | Tool 接口定义 |
| **工具系统** | `tool/tool.go` | :43 | Context 接口 |
| **工具系统** | `tool/functiontool/functiontool.go` | - | FunctionTool 实现 |
| **会话管理** | `session/session.go` | :32 | Session 接口 |
| **会话管理** | `session/session.go` | :51 | State 接口 |
| **会话管理** | `session/session.go` | :92 | Event 结构 |
| **会话管理** | `session/inmemory.go` | - | 内存会话存储实现 |
| **模型集成** | `model/llm.go` | :26 | LLM 接口定义 |
| **模型集成** | `model/gemini/gemini.go` | - | Gemini 模型实现 |

### 核心技术点

1. **工具系统**：统一工具调用接口，支持同步和异步执行
2. **会话管理**：事件溯源模式，Event 序列记录所有交互
3. **模型集成**：流式响应处理，iter.Seq2 类型

---

## 第三阶段：扩展系统

**优先级**：中

### 研究重点

| 模块 | 关键文件 | 研究重点 |
|------|---------|---------|
| `plugin/plugin.go` | 插件配置和回调类型 |
| `internal/plugininternal/` | 插件管理器实现 |
| `agent/workflowagents/sequentialagent/` | 顺序执行代理 |
| `agent/workflowagents/parallelagent/` | 并行执行代理 |
| `agent/workflowagents/loopagent/` | 循环执行代理 |
| `server/adkrest/` | REST API 服务器 |
| `server/adka2a/` | A2A 协议服务器 |

### 插件回调类型

- 用户消息回调：OnUserMessageCallback
- 事件回调：OnEventCallback
- 运行前后回调：BeforeRunCallback / AfterRunCallback
- Agent 回调：BeforeAgentCallback / AfterAgentCallback
- Model 回调：BeforeModelCallback / AfterModelCallback / OnModelErrorCallback
- Tool 回调：BeforeToolCallback / AfterToolCallback / OnToolErrorCallback

---

## 第四阶段：支撑模块

**优先级**：低

| 模块 | 研究重点 |
|------|---------|
| `memory/` | 长期记忆服务，跨会话记忆能力 |
| `artifact/` | 文件资源管理，版本控制 |
| `telemetry/` | OpenTelemetry 集成，可观测性 |
| `cmd/launcher/` | 启动器配置，多协议端口管理 |

---

## 核心技术点聚焦

### 1. 事件溯源模式

- 使用 `Event` 序列记录所有交互
- 支持完整的对话历史追踪和回放
- 每次交互都生成独立的事件，支持审计和调试

### 2. Flow 循环执行

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

### 3. 回调钩子链

执行顺序：
1. BeforeAgentCallback
2. BeforeModelCallback
3. LLM 调用
4. AfterModelCallback
5. BeforeToolCallback
6. 工具执行
7. AfterToolCallback
8. Agent Transfer
9. AfterAgentCallback

### 4. InvocationContext

贯穿整个请求的生命周期上下文，包含：
- Agent() - 当前执行的代理
- Session() - 会话状态
- Artifacts() - 资源文件
- Memory() - 长期记忆
- InvocationID() - 调用唯一标识
- Branch() - 调用分支路径

### 5. 泛型迭代器

大量使用 Go 1.22+ 的泛型迭代器：
- `iter.Seq2[*session.Event, error]` - 事件序列和 `iter.Seq错误处理
-2[string, any]` - 键值对迭代
- `iter.Seq[*Event]` - 只读事件序列

---

## 建议研究顺序

1. **入口理解**：先看 `runner/runner.go:114` 入口，理解整体流程
2. **LLM 实现**：再看 `agent/llmagent/llmagent.go` 理解 LLMAgent
3. **核心执行**：重点研究 `internal/llminternal/base_flow.go` 核心执行逻辑
4. **支撑系统**：最后看 `session/` 和 `tool/` 理解支撑系统

---

## 忽略的内容

以下模块和内容在初步研究阶段可以忽略：

- 特定云平台集成细节（如 GCS 存储细节）
- 测试代码实现（`*_test.go`）
- HTTP 录制测试工具（`internal/httprr/`）
- CLI 部署工具（`cmd/adkgo/`）
- 云平台部署示例（`cloudrun/`）
- 文档和配置示例
