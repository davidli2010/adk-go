# ADK Go 工作流代理详解

## 概述

本文档深入分析 ADK Go 中的三种工作流代理：SequentialAgent、ParallelAgent 和 LoopAgent。

---

## 1. SequentialAgent 顺序执行代理

### 概述

SequentialAgent 按顺序依次执行其子代理。

### 结构定义

**位置**: `agent/workflowagents/sequentialagent/agent.go`

```go
type sequentialAgent struct{}

func (a *sequentialAgent) Run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error]
```

### Config 配置

```go
type Config struct {
    AgentConfig agent.Config
}
```

### 创建流程

```go
func New(cfg Config) (agent.Agent, error) {
    // 禁止自定义 Run
    if cfg.AgentConfig.Run != nil {
        return nil, fmt.Errorf("LoopAgent doesn't allow custom Run implementations")
    }
    
    sequentialAgentImpl := &sequentialAgent{}
    cfg.AgentConfig.Run = sequentialAgentImpl.Run
    
    // 创建基础 Agent
    sequentialAgent, err := agent.New(cfg.AgentConfig)
    
    // 设置内部状态
    state.AgentType = agentinternal.TypeSequentialAgent
    
    return sequentialAgent, nil
}
```

### Run() 实现

```go
func (a *sequentialAgent) Run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error] {
    return func(yield func(*session.Event, error) bool) {
        // 按顺序遍历子代理
        for _, subAgent := range ctx.Agent().SubAgents() {
            // 执行子代理
            for event, err := range subAgent.Run(ctx) {
                if !yield(event, err) {
                    return
                }
            }
        }
    }
}
```

### 执行流程图

```
SequentialAgent.Run()
    │
    ├─ for subAgent in SubAgents (顺序)
    │   │
    │   └─ for event in subAgent.Run(ctx)
    │       │
    │       ├─ yield event
    │       └─ 检查错误
    │
    └─ 全部完成 → 返回
```

### 使用示例

```go
rootAgent, err := sequentialagent.New(sequentialagent.Config{
    AgentConfig: agent.Config{
        Name: "data_pipeline",
        SubAgents: []agent.Agent{
            validatorAgent,
            transformerAgent,
            exporterAgent,
        },
    },
})
```

---

## 2. ParallelAgent 并行执行代理

### 概述

ParallelAgent 并行执行其子代理，每个子代理在独立的分支中运行。

### 结构定义

**位置**: `agent/workflowagents/parallelagent/agent.go`

```go
type result struct {
    event   *session.Event
    err     error
    ackChan chan struct{}  // 确认通道
}
```

### Config 配置

```go
type Config struct {
    AgentConfig agent.Config
}
```

### 创建流程

```go
func New(cfg Config) (agent.Agent, error) {
    // 禁止自定义 Run
    if cfg.AgentConfig.Run != nil {
        return nil, fmt.Errorf("ParallelAgent doesn't allow custom Run implementations")
    }
    
    cfg.AgentConfig.Run = run
    
    // 创建基础 Agent
    parallelAgent, err := agent.New(cfg.AgentConfig)
    
    // 设置内部状态
    state.AgentType = agentinternal.TypeParallelAgent
    
    return parallelAgent, nil
}
```

### Run() 实现

```go
func run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error] {
    curAgent := ctx.Agent()
    
    // 创建错误组和通道
    var (
        errGroup, errGroupCtx = errgroup.WithContext(ctx)
        doneChan              = make(chan bool)
        resultsChan           = make(chan result)
    )
    
    // 为每个子代理启动 goroutine
    for _, sa := range ctx.Agent().SubAgents() {
        // 创建独立分支
        branch := fmt.Sprintf("%s.%s", curAgent.Name(), sa.Name())
        
        subAgent := sa
        errGroup.Go(func() error {
            subCtx := icontext.NewInvocationContext(errGroupCtx, icontext.InvocationContextParams{
                // 独立分支
                Branch: branch,
                // 共享会话和资源
                Session:      ctx.Session(),
                Artifacts:    ctx.Artifacts(),
                Memory:       ctx.Memory(),
                UserContent:  ctx.UserContent(),
                RunConfig:    ctx.RunConfig(),
                InvocationID: ctx.InvocationID(),
            })
            
            return runSubAgent(subCtx, subAgent, resultsChan, doneChan)
        })
    }
    
    // 等待所有子代理完成
    go func() {
        if err := errGroup.Wait(); err != nil {
            resultsChan <- result{err: err}
        }
        close(resultsChan)
    }()
    
    // 产出结果
    return func(yield func(*session.Event, error) bool) {
        defer close(doneChan)
        
        for res := range resultsChan {
            shouldContinue := yield(res.event, res.err)
            
            // 等待确认
            if res.ackChan != nil {
                close(res.ackChan)
            }
            
            if !shouldContinue {
                break
            }
        }
    }
}
```

### 执行流程图

```
ParallelAgent.Run()
    │
    ├─ 创建 errgroup
    ├─ 创建通道
    │   ├─ doneChan: 终止信号
    │   └─ resultsChan: 结果传递
    │
    ├─ 为每个子代理启动 goroutine
    │   └─ runSubAgent()
    │       ├─ 创建独立分支
    │       ├─ 执行 subAgent.Run()
    │       └─ 发送到 resultsChan
    │
    ├─ 等待 errgroup 完成
    │
    └─ yield 每个结果
        └─ 等待确认
```

### 分支隔离

每个子代理运行在独立的分支中：

```go
branch := fmt.Sprintf("%s.%s", curAgent.Name(), sa.Name())
// 例如: "data_pipeline.validator"
```

**用途**: 子代理只能看到自己的对话历史，避免相互干扰。

### 同步机制

```go
// 子代理产出一个事件后等待确认
select {
case <-ackChan:        // 确认后继续
case <-doneChan:       // 终止信号
case <-ctx.Done():     // 上下文取消
}
```

---

## 3. LoopAgent 循环执行代理

### 概述

LoopAgent 循环执行其子代理，直到达到最大迭代次数或子代理升级。

### 结构定义

**位置**: `agent/workflowagents/loopagent/agent.go`

```go
type loopAgent struct {
    maxIterations uint
}
```

### Config 配置

```go
type Config struct {
    AgentConfig agent.Config
    
    // 最大迭代次数
    // 0 表示无限循环直到升级
    MaxIterations uint
}
```

### 创建流程

```go
func New(cfg Config) (agent.Agent, error) {
    // 禁止自定义 Run
    if cfg.AgentConfig.Run != nil {
        return nil, fmt.Errorf("LoopAgent doesn't allow custom Run implementations")
    }
    
    loopAgentImpl := &loopAgent{
        maxIterations: cfg.MaxIterations,
    }
    cfg.AgentConfig.Run = loopAgentImpl.Run
    
    // 创建基础 Agent
    loopAgent, err := agent.New(cfg.AgentConfig)
    
    // 设置内部状态
    state.AgentType = agentinternal.TypeLoopAgent
    
    return loopAgent, nil
}
```

### Run() 实现

```go
func (a *loopAgent) Run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error] {
    count := a.maxIterations
    
    return func(yield func(*session.Event, error) bool) {
        for {
            shouldExit := false
            
            // 顺序执行每个子代理
            for _, subAgent := range ctx.Agent().SubAgents() {
                for event, err := range subAgent.Run(ctx) {
                    if !yield(event, err) {
                        return
                    }
                    
                    // 检查升级标志
                    if event != nil && event.Actions.Escalate {
                        shouldExit = true
                    }
                }
                if shouldExit {
                    return
                }
            }
            
            // 迭代计数
            if count > 0 {
                count--
                if count == 0 {
                    return
                }
            }
        }
    }
}
```

### 执行流程图

```
LoopAgent.Run()
    │
    ├─ count = MaxIterations
    │
    ├─ for { 无限循环 }
    │   │
    │   ├─ for subAgent in SubAgents
    │   │   └─ subAgent.Run(ctx)
    │   │       ├─ yield event
    │   │       └─ 检查 Escalate
    │   │
    │   ├─ 检查 shouldExit
    │   │   └─ true → return
    │   │
    │   └─ 迭代计数
    │       ├─ count > 0 → count--
    │       └─ count == 0 → return
    │
    └─ 退出条件
        ├─ 达到最大迭代次数
        ├─ 子代理升级 (Escalate)
        └─ 消费者终止
```

### 退出条件

| 条件 | 说明 |
|------|------|
| `MaxIterations` 达到 | 循环指定次数后退出 |
| `Escalate` 标志 | 子代理设置升级标志时退出 |
| 消费者终止 | yield 返回 false 时退出 |

### 使用示例

```go
codeReviewAgent, err := loopagent.New(loopagent.Config{
    AgentConfig: agent.Config{
        Name: "code_reviewer",
        SubAgents: []agent.Agent{
            linterAgent,
            formatterAgent,
            testerAgent,
        },
    },
    MaxIterations: 3,  // 最多迭代 3 次
})
```

---

## 三种代理对比

### 执行模式

| 代理 | 执行模式 | 并发 | 分支隔离 |
|------|----------|------|----------|
| SequentialAgent | 顺序 | 否 | 否 |
| ParallelAgent | 并行 | 是 | 是 |
| LoopAgent | 循环 | 否 | 否 |

### 使用场景

| 代理 | 适用场景 |
|------|----------|
| SequentialAgent | 数据处理管道、验证链 |
| ParallelAgent | 多视角分析、多算法尝试 |
| LoopAgent | 代码修订、迭代优化 |

### 示例对比

```go
// 1. 顺序执行：数据处理管道
dataPipeline, _ := sequentialagent.New(sequentialagent.Config{
    AgentConfig: agent.Config{
        Name: "data_pipeline",
        SubAgents: []agent.Agent{extractAgent, transformAgent, loadAgent},
    },
})

// 2. 并行执行：多视角分析
multiViewAnalysis, _ := parallelagent.New(parallelagent.Config{
    AgentConfig: agent.Config{
        Name: "multi_view",
        SubAgents: []agent.Agent{analysisAgent1, analysisAgent2, analysisAgent3},
    },
})

// 3. 循环执行：代码修订
codeRevision, _ := loopagent.New(loopagent.Config{
    AgentConfig: agent.Config{
        Name: "code_revisor",
        SubAgents: []agent.Agent{reviewAgent, fixAgent},
    },
    MaxIterations: 5,
})
```

---

## 关键设计决策

### 1. 禁止自定义 Run

```go
if cfg.AgentConfig.Run != nil {
    return nil, fmt.Errorf("doesn't allow custom Run implementations")
}
```

**原因**: 工作流代理内部管理执行逻辑，自定义 Run 会破坏设计。

### 2. 分支隔离（ParallelAgent）

```go
Branch: branch  // "parent.child"
```

**原因**: 并行执行时需要隔离对话历史，避免相互干扰。

### 3. 同步确认机制

```go
// 等待确认后继续
select {
case <-ackChan:
case <-doneChan:
}
```

**原因**: 确保消费者处理完事件后再继续，防止过快产出。

### 4. 升级标志

```go
if event != nil && event.Actions.Escalate {
    shouldExit = true
}
```

**原因**: 子代理可主动请求退出循环。

### 5. errgroup 并发管理

```go
errGroup, errGroupCtx := errgroup.WithContext(ctx)
```

**原因**:
- 管理多个 goroutine 生命周期
- 共享上下文，取消时全部取消
- 收集错误

---

## 与 Agent 接口的关系

```go
type Agent interface {
    Name() string
    Description() string
    Run(InvocationContext) iter.Seq2[*session.Event, error]
    SubAgents() []Agent
    internal() *agent
}
```

**实现对照**:

| 方法 | 实现方式 |
|------|----------|
| Name() | 来自 AgentConfig |
| Description() | 来自 AgentConfig |
| Run() | 自定义实现 |
| SubAgents() | 来自 AgentConfig |
| internal() | agent.New() 返回 |

---

## 总结

ADK Go 的工作流代理具有以下特点：

1. **SequentialAgent**: 简单顺序执行，适用于管道式处理
2. **ParallelAgent**: 并行执行 + 分支隔离 + 同步确认
3. **LoopAgent**: 循环执行 + 迭代计数 + 升级退出

共同设计：
- 禁止自定义 Run 方法
- 复用 Agent.Config 结构
- 设置内部 AgentType
- 使用 iter.Seq2 返回事件流
