# ADK Go 事件溯源模式研究

## 概述

ADK Go 采用事件溯源（Event Sourcing）模式来记录所有用户与代理之间的交互。每次交互都生成独立的事件，支持完整的对话历史追踪、回放和审计。

---

## 核心概念

### 1. Event 结构

**位置**: `session/session.go:89-118`

```go
type Event struct {
    model.LLMResponse
    
    // 存储层设置
    ID        string
    Timestamp time.Time
    
    // 上下文设置
    InvocationID string
    Branch       string  // 分支路径，格式：agent_1.agent_2.agent_3
    Author       string
    
    // 行为记录
    Actions          EventActions
    LongRunningToolIDs []string
}
```

**关键字段说明**:

| 字段 | 用途 |
|------|------|
| `ID` | 事件唯一标识符（UUID） |
| `Timestamp` | 事件创建时间 |
| `InvocationID` | 调用唯一标识，关联整个请求生命周期 |
| `Branch` | 代理分支路径，用于隔离子代理的对话历史 |
| `Author` | 事件作者名称 |
| `Actions` | 事件行为集合（状态变更、工具调用等） |
| `LongRunningToolIDs` | 长运行工具调用 ID 列表 |

### 2. Event 继承关系

`Event` 嵌入 `model.LLMResponse`，包含：
- `Content` - LLM 响应内容
- `Partial` - 是否为部分响应
- 其他 LLM 相关字段

这种设计使得 Event 既是 LLM 响应，又是持久化的交互记录。

---

## 事件类型判断

### IsFinalResponse() 方法

**位置**: `session/session.go:120-129`

```go
func (e *Event) IsFinalResponse() bool {
    if (e.Actions.SkipSummarization) || len(e.LongRunningToolIDs) > 0 {
        return true
    }
    return !hasFunctionCalls(&e.LLMResponse) && 
           !hasFunctionResponses(&e.LLMResponse) && 
           !e.LLMResponse.Partial && 
           !hasTrailingCodeExecutionResult(&e.LLMResponse)
}
```

**判断逻辑**:
1. **立即返回 true 的条件**:
   - `SkipSummarization` 为 true（跳过总结）
   - 存在长运行工具调用

2. **返回 false 的条件**（需要继续循环）:
   - 存在函数调用（FunctionCall）
   - 存在函数响应（FunctionResponse）
   - 是部分响应（Partial）
   - 存在尾部代码执行结果

**用途**: Flow 循环执行的退出条件判断

---

## EventActions 结构

**位置**: `session/session.go:142-160`

```go
type EventActions struct {
    StateDelta                 map[string]any
    ArtifactDelta              map[string]int64
    RequestedToolConfirmations map[string]toolconfirmation.ToolConfirmation
    SkipSummarization          bool
    TransferToAgent            string
    Escalate                   bool
}
```

**字段说明**:

| 字段 | 用途 |
|------|------|
| `StateDelta` | 会话状态变更（键值对） |
| `ArtifactDelta` | 文件资源版本变更（文件名 -> 版本号） |
| `RequestedToolConfirmations` | 需要用户确认的工具 |
| `SkipSummarization` | 跳过工具响应总结标志 |
| `TransferToAgent` | 代理转移目标名称 |
| `Escalate` | 升级到上级代理标志 |

---

## 状态作用域前缀

**位置**: `session/session.go:162-177`

```go
const (
    KeyPrefixApp  string = "app:"   // 应用级状态（跨用户、跨会话共享）
    KeyPrefixTemp string = "temp:"  // 临时状态（当前调用专用）
    KeyPrefixUser string = "user:"  // 用户级状态（跨会话共享）
)
```

**状态层级**:
1. **App 级**: 所有用户和会话共享
2. **User 级**: 同一用户的所有会话共享
3. **Session 级**: 当前会话专用
4. **Temp 级**: 当前调用专用，调用结束后丢弃

---

## 内存会话存储实现

### inMemoryService 结构

**位置**: `session/inmemory.go:39-44`

```go
type inMemoryService struct {
    mu        sync.RWMutex
    sessions  omap.Map[string, *session]  // session.ID -> storedSession
    userState map[string]map[string]stateMap
    appState  map[string]stateMap
}
```

**特点**:
- 线程安全（使用 `sync.RWMutex`）
- 使用有序映射（`omap.Map`）存储会话
- 分离存储用户状态和应用状态

### session 内部结构

**位置**: `session/inmemory.go:299-313`

```go
type session struct {
    id        id
    mu        sync.RWMutex
    events    []*Event
    state     map[string]any
    updatedAt time.Time
}
```

---

## 核心操作流程

### 1. 创建会话

**位置**: `session/inmemory.go:46-93`

```go
func (s *inMemoryService) Create(ctx context.Context, req *CreateRequest) (*CreateResponse, error) {
    // 1. 验证参数
    // 2. 生成 SessionID（如未提供）
    // 3. 创建 session 对象
    // 4. 合并状态（appState + userState + sessionState）
    // 5. 存储并返回副本
}
```

### 2. 获取会话

**位置**: `session/inmemory.go:95-140`

```go
func (s *inMemoryService) Get(ctx context.Context, req *GetRequest) (*GetResponse, error) {
    // 1. 验证参数
    // 2. 查找会话
    // 3. 合并状态
    // 4. 过滤事件（支持 NumRecentEvents 和 After 时间过滤）
    // 5. 返回副本
}
```

**事件过滤逻辑**:
- `NumRecentEvents`: 只返回最近 N 条事件
- `After`: 只返回指定时间之后的事件

### 3. 追加事件（核心）

**位置**: `session/inmemory.go:197-254`

```go
func (s *inMemoryService) AppendEvent(ctx context.Context, curSession Session, event *Event) error {
    // 1. 验证参数
    // 2. 忽略 Partial 事件
    // 3. 更新会话状态
    // 4. 深拷贝事件并追加
    // 5. 更新 appState 和 userState（如有状态变更）
}
```

**关键处理**:
- **Partial 事件处理**: 直接返回，不存储（`event.Partial` 为 true 时）
- **状态提取**: `sessionutils.ExtractStateDeltas` 分离不同作用域的状态
- **深拷贝**: 确保事件数据不可变

### 4. 会话内部追加事件

**位置**: `session/inmemory.go:347-363`

```go
func (s *session) appendEvent(event *Event) error {
    // 1. 忽略 Partial 事件
    // 2. 更新会话状态（updateSessionState）
    // 3. 移除临时状态（trimTempDeltaState）
    // 4. 追加事件到列表
    // 5. 更新时间戳
}
```

---

## 状态管理

### state 实现

**位置**: `session/inmemory.go:388-419`

```go
type state struct {
    mu    *sync.RWMutex
    state map[string]any
}

func (s *state) Get(key string) (any, error)
func (s *state) Set(key string, value any) error
func (s *state) All() iter.Seq2[string, any]
```

**特点**:
- 线程安全（读写锁）
- `All()` 方法返回克隆的副本，避免并发问题
- 使用 Go 1.22+ 泛型迭代器

### 状态合并

**位置**: `session/inmemory.go:281-289`

```go
func (s *inMemoryService) mergeStates(state stateMap, appName, userID string) stateMap {
    appState := s.appState[appName]
    userState := s.userState[appName][userID]
    return sessionutils.MergeStates(appState, userState, state)
}
```

**合并顺序**: `appState` → `userState` → `sessionState`（后者覆盖前者）

---

## 临时状态处理

### trimTempDeltaState 函数

**位置**: `session/inmemory.go:428-447`

```go
func trimTempDeltaState(event *Event) *Event {
    if len(event.Actions.StateDelta) == 0 {
        return event
    }
    
    filteredStateDelta := make(map[string]any)
    for key, value := range event.Actions.StateDelta {
        if !strings.HasPrefix(key, KeyPrefixTemp) {
            filteredStateDelta[key] = value
        }
    }
    
    event.Actions.StateDelta = filteredStateDelta
    return event
}
```

**用途**: 在持久化事件前，移除临时状态（`temp:` 前缀），避免存储不必要的临时数据。

---

## 事件溯源优势

### 1. 完整的历史追踪
- 每次交互都记录为独立事件
- 支持完整的对话回放
- 便于调试和审计

### 2. 状态可重现
- 通过重放事件序列可重建任意时间点的状态
- 支持断点续传和错误恢复

### 3. 分支隔离
- `Branch` 字段支持多代理分支
- 子代理看不到同级代理的对话历史
- 适用于复杂的工作流场景

### 4. 灵活的事件过滤
- 支持按时间范围过滤
- 支持按数量过滤（最近 N 条）
- 支持按分支过滤（未来扩展）

---

## 使用示例

### 创建事件

```go
event := session.NewEvent(invocationID)
event.Author = "assistant"
event.Branch = "root.subagent1"
event.Content = &model.Content{
    Parts: []*model.Part{
        {Text: "Hello, how can I help you?"},
    },
}
```

### 追加事件到会话

```go
err := sessionService.AppendEvent(ctx, curSession, event)
if err != nil {
    return fmt.Errorf("failed to append event: %w", err)
}
```

### 遍历事件

```go
events := session.Events()
for event := range events.All() {
    fmt.Printf("[%s] %s: %v\n", 
        event.Timestamp, 
        event.Author, 
        event.Content)
}
```

---

## 关键设计决策

### 1. 事件不可变性
- 事件一旦创建并追加，不再修改
- 所有更新操作都生成新事件
- 保证历史记录的真实性

### 2. Partial 事件处理
- Partial 事件（流式响应的中间片段）不存储
- 只存储最终完整的响应事件
- 减少存储压力，保持事件语义清晰

### 3. 状态分层管理
- 应用级、用户级、会话级、临时级四层状态
- 不同作用域的状态独立管理
- 支持灵活的状态共享策略

### 4. 线程安全设计
- 所有公开方法都使用锁保护
- 返回数据都是深拷贝
- 避免并发读写问题

---

## 与 Flow 循环的关系

事件溯源模式是 Flow 循环执行的基础：

```
Flow.Run() 循环
    ↓
runOneStep()
    ↓
preprocess → callLLM → postprocess → handleFunctionCalls
    ↓
生成 Event → AppendEvent → Session
    ↓
检查 IsFinalResponse()
    ├─ YES → 退出循环
    └─ NO  → 继续循环
```

每个步骤都通过事件记录，形成完整的执行轨迹。

---

## 总结

ADK Go 的事件溯源模式具有以下特点：

1. **结构清晰**: Event 结构继承 LLMResponse，语义明确
2. **功能完整**: 支持状态管理、分支隔离、事件过滤
3. **线程安全**: 全面使用锁保护，返回数据深拷贝
4. **性能优化**: Partial 事件不存储，临时状态自动清理
5. **扩展性强**: 支持多级状态作用域，适应复杂场景

这种设计为 ADK Go 提供了可靠的对话历史管理能力，是实现多轮对话、工具调用和代理协作的基础。
