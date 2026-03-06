# ADK Go 会话状态管理详解

## 概述

本文档深入分析 ADK Go 中会话状态管理的实现，包括 Session 接口、State 接口、会话状态存储机制和层级结构。

---

## 核心接口定义

### Session 接口

**位置**: `session/session.go:28-46`

```go
type Session interface {
    ID() string
    AppName() string
    UserID() string
    
    State() State
    Events() Events
    LastUpdateTime() time.Time
}
```

**接口方法说明**:

| 方法 | 返回值 | 用途 |
|------|--------|------|
| `ID()` | string | 会话唯一标识 |
| `AppName()` | string | 应用名称 |
| `UserID()` | string | 用户标识 |
| `State()` | State | 会话状态 |
| `Events()` | Events | 事件历史 |
| `LastUpdateTime()` | time.Time | 最后更新时间 |

---

## State 接口

**位置**: `session/session.go:48-62`

```go
type State interface {
    Get(string) (any, error)
    Set(string, any) error
    All() iter.Seq2[string, any]
}
```

**接口方法说明**:

| 方法 | 参数 | 返回值 | 用途 |
|------|------|--------|------|
| `Get(key)` | string | (any, error) | 获取值，键不存在返回错误 |
| `Set(key, value)` | string, any | error | 设置值 |
| `All()` | - | iter.Seq2[string, any] | 遍历所有键值对 |

### ReadonlyState 接口

**位置**: `session/session.go:64-74`

```go
type ReadonlyState interface {
    Get(string) (any, error)
    All() iter.Seq2[string, any]
}
```

**用途**: 只读访问状态，用于回调等场景

---

## 状态层级结构

### 状态前缀常量

**位置**: `session/session.go:162-176`

```go
const (
    KeyPrefixApp  string = "app:"   // 应用级
    KeyPrefixTemp string = "temp:"  // 临时级
    KeyPrefixUser string = "user:"  // 用户级
)
```

### 层级说明

```
┌─────────────────────────────────────────┐
│            状态层级                       │
├─────────────────────────────────────────┤
│  Level 1: App State (app:)              │
│    - 所有用户和会话共享                    │
│    - 应用级配置和数据                      │
├─────────────────────────────────────────┤
│  Level 2: User State (user:)            │
│    - 同一用户的所有会话共享                │
│    - 用户级偏好和历史                      │
├─────────────────────────────────────────┤
│  Level 3: Session State (无前缀)          │
│    - 当前会话专用                         │
│    - 会话级数据                          │
├─────────────────────────────────────────┤
│  Level 4: Temp State (temp:)            │
│    - 当前调用专用                         │
│    - 调用结束后丢弃                        │
└─────────────────────────────────────────┘
```

### 状态优先级

**合并顺序**: `appState` → `userState` → `sessionState` → `tempState`

**后定义的值覆盖先定义的**: 同名键以更具体作用域的值优先

---

## 内存会话服务实现

### inMemoryService 结构

**位置**: `session/inmemory.go:39-44`

```go
type inMemoryService struct {
    mu        sync.RWMutex
    sessions  omap.Map[string, *session]  // appName+userID+sessionID -> session
    userState map[string]map[string]stateMap  // appName -> userID -> state
    appState  map[string]stateMap  // appName -> state
}
```

**存储结构**:

```
inMemoryService
├── sessions: omap
│   ├── "app1|user1|session1" → session
│   ├── "app1|user1|session2" → session
│   └── "app1|user2|session1" → session
├── userState: map
│   └── "app1"
│       ├── "user1" → state{...}
│       └── "user2" → state{...}
└── appState: map
    └── "app1" → state{...}
```

---

## session 内部结构

**位置**: `session/inmemory.go:305-313`

```go
type session struct {
    id id
    
    mu        sync.RWMutex
    events    []*Event
    state     map[string]any
    updatedAt time.Time
}
```

**字段说明**:

| 字段 | 类型 | 用途 |
|------|------|------|
| `id` | id | 复合标识 (appName, userID, sessionID) |
| `mu` | sync.RWMutex | 读写锁 |
| `events` | []*Event | 事件历史列表 |
| `state` | map[string]any | 会话状态 |
| `updatedAt` | time.Time | 最后更新时间 |

### id 结构

```go
type id struct {
    appName   string
    userID    string
    sessionID string
}

func (id id) Encode() string {
    return string(ordered.Encode(id.appName, id.userID, id.sessionID))
}
```

---

## 状态存储实现

### state 结构

**位置**: `session/inmemory.go:388-426`

```go
type state struct {
    mu    *sync.RWMutex
    state map[string]any
}
```

### Get() 获取值

```go
func (s *state) Get(key string) (any, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    val, ok := s.state[key]
    if !ok {
        return nil, ErrStateKeyNotExist
    }
    
    return val, nil
}
```

**特点**:
- 读锁保护
- 键不存在返回 `ErrStateKeyNotExist` 错误

### Set() 设置值

```go
func (s *state) Set(key string, value any) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    s.state[key] = value
    return nil
}
```

**特点**:
- 写锁保护
- 直接覆盖原有值

### All() 遍历所有值

```go
func (s *state) All() iter.Seq2[string, any] {
    s.mu.RLock()
    // 克隆状态避免持锁
    stateCopy := maps.Clone(s.state)
    s.mu.RUnlock()
    
    return func(yield func(key string, val any) bool) {
        for k, v := range stateCopy {
            if !yield(k, v) {
                return
            }
        }
    }
}
```

**特点**:
- 读锁保护
- 返回克隆副本
- 使用 Go 1.22+ 泛型迭代器

---

## 状态合并机制

### mergeStates() 合并函数

**位置**: `session/inmemory.go:281-289`

```go
func (s *inMemoryService) mergeStates(state stateMap, appName, userID string) stateMap {
    appState := s.appState[appName]
    var userState stateMap
    userStateMap, ok := s.userState[appName]
    if ok {
        userState = userStateMap[userID]
    }
    return sessionutils.MergeStates(appState, userState, state)
}
```

**合并流程**:

```
mergeStates(sessionState, appName, userID)
    │
    ├─ 获取 appState (appName)
    ├─ 获取 userState (appName + userID)
    ├─ sessionutils.MergeStates(appState, userState, sessionState)
    │   │
    │   └─ 优先级: sessionState > userState > appState
    │
    └─ 返回合并后的状态
```

---

## 状态更新机制

### appendEvent 时的状态更新

**位置**: `session/inmemory.go:347-363`

```go
func (s *session) appendEvent(event *Event) error {
    if event.Partial {
        return nil  // Partial 事件不处理
    }
    
    s.mu.Lock()
    defer s.mu.Unlock()
    
    // 更新会话状态
    if err := updateSessionState(s, event); err != nil {
        return fmt.Errorf("error on appendEvent: %w", err)
    }
    
    // 移除临时状态
    processedEvent := trimTempDeltaState(event)
    
    // 追加事件
    s.events = append(s.events, processedEvent)
    s.updatedAt = event.Timestamp
    return nil
}
```

### updateSessionState() 更新状态

**位置**: `session/inmemory.go:448-461`

```go
func updateSessionState(session *session, event *Event) error {
    if event.Actions.StateDelta == nil {
        return nil
    }
    
    // 确保状态map已初始化
    if session.state == nil {
        session.state = make(map[string]any)
    }
    
    // 复制状态变更
    maps.Copy(session.state, event.Actions.StateDelta)
    return nil
}
```

---

## 临时状态处理

### trimTempDeltaState() 移除临时状态

**位置**: `session/inmemory.go:428-447`

```go
func trimTempDeltaState(event *Event) *Event {
    if len(event.Actions.StateDelta) == 0 {
        return event
    }
    
    // 过滤掉 temp: 前缀的键
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

**用途**: 持久化事件时移除临时状态，只保留 app/user/session 级别的状态

---

## 状态提取机制

### ExtractStateDeltas()

在事件追加时，系统会从 `StateDelta` 中提取不同级别的状态变更：

```go
appDelta, userDelta, sessionDelta := sessionutils.ExtractStateDeltas(event.Actions.StateDelta)
```

**提取逻辑**:

```
StateDelta
    │
    ├─ "app:key1" → appDelta
    ├─ "user:key2" → userDelta
    ├─ "key3" → sessionDelta (无前缀)
    └─ "temp:key4" → 丢弃 (临时状态)
```

---

## 会话获取和过滤

### Get() 方法

**位置**: `session/inmemory.go:95-140`

```go
func (s *inMemoryService) Get(ctx context.Context, req *GetRequest) (*GetResponse, error) {
    // ... 参数验证 ...
    
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    // 查找会话
    res, ok := s.sessions.Get(id.Encode())
    
    // 合并状态
    copiedSession.state = s.mergeStates(res.state, appName, userID)
    
    // 事件过滤
    if req.NumRecentEvents > 0 {
        start := max(len(filteredEvents)-req.NumRecentEvents, 0)
        filteredEvents = filteredEvents[start:]
    }
    if !req.After.IsZero() {
        firstIndexToKeep := sort.Search(len(filteredEvents), func(i int) bool {
            return !filteredEvents[i].Timestamp.Before(req.After)
        })
        filteredEvents = filteredEvents[firstIndexToKeep:]
    }
    
    // 返回
    return &GetResponse{
        Session: copiedSession,
    }, nil
}
```

### 过滤参数

| 参数 | 用途 |
|------|------|
| `NumRecentEvents` | 只返回最近 N 条事件 |
| `After` | 只返回指定时间之后的事件 |

---

## Events 接口

**位置**: `session/session.go:76-87`

```go
type Events interface {
    All() iter.Seq[*Event]
    Len() int
    At(i int) *Event
}
```

### events 内部实现

```go
type events []*Event

func (e events) All() iter.Seq[*Event] {
    return func(yield func(*Event) bool) {
        for _, event := range e {
            if !yield(event) {
                return
            }
        }
    }
}

func (e events) Len() int {
    return len(e)
}

func (e events) At(i int) *Event {
    if i >= 0 && i < len(e) {
        return e[i]
    }
    return nil
}
```

---

## 线程安全设计

### 锁机制

```go
type session struct {
    mu sync.RWMutex
    // ...
}

func (s *session) State() State {
    return &state{
        mu:    &s.mu,
        state: s.state,
    }
}
```

**设计原则**:
- session 级别使用 RWMutex
- state 级别共享同一个锁
- 读取时使用 RLock
- 写入时使用 Lock

### 深拷贝返回

```go
func (s *state) All() iter.Seq2[string, any] {
    s.mu.RLock()
    stateCopy := maps.Clone(s.state)  // 克隆副本
    s.mu.RUnlock()
    
    return func(yield func(key string, val any) bool) {
        for k, v := range stateCopy {  // 迭代副本
            if !yield(k, v) {
                return
            }
        }
    }
}
```

---

## 使用示例

### 创建会话

```go
resp, err := sessionService.Create(ctx, &session.CreateRequest{
    AppName: "my_app",
    UserID:  "user123",
    State: map[string]any{
        "initial_key": "initial_value",
    },
})
```

### 读取状态

```go
state := session.State()

// 获取单个值
val, err := state.Get("key")
if err != nil {
    // 处理错误
}

// 遍历所有值
for key, val := range state.All() {
    fmt.Printf("%s: %v\n", key, val)
}
```

### 修改状态

```go
state := session.State()

// 设置值
err := state.Set("key", "value")
if err != nil {
    // 处理错误
}

// 在事件中携带状态变更
event.Actions.StateDelta = map[string]any{
    "app:app_key":  "app_value",    // 应用级
    "user:user_key": "user_value",  // 用户级
    "session_key":   "session_value", // 会话级
    "temp:temp_key": "temp_value",   // 临时级（不持久化）
}
```

### 状态提取示例

```go
// 从 StateDelta 提取不同级别
appDelta, userDelta, sessionDelta := sessionutils.ExtractStateDeltas(event.Actions.StateDelta)

// appDelta: "app:" 前缀的键
// userDelta: "user:" 前缀的键
// sessionDelta: 无前缀的键
// temp: 前缀的被丢弃
```

---

## 与其他模块的关系

### Runner 状态流程

```
Runner.Run()
    │
    ├─ 获取 Session
    │   └─ mergeStates(appState + userState + sessionState)
    │
    ├─ 创建 Event (含 StateDelta)
    │   └─ StateDelta: {app:, user:, session:, temp: }
    │
    ├─ Agent 执行
    │   └─ 修改 StateDelta
    │
    └─ AppendEvent
        ├─ ExtractStateDeltas()
        ├─ updateAppState()
        ├─ updateUserState()
        ├─ updateSessionState()
        └─ trimTempDeltaState() // 移除 temp:
```

### Agent 回调状态

```go
func callback(ctx agent.CallbackContext) (*genai.Content, error) {
    state := ctx.State()
    
    // 读取
    val, _ := state.Get("key")
    
    // 写入 (会记录到 StateDelta)
    state.Set("key", "new_value")
    
    return nil, nil
}
```

---

## 关键设计决策

### 1. 分层状态管理

**设计**: app/user/session/temp 四层

**优势**:
- 灵活的状态共享
- 自动的临时状态清理
- 清晰的权限边界

### 2. 事件携带状态变更

**设计**: Event.Actions.StateDelta

**优势**:
- 状态变更可追溯
- 支持审计和回放
- 自动合并到会话状态

### 3. 前缀区分作用域

**设计**: KeyPrefixApp/KeyPrefixUser/KeyPrefixTemp

**优势**:
- 简单直观
- 易于理解和实现
- 无需额外配置

### 4. 读锁遍历

**设计**: 返回克隆副本

**优势**:
- 避免持锁遍历
- 支持并发读取
- 数据一致性

### 5. Partial 事件不处理

**设计**: appendEvent 忽略 Partial 事件

**优势**:
- 减少存储压力
- 保持事件语义清晰
- 简化状态管理

---

## 总结

ADK Go 的会话状态管理具有以下特点：

1. **分层状态**: app/user/session/temp 四级作用域
2. **前缀标识**: 使用 KeyPrefix 区分状态级别
3. **线程安全**: RWMutex + 克隆副本
4. **事件溯源**: 状态变更通过 Event 记录
5. **自动清理**: 临时状态自动移除
6. **状态合并**: 多层状态自动合并

设计优势：
- 清晰的状态层级
- 灵活的状态共享
- 完整的线程安全
- 可追溯的状态变更
- 自动的临时状态清理
