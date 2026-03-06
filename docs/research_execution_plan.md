# ADK Go 深入研究执行计划

## 概述

本计划将研究任务分解为原子粒度，按依赖关系和重要性排序。每个任务完成后标记为完成状态。

---

## 执行计划

### 阶段一：基础概念理解

#### 任务 1.1：理解项目整体结构
- [x] 1.1.1 阅读 `agent/agent.go` - Agent 接口定义 (约 50 行)
- [x] 1.1.2 阅读 `agent/context.go` - InvocationContext 接口 (约 100 行)
- [x] 1.1.3 阅读 `session/session.go` - Session 和 State 接口 (约 150 行)
- [x] 1.1.4 阅读 `model/llm.go` - LLM 接口定义 (约 50 行)
- [x] 1.1.5 阅读 `tool/tool.go` - Tool 接口定义 (约 80 行)

**产出文档**：`docs/research/01_basic_interfaces.md`

---

#### 任务 1.2：理解事件溯源模式
- [x] 1.2.1 阅读 `session/session.go` 中的 Event 结构 (Event 定义)
- [x] 1.2.2 阅读 `session/inmemory.go` - 内存会话存储实现
- [x] 1.2.3 分析 Event 类型和用途

**产出文档**：`docs/research/02_event_sourcing.md` ✓

---

### 阶段二：核心执行流程

#### 任务 2.1：Runner 入口分析
- [x] 2.1.1 阅读 `runner/runner.go:114` - Run() 函数入口
- [x] 2.1.2 分析 Session 获取逻辑 (runner/runner.go:119)
- [x] 2.1.3 分析 Agent 查找逻辑 (runner/runner.go:131)
- [x] 2.1.4 分析上下文构建过程 (runner/runner.go:137-170)
- [x] 2.1.5 分析用户消息处理 (runner/runner.go:171-287)

**产出文档**：`docs/research/03_runner_entry.md` ✓

---

#### 任务 2.2：Agent 执行入口
- [x] 2.2.1 阅读 `agent/agent.go:160` - Agent.Run() 接口实现
- [x] 2.2.2 分析 beforeAgentCallbacks 执行 (agent/agent.go:184)
- [x] 2.2.3 分析 afterAgentCallbacks 执行 (agent/agent.go:208)

**产出文档**：`docs/research/04_agent_execution.md` ✓

---

#### 任务 2.3：LLMAgent 核心实现
- [x] 2.3.1 阅读 `agent/llmagent/llmagent.go:347` - llmAgent.run() 方法
- [x] 2.3.2 分析 Flow 对象创建过程
- [x] 2.3.3 理解 LLMAgent 的配置结构

**产出文档**：`docs/research/05_llmagent_core.md` ✓

---

#### 任务 2.4：Flow 执行引擎 (核心)
- [x] 2.4.1 阅读 `internal/llminternal/base_flow.go:97` - Flow.Run() 循环
- [x] 2.4.2 阅读 `internal/llminternal/base_flow.go:125` - runOneStep() 步骤
- [x] 2.4.3 分析 preprocess 请求预处理 (base_flow.go:245)
- [x] 2.4.4 分析 callLLM 模型调用 (base_flow.go:290)
- [x] 2.4.5 分析 postprocess 响应后处理 (base_flow.go:469)
- [x] 2.4.6 分析 handleFunctionCalls 工具调用 (base_flow.go:559)

**产出文档**：`docs/research/06_flow_engine.md` ✓

---

#### 任务 2.5：LLM 调用详解
- [x] 2.5.1 分析 BeforeModelCallbacks 执行时机
- [x] 2.5.2 分析 generateContent 调用流程
- [x] 2.5.3 分析 AfterModelCallbacks 执行
- [x] 2.5.4 理解流式响应处理 (iter.Seq2)

**产出文档**：`docs/research/07_llm_invocation.md` ✓

---

#### 任务 2.6：工具调用流程
- [x] 2.6.1 分析 BeforeToolCallback 执行
- [x] 2.6.2 分析 tool.Run() 实际执行
- [x] 2.6.3 分析 AfterToolCallback 执行
- [x] 2.6.4 分析 FunctionResponse Event 创建

**产出文档**：`docs/research/08_tool_invocation.md` ✓

---

### 阶段三：关键子系统

#### 任务 3.1：FunctionTool 实现
- [x] 3.1.1 阅读 `tool/functiontool/functiontool.go` - FunctionTool 结构
- [x] 3.1.2 分析工具处理参数和结果返回

**产出文档**：`docs/research/09_function_tool.md` ✓

---

#### 任务 3.2：模型集成
- [x] 3.2.1 阅读 `model/gemini/gemini.go` - Gemini 模型实现
- [x] 3.2.2 分析流式响应处理

**产出文档**：`docs/research/10_model_integration.md` ✓

---

#### 任务 3.3：会话状态管理
- [x] 3.3.1 分析 Session 状态存储机制
- [x] 3.3.2 分析 State 接口的 Get/Set/All 方法
- [x] 3.3.3 分析会话状态层级结构

**产出文档**：`docs/research/11_session_state.md` ✓

---

### 阶段四：扩展系统

#### 任务 4.1：插件系统
- [ ] 4.1.1 阅读 `plugin/plugin.go` - 插件配置和回调类型
- [ ] 4.1.2 分析插件管理器实现

**产出文档**：`docs/research/12_plugin_system.md`

---

#### 任务 4.2：工作流代理
- [ ] 4.2.1 分析 SequentialAgent 实现
- [ ] 4.2.2 分析 ParallelAgent 实现
- [ ] 4.2.3 分析 LoopAgent 实现

**产出文档**：`docs/research/13_workflow_agents.md`

---

#### 任务 4.3：部署服务
- [ ] 4.3.1 分析 REST API 服务器实现
- [ ] 4.3.2 分析 A2A 协议服务器实现

**产出文档**：`docs/research/14_deployment_services.md`

---

### 阶段五：支撑模块

#### 任务 5.1：Memory 服务
- [ ] 5.1.1 阅读 `memory/service.go` - Memory 服务接口
- [ ] 5.1.2 分析跨会话记忆机制

**产出文档**：`docs/research/15_memory_service.md`

---

#### 任务 5.2：Artifact 服务
- [ ] 5.2.1 阅读 `artifact/service.go` - Artifact 服务接口
- [ ] 5.2.2 分析版本控制机制

**产出文档**：`docs/research/16_artifact_service.md`

---

#### 任务 5.3：Telemetry 集成
- [ ] 5.3.1 阅读 `telemetry/telemetry.go` - 遥测接口
- [ ] 5.3.2 分析 OpenTelemetry 集成

**产出文档**：`docs/research/17_telemetry.md`

---

## 执行顺序说明

### 第一阶段：基础概念 (任务 1.1 - 1.2)
- 理解核心接口定义
- 理解事件溯源模式
- **前置要求**：无

### 第二阶段：核心执行流程 (任务 2.1 - 2.6)
- 从入口到执行引擎
- 理解完整的请求处理流程
- **前置要求**：第一阶段完成

### 第三阶段：关键子系统 (任务 3.1 - 3.3)
- 工具系统、模型集成、会话状态
- **前置要求**：第二阶段完成

### 第四阶段：扩展系统 (任务 4.1 - 4.3)
- 插件系统、工作流代理、部署服务
- **前置要求**：第三阶段完成

### 第五阶段：支撑模块 (任务 5.1 - 5.3)
- Memory、Artifact、Telemetry
- **前置要求**：第三阶段完成

---

## 文档命名规范

所有产出文档保存到 `docs/research/` 目录下，命名格式为：
- `XX_<topic>.md` - XX 为任务编号

---

## 进度追踪

每次完成一个任务后，在对应任务前标记 `[x]`。重启后可继续执行。

## 最新进度

- 阶段一：基础概念理解 - 2/5 (任务 1.1-1.2 ✓)
- 阶段二：核心执行流程 - 6/6 (任务 2.1-2.6 ✓)
- 阶段三：关键子系统 - 3/3 (任务 3.1-3.3 ✓)
- 阶段四：扩展系统 - 0/3
- 阶段五：支撑模块 - 0/3

---

## 执行命令参考

```bash
# 阅读指定文件
cat agent/agent.go

# 阅读指定行号范围
sed -n '160,200p' runner/runner.go

# 运行相关测试
go test -mod=readonly -v -count=1 ./agent -run TestAgentCallbacks
go test -mod=readonly -v -count=1 ./runner -run TestRunner
```
