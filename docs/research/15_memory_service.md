# Memory 服务深入研究

## 概述

Memory 服务是 ADK Go 中实现**跨会话记忆**能力的关键子系统。它允许 Agent 访问用户历史会话中的信息，实现长期记忆功能。

---

## 核心架构

### 1. 接口定义 (`memory/service.go`)

```go
type Service interface {
    AddSession(ctx context.Context, s session.Session) error
    Search(ctx context.Context, req *SearchRequest) (*SearchResponse, error)
}
```

**核心方法**：
- `AddSession`: 将会话添加到记忆服务
- `Search`: 根据查询搜索相关记忆

### 2. Agent 层面的 Memory 接口 (`agent/agent.go`)

```go
type Memory interface {
    AddSession(context.Context, session.Session) error
    Search(ctx context.Context, query string) (*memory.SearchResponse, error)
}
```

**设计意图**：
- 为 Agent 提供简化的记忆访问接口
- 隐藏了 `SearchRequest` 的复杂参数（AppName, UserID）
- 通过 `InvocationContext` 自动注入上下文信息

### 3. 内部 Memory 包装器 (`internal/memory/memory.go`)

```go
type Memory struct {
    Service   memory.Service
    SessionID string
    UserID    string
    AppName   string
}

func (a *Memory) Search(ctx context.Context, query string) (*memory.SearchResponse, error) {
    return a.Service.Search(ctx, &memory.SearchRequest{
        AppName: a.AppName,
        UserID:  a.UserID,
        Query:   query,
    })
}
```

**作用**：
- 封装底层 `memory.Service`
- 自动填充 `AppName` 和 `UserID`
- 提供更简洁的 API

---

## 数据结构

### SearchRequest

```go
type SearchRequest struct {
    Query   string  // 搜索查询
    UserID  string  // 用户 ID
    AppName string  // 应用名称
}
```

### SearchResponse

```go
type SearchResponse struct {
    Memories []Entry
}
```

### Entry

```go
type Entry struct {
    Content   *genai.Content  // 记忆内容
    Author    string          // 作者
    Timestamp time.Time       // 时间戳
}
```

---

## 内存实现 (`memory/inmemory.go`)

### 存储结构

```go
type inMemoryService struct {
    mu    sync.RWMutex
    store map[key]map[sessionID][]value
}

type key struct {
    appName, userID string
}

type sessionID string

type value struct {
    content   *genai.Content
    author    string
    timestamp time.Time
    words     map[string]struct{}  // 预计算的关键词集合
}
```

**设计特点**：
1. **三级索引结构**：`AppName + UserID` → `SessionID` → `[]Event`
2. **预计算关键词**：添加会话时提取关键词，加速搜索
3. **线程安全**：使用 `sync.RWMutex` 保护并发访问

### AddSession 实现

```go
func (s *inMemoryService) AddSession(ctx context.Context, curSession session.Session) error {
    var values []value
    
    // 遍历会话中的所有事件
    for event := range curSession.Events().All() {
        if event.LLMResponse.Content == nil {
            continue
        }
        
        // 提取关键词
        words := make(map[string]struct{})
        for _, part := range event.LLMResponse.Content.Parts {
            if part.Text == "" {
                continue
            }
            maps.Copy(words, extractWords(part.Text))
        }
        
        if len(words) == 0 {
            continue
        }
        
        values = append(values, value{
            content:   event.LLMResponse.Content,
            author:    event.Author,
            timestamp: event.Timestamp,
            words:     words,
        })
    }
    
    // 存储到索引
    k := key{
        appName: curSession.AppName(),
        userID:  curSession.UserID(),
    }
    
    s.mu.Lock()
    defer s.mu.Unlock()
    
    v, ok := s.store[k]
    if !ok {
        v = map[sessionID][]value{}
        s.store[k] = v
    }
    
    sid := sessionID(curSession.ID())
    v[sid] = values
    return nil
}
```

**关键步骤**：
1. 过滤只包含 `LLMResponse.Content` 的事件
2. 提取文本内容中的关键词（小写化、分词）
3. 预计算关键词集合用于后续搜索
4. 按 `AppName + UserID + SessionID` 三级索引存储

### Search 实现

```go
func (s *inMemoryService) Search(ctx context.Context, req *SearchRequest) (*SearchResponse, error) {
    // 提取查询关键词
    queryWords := extractWords(req.Query)
    
    k := key{
        appName: req.AppName,
        userID:  req.UserID,
    }
    
    s.mu.RLock()
    values, ok := s.store[k]
    s.mu.RUnlock()
    
    if !ok {
        return &SearchResponse{}, nil
    }
    
    res := &SearchResponse{}
    
    // 遍历所有会话的所有事件
    for _, events := range values {
        for _, e := range events {
            // 检查关键词交集
            if checkMapsIntersect(e.words, queryWords) {
                res.Memories = append(res.Memories, Entry{
                    Content:   e.content,
                    Author:    e.author,
                    Timestamp: e.timestamp,
                })
            }
        }
    }
    
    return res, nil
}
```

**搜索算法**：
1. 提取查询关键词
2. 按 `AppName + UserID` 查找所有会话
3. 遍历所有事件，检查关键词交集
4. 返回包含任一查询关键词的事件

### 关键词提取

```go
func extractWords(text string) map[string]struct{} {
    res := make(map[string]struct{})
    
    for s := range strings.SplitSeq(text, " ") {
        if s == "" {
            continue
        }
        res[strings.ToLower(s)] = struct{}{}
    }
    
    return res
}
```

**特点**：
- 简单的空格分词
- 转为小写统一处理
- 无复杂 NLP 处理

### 关键词交集检查

```go
func checkMapsIntersect(m1, m2 map[string]struct{}) bool {
    if len(m1) == 0 || len(m2) == 0 {
        return false
    }
    
    // 优化：遍历较小的 map
    if len(m1) > len(m2) {
        m1, m2 = m2, m1
    }
    
    for k := range m1 {
        if _, ok := m2[k]; ok {
            return true
        }
    }
    
    return false
}
```

**优化策略**：
- 遍历较小的 map 提高效率
- 只要有一个关键词匹配即返回 true

---

## 集成与使用

### 在 Runner 中的初始化

```go
// runner/runner.go:153-161
var memoryImpl agent.Memory = nil
if r.memoryService != nil {
    memoryImpl = &imemory.Memory{
        Service:   r.memoryService,
        SessionID: storedSession.ID(),
        UserID:    storedSession.UserID(),
        AppName:   storedSession.AppName(),
    }
}
```

**注入流程**：
1. Runner 持有 `memoryService` 实例
2. 创建 `InvocationContext` 时包装为 `agent.Memory`
3. 自动填充 `AppName`、`UserID` 等上下文信息

### 在 InvocationContext 中的访问

```go
// agent/context.go:70
Memory() Memory
```

**作用域**：
- Memory 作用域为当前 `UserID` 的所有会话
- 不限定于单个会话，实现跨会话记忆

### 在 Tool 中的使用

```go
// internal/toolinternal/context.go:103-108
func (c *toolContext) SearchMemory(ctx context.Context, query string) (*memory.SearchResponse, error) {
    if c.invocationContext.Memory() == nil {
        return nil, fmt.Errorf("memory service is not set")
    }
    return c.invocationContext.Memory().Search(ctx, query)
}
```

**Tool 访问方式**：
1. 通过 `tool.Context` 接口访问
2. 调用 `SearchMemory()` 方法
3. 返回与查询相关的历史记忆

---

## 使用场景

### 场景 1：用户偏好记忆

```
会话 1:
用户: 我喜欢 Python
助手: 好的，我记住了您喜欢 Python

会话 2 (新会话):
用户: 帮我写个示例代码
助手: [搜索记忆 "代码"]
      [找到 "Python"]
      [使用 Python 编写示例]
```

### 场景 2：项目上下文记忆

```
会话 1:
用户: 我正在开发一个电商系统
助手: 好的，了解您在开发电商系统

会话 2:
用户: 如何处理订单？
助手: [搜索记忆 "订单"]
      [找到 "电商系统"]
      [结合电商场景回答订单处理方案]
```

### 场景 3：Tool 中访问记忆

```go
func myTool(ctx tool.Context, args map[string]any) (any, error) {
    // 搜索用户历史偏好
    resp, err := ctx.SearchMemory(ctx, "user preferences")
    if err != nil {
        return nil, err
    }
    
    // 使用历史记忆
    for _, memory := range resp.Memories {
        // 处理记忆内容
    }
    
    return result, nil
}
```

---

## 工作流程

### 添加会话到记忆

```
┌──────────────────────────────────────────┐
│  会话结束 / 显式调用 AddSession          │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  遍历会话中的所有事件                     │
│  - 过滤有 LLMResponse.Content 的事件     │
│  - 提取文本内容                          │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  预计算关键词集合                         │
│  - 分词（空格分隔）                      │
│  - 小写化                               │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  存储到三级索引                          │
│  [AppName + UserID] → SessionID → Events │
└──────────────────────────────────────────┘
```

### 搜索记忆流程

```
┌──────────────────────────────────────────┐
│  Tool 调用 SearchMemory(query)           │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  提取查询关键词                           │
│  - 分词、小写化                          │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  查找 [AppName + UserID] 索引            │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  遍历所有会话的所有事件                   │
│  - 检查关键词交集                        │
│  - 收集匹配的事件                        │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  返回 SearchResponse                     │
│  - 包含匹配的记忆条目                    │
│  - 保留原始内容、作者、时间戳            │
└──────────────────────────────────────────┘
```

---

## 设计亮点

### 1. 预计算优化

- 添加会话时预计算关键词集合
- 搜索时无需重复解析文本
- 以空间换时间提高搜索效率

### 2. 作用域隔离

- 按 `AppName + UserID` 隔离记忆
- 防止跨应用/用户的数据泄露
- 支持多租户场景

### 3. 简化 API

- Agent 层接口隐藏实现细节
- 自动注入上下文信息
- Tool 可直接调用无需额外配置

### 4. 灵活扩展

- `memory.Service` 接口简单清晰
- 可替换为外部存储实现（如向量数据库）
- 支持自定义搜索算法

---

## 限制与改进方向

### 当前限制

1. **简单关键词匹配**：
   - 仅支持空格分词
   - 无语义理解能力
   - 无模糊匹配

2. **内存存储**：
   - `InMemoryService` 仅适合开发测试
   - 无法持久化
   - 不支持分布式场景

3. **无权重排序**：
   - 所有匹配事件平权返回
   - 无时间衰减
   - 无相关性评分

### 改进方向

1. **集成向量数据库**：
   - 使用 Embedding 实现语义搜索
   - 支持相似度排序
   - 提高检索质量

2. **实现持久化存储**：
   - 支持 GCS、数据库等存储后端
   - 定期清理过期记忆

3. **增强搜索能力**：
   - TF-IDF 权重
   - BM25 排序
   - 时间衰减因子

4. **智能摘要**：
   - 长文本自动摘要
   - 关键信息提取
   - 减少存储和传输开销

---

## 与其他组件的关系

### 与 Session 的关系

- **Session**: 单次会话的短期记忆（事件流）
- **Memory**: 跨会话的长期记忆（搜索索引）
- **关系**: Memory 从 Session 中提取和索引信息

### 与 Tool 的关系

- Tool 通过 `tool.Context.SearchMemory()` 访问记忆
- 可在工具执行中获取历史上下文
- 增强工具的决策能力

### 与 Agent 的关系

- Agent 通过 `InvocationContext.Memory()` 访问
- 可在 Agent 回调中查询历史信息
- 支持构建具有长期记忆的智能代理

---

## 总结

Memory 服务是 ADK Go 中实现跨会话记忆的关键组件：

1. **核心功能**：存储和检索跨会话的用户交互历史
2. **设计理念**：预计算优化、作用域隔离、API 简化
3. **使用场景**：用户偏好记忆、项目上下文、Tool 增强
4. **扩展性**：接口清晰，可替换底层存储实现

通过 Memory 服务，Agent 可以"记住"用户的历史交互，提供更智能、更个性化的服务。