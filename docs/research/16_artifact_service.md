# Artifact 服务深入研究

## 概述

Artifact 服务是 ADK Go 中实现**文件资源管理**的关键子系统。它支持在会话中存储、加载、删除和版本控制文件资源（artifacts），为 Agent 和 Tool 提供文件操作能力。

---

## 核心架构

### 1. 接口定义 (`artifact/service.go`)

```go
type Service interface {
    Save(ctx context.Context, req *SaveRequest) (*SaveResponse, error)
    Load(ctx context.Context, req *LoadRequest) (*LoadResponse, error)
    Delete(ctx context.Context, req *DeleteRequest) error
    List(ctx context.Context, req *ListRequest) (*ListResponse, error)
    Versions(ctx context.Context, req *VersionsRequest) (*VersionsResponse, error)
}
```

**核心方法**：
- `Save`: 保存 artifact，返回版本号
- `Load`: 加载 artifact，支持指定版本
- `Delete`: 删除 artifact，支持删除特定版本或全部版本
- `List`: 列出会话中的所有文件名
- `Versions`: 列出 artifact 的所有版本号

### 2. Agent 层面的 Artifacts 接口 (`agent/agent.go`)

```go
type Artifacts interface {
    Save(ctx context.Context, name string, data *genai.Part) (*artifact.SaveResponse, error)
    List(context.Context) (*artifact.ListResponse, error)
    Load(ctx context.Context, name string) (*artifact.LoadResponse, error)
    LoadVersion(ctx context.Context, name string, version int) (*artifact.LoadResponse, error)
}
```

**设计意图**：
- 为 Agent 和 Tool 提供简化的文件操作接口
- 隐藏了 `Request/Response` 的复杂参数
- 通过 `InvocationContext` 自动注入上下文信息

### 3. 内部 Artifacts 包装器 (`internal/artifact/artifacts.go`)

```go
type Artifacts struct {
    Service   artifact.Service
    AppName   string
    UserID    string
    SessionID string
}

func (a *Artifacts) Save(ctx context.Context, name string, data *genai.Part) (*artifact.SaveResponse, error) {
    return a.Service.Save(ctx, &artifact.SaveRequest{
        AppName:   a.AppName,
        UserID:    a.UserID,
        SessionID: a.SessionID,
        FileName:  name,
        Part:      data,
    })
}
```

**作用**：
- 封装底层 `artifact.Service`
- 自动填充 `AppName`、`UserID`、`SessionID`
- 提供更简洁的 API

---

## 数据结构

### 请求类型

#### SaveRequest

```go
type SaveRequest struct {
    AppName, UserID, SessionID, FileName string
    Part *genai.Part  // artifact 内容
    
    // 可选字段
    Version int64  // 指定版本号（不设置则自动递增）
}
```

#### LoadRequest

```go
type LoadRequest struct {
    AppName, UserID, SessionID, FileName string
    
    // 可选字段
    Version int64  // 不设置则加载最新版本
}
```

#### DeleteRequest

```go
type DeleteRequest struct {
    AppName, UserID, SessionID, FileName string
    
    // 可选字段
    Version int64  // 不设置则删除所有版本
}
```

#### ListRequest

```go
type ListRequest struct {
    AppName, UserID, SessionID string
}
```

#### VersionsRequest

```go
type VersionsRequest struct {
    AppName, UserID, SessionID, FileName string
}
```

### 响应类型

#### SaveResponse

```go
type SaveResponse struct {
    Version int64  // 新创建的版本号
}
```

#### LoadResponse

```go
type LoadResponse struct {
    Part *genai.Part  // artifact 内容
}
```

#### ListResponse

```go
type ListResponse struct {
    FileNames []string  // 文件名列表
}
```

#### VersionsResponse

```go
type VersionsResponse struct {
    Versions []int64  // 版本号列表
}
```

---

## 内存实现 (`artifact/inmemory.go`)

### 存储结构

```go
type inMemoryService struct {
    mu sync.RWMutex
    artifacts omap.Map[string, *genai.Part]  // 有序 map
}

type artifactKey struct {
    AppName   string
    UserID    string
    SessionID string
    FileName  string
    Version   int64
}
```

**设计特点**：
1. **有序存储**：使用 `omap.Map` 实现有序键值对
2. **复合键编码**：使用 `ordered.Encode` 将结构体编码为字符串键
3. **版本倒序**：版本号使用 `ordered.Rev` 反转，确保最新版本在最前
4. **线程安全**：使用 `sync.RWMutex` 保护并发访问

### 键编码机制

```go
func (ak artifactKey) Encode() string {
    return string(ordered.Encode(
        ak.AppName, 
        ak.UserID, 
        ak.SessionID, 
        ak.FileName, 
        ordered.Rev(ak.Version),  // 版本反转
    ))
}
```

**编码效果**：
- 键格式：`appName/userID/sessionID/fileName/version`
- 有序性保证：相同前缀的键连续存储
- 版本倒序：版本号大的排在前面（便于查找最新版本）

### Save 实现

```go
func (s *inMemoryService) Save(ctx context.Context, req *SaveRequest) (*SaveResponse, error) {
    err := req.Validate()
    if err != nil {
        return nil, fmt.Errorf("request validation failed: %w", err)
    }
    
    appName, userID, sessionID, fileName := req.AppName, req.UserID, req.SessionID, req.FileName
    artifact := req.Part
    
    // 处理用户作用域文件
    if fileHasUserNamespace(fileName) {
        sessionID = userScopedArtifactKey  // "user"
    }
    
    s.mu.Lock()
    defer s.mu.Unlock()
    
    // 查找最新版本号
    nextVersion := int64(1)
    if internalVer, _, ok := s.find(appName, userID, sessionID, fileName); ok {
        nextVersion = internalVer + 1
    }
    
    // 存储新版本
    s.set(appName, userID, sessionID, fileName, nextVersion, artifact)
    return &SaveResponse{Version: nextVersion}, nil
}
```

**关键步骤**：
1. 验证请求参数
2. 处理用户作用域文件（`user:` 前缀）
3. 查找最新版本号并递增
4. 存储新版本

### Load 实现

```go
func (s *inMemoryService) Load(ctx context.Context, req *LoadRequest) (*LoadResponse, error) {
    err := req.Validate()
    if err != nil {
        return nil, fmt.Errorf("request validation failed: %w", err)
    }
    
    appName, userID, sessionID, fileName := req.AppName, req.UserID, req.SessionID, req.FileName
    version := req.Version
    
    // 处理用户作用域文件
    if fileHasUserNamespace(fileName) {
        sessionID = userScopedArtifactKey
    }
    
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    // 加载特定版本
    if version > 0 {
        artifact, ok := s.get(appName, userID, sessionID, fileName, version)
        if !ok {
            return nil, fmt.Errorf("artifact not found: %w", fs.ErrNotExist)
        }
        return &LoadResponse{Part: artifact}, nil
    }
    
    // 加载最新版本
    _, artifact, ok := s.find(appName, userID, sessionID, fileName)
    if !ok {
        return nil, fmt.Errorf("artifact not found: %w", fs.ErrNotExist)
    }
    return &LoadResponse{Part: artifact}, nil
}
```

**加载逻辑**：
- 指定版本号：直接获取
- 不指定版本号：查找最新版本（版本倒序，第一个即最新）

### Delete 实现

```go
func (s *inMemoryService) Delete(ctx context.Context, req *DeleteRequest) error {
    err := req.Validate()
    if err != nil {
        return fmt.Errorf("request validation failed: %w", err)
    }
    
    appName, userID, sessionID, fileName := req.AppName, req.UserID, req.SessionID, req.FileName
    version := req.Version
    
    // 处理用户作用域文件
    if fileHasUserNamespace(fileName) {
        sessionID = userScopedArtifactKey
    }
    
    s.mu.Lock()
    defer s.mu.Unlock()
    
    // 删除特定版本
    if version != 0 {
        s.delete(appName, userID, sessionID, fileName, version)
        return nil
    }
    
    // 删除所有版本
    lo := artifactKey{AppName: appName, UserID: userID, SessionID: sessionID, FileName: fileName, Version: math.MaxInt64}.Encode()
    hi := artifactKey{AppName: appName, UserID: userID, SessionID: sessionID, FileName: fileName}.Encode()
    s.artifacts.DeleteRange(lo, hi)
    return nil
}
```

**删除逻辑**：
- 指定版本号：删除单个版本
- 不指定版本号：删除所有版本（使用范围删除）

### List 实现

```go
func (s *inMemoryService) List(ctx context.Context, req *ListRequest) (*ListResponse, error) {
    err := req.Validate()
    if err != nil {
        return nil, fmt.Errorf("request validation failed: %w", err)
    }
    
    appName, userID, sessionID := req.AppName, req.UserID, req.SessionID
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    files := map[string]bool{}
    
    // 扫描会话作用域文件
    lo := artifactKey{AppName: appName, UserID: userID, SessionID: sessionID}.Encode()
    hi := artifactKey{AppName: appName, UserID: userID, SessionID: sessionID + "\x00"}.Encode()
    for key := range s.scan(lo, hi) {
        if key.SessionID != sessionID {
            continue
        }
        files[key.FileName] = true
    }
    
    // 扫描用户作用域文件
    userScopeLo := artifactKey{AppName: appName, UserID: userID, SessionID: userScopedArtifactKey}.Encode()
    userScopeHi := artifactKey{AppName: appName, UserID: userID, SessionID: userScopedArtifactKey + "\x00"}.Encode()
    for key := range s.scan(userScopeLo, userScopeHi) {
        if key.SessionID != userScopedArtifactKey {
            continue
        }
        files[key.FileName] = true
    }
    
    filenames := slices.Collect(maps.Keys(files))
    sort.Strings(filenames)
    return &ListResponse{FileNames: filenames}, nil
}
```

**列出逻辑**：
- 扫描会话作用域文件
- 扫描用户作用域文件
- 合并去重并排序返回

### Versions 实现

```go
func (s *inMemoryService) Versions(ctx context.Context, req *VersionsRequest) (*VersionsResponse, error) {
    err := req.Validate()
    if err != nil {
        return nil, fmt.Errorf("request validation failed: %w", err)
    }
    
    appName, userID, sessionID, fileName := req.AppName, req.UserID, req.SessionID, req.FileName
    if fileHasUserNamespace(fileName) {
        sessionID = userScopedArtifactKey
    }
    
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    var versions []int64
    lo := artifactKey{AppName: appName, UserID: userID, SessionID: sessionID, FileName: fileName, Version: math.MaxInt64}.Encode()
    hi := artifactKey{AppName: appName, UserID: userID, SessionID: sessionID, FileName: fileName}.Encode()
    
    // 扫描所有版本
    for key := range s.scan(lo, hi) {
        versions = append(versions, key.Version)
    }
    
    if len(versions) == 0 {
        return nil, fmt.Errorf("artifact not found: %w", fs.ErrNotExist)
    }
    return &VersionsResponse{Versions: versions}, nil
}
```

---

## 用户作用域文件

### 概念

- **会话作用域**：文件仅在特定会话中可见
- **用户作用域**：文件在用户的所有会话中可见

### 实现机制

```go
const userScopedArtifactKey = "user"

func fileHasUserNamespace(filename string) bool {
    return strings.HasPrefix(filename, "user:")
}
```

**命名规则**：
- 会话作用域：任意文件名，如 `report.pdf`
- 用户作用域：以 `user:` 前缀开头，如 `user:preferences.json`

**存储路径**：
- 会话作用域：`appName/userID/sessionID/fileName/version`
- 用户作用域：`appName/userID/user/user:fileName/version`

**访问范围**：
- 会话作用域：仅当前会话可访问
- 用户作用域：该用户的所有会话均可访问

---

## GCS 实现 (`artifact/gcsartifact/`)

### 存储结构

```go
func buildBlobName(appName, userID, sessionID, fileName string, version int64) string {
    if fileHasUserNamespace(fileName) {
        return fmt.Sprintf("%s/%s/user/%s/%d", appName, userID, fileName, version)
    }
    return fmt.Sprintf("%s/%s/%s/%s/%d", appName, userID, sessionID, fileName, version)
}
```

**GCS 对象路径**：
- 会话作用域：`{appName}/{userID}/{sessionID}/{fileName}/{version}`
- 用户作用域：`{appName}/{userID}/user/{fileName}/{version}`

### 初始化

```go
func NewService(ctx context.Context, bucketName string, opts ...option.ClientOption) (artifact.Service, error) {
    storageClient, err := storage.NewClient(ctx, opts...)
    if err != nil {
        return nil, fmt.Errorf("failed to create gcs service: %w", err)
    }
    
    clientWrapper := &gcsClientWrapper{client: storageClient}
    s := &gcsService{
        bucketName:    bucketName,
        storageClient: clientWrapper,
        bucket:        clientWrapper.bucket(bucketName),
    }
    return s, nil
}
```

**特点**：
- 使用 GCS Bucket 作为存储后端
- 支持自定义 ClientOption
- 适合生产环境部署

---

## 集成与使用

### 在 Runner 中的初始化

```go
// runner/runner.go:143-151
var artifacts agent.Artifacts
if r.artifactService != nil {
    artifacts = &artifactinternal.Artifacts{
        Service:   r.artifactService,
        SessionID: storedSession.ID(),
        AppName:   storedSession.AppName(),
        UserID:    storedSession.UserID(),
    }
}
```

**注入流程**：
1. Runner 持有 `artifactService` 实例
2. 创建 `InvocationContext` 时包装为 `agent.Artifacts`
3. 自动填充 `AppName`、`UserID`、`SessionID`

### 在 Tool 中的使用

```go
// internal/toolinternal/context.go:38-49
type internalArtifacts struct {
    agent.Artifacts
    eventActions *session.EventActions
}

func (ia *internalArtifacts) Save(ctx context.Context, name string, data *genai.Part) (*artifact.SaveResponse, error) {
    resp, err := ia.Artifacts.Save(ctx, name, data)
    if err != nil {
        return resp, err
    }
    if ia.eventActions != nil {
        if ia.eventActions.ArtifactDelta == nil {
            ia.eventActions.ArtifactDelta = make(map[string]int64)
        }
        ia.eventActions.ArtifactDelta[name] = resp.Version
    }
    return resp, nil
}
```

**Tool 保存文件时**：
1. 调用底层 `artifact.Service.Save`
2. 记录文件版本到 `EventActions.ArtifactDelta`
3. 版本信息随 Event 一起持久化

### 在 Agent 中的访问

```go
// agent/context.go:67
Artifacts() Artifacts
```

**访问方式**：
- Agent 通过 `InvocationContext.Artifacts()` 访问
- Tool 通过 `tool.Context.Artifacts()` 访问

---

## 使用场景

### 场景 1：Tool 生成文件

```
用户: 帮我生成一份 PDF 报告
Tool: [调用 artifacts.Save("report.pdf", pdfData)]
      [返回版本号: 1]
助手: 报告已生成，版本号为 1
```

### 场景 2：Tool 读取历史文件

```
用户: 读取上次的报告
Tool: [调用 artifacts.Load("report.pdf")]
      [获取最新版本的内容]
助手: 这是报告内容...
```

### 场景 3：版本控制

```
用户: 修改报告
Tool: [调用 artifacts.Save("report.pdf", newPdfData)]
      [返回版本号: 2]
助手: 报告已更新，新版本号为 2

用户: 查看所有版本
Tool: [调用 artifacts.Versions("report.pdf")]
      [返回 [2, 1]]
助手: 报告有 2 个版本

用户: 恢复到版本 1
Tool: [调用 artifacts.LoadVersion("report.pdf", 1)]
      [获取版本 1 的内容]
助手: 已恢复到版本 1
```

### 场景 4：用户偏好设置

```
用户: 设置我的语言偏好为中文
Tool: [调用 artifacts.Save("user:preferences.json", {lang: "zh"})]
      [存储在用户作用域]
助手: 偏好已保存

[新会话]
Tool: [调用 artifacts.Load("user:preferences.json")]
      [跨会话读取用户偏好]
助手: 检测到您的语言偏好是中文
```

### 场景 5：列出会话文件

```
用户: 我上传了哪些文件？
Tool: [调用 artifacts.List()]
      [返回 ["report.pdf", "data.csv", "user:preferences.json"]]
助手: 您有 3 个文件
```

---

## 工作流程

### 保存文件流程

```
┌──────────────────────────────────────────┐
│  Tool/Agent 调用 Save(name, data)        │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  验证请求参数                             │
│  - 必填字段检查                          │
│  - 文件名不能包含路径分隔符              │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  判断文件作用域                           │
│  - user: 前缀 → 用户作用域               │
│  - 否则 → 会话作用域                     │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  查找最新版本号                           │
│  - 搜索范围：同文件名的所有版本          │
│  - 版本号递增 +1                         │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  存储文件内容                             │
│  - 编码键：appName/userID/sessionID/     │
│            fileName/version               │
│  - 存储值：genai.Part                    │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  返回新版本号                             │
│  - Tool 记录到 EventActions.ArtifactDelta│
└──────────────────────────────────────────┘
```

### 加载文件流程

```
┌──────────────────────────────────────────┐
│  Tool/Agent 调用 Load(name) 或           │
│  LoadVersion(name, version)              │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  验证请求参数                             │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  判断文件作用域                           │
└────────────────┬─────────────────────────┘
                 │
                 ▼
        ┌────────┴────────┐
        │                 │
        ▼                 ▼
  指定版本            不指定版本
        │                 │
        ▼                 ▼
  直接获取            查找最新版本
  特定版本            (版本倒序第一个)
        │                 │
        └────────┬────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  返回文件内容                             │
│  - 不存在则返回 fs.ErrNotExist           │
└──────────────────────────────────────────┘
```

---

## 设计亮点

### 1. 版本控制

- 自动版本号递增
- 支持加载任意历史版本
- 支持列出所有版本
- 支持删除特定版本或全部版本

### 2. 双重作用域

- **会话作用域**：文件仅在当前会话可见
- **用户作用域**：文件在用户所有会话可见
- 通过文件名前缀 `user:` 区分

### 3. 有序存储

- 使用 `omap.Map` 实现有序键值对
- 版本倒序排列，便于查找最新版本
- 支持范围扫描和删除

### 4. 简化 API

- Agent 层接口隐藏实现细节
- 自动注入上下文信息
- Tool 可直接调用无需额外配置

### 5. 灵活扩展

- `artifact.Service` 接口简单清晰
- 可替换为外部存储实现（如 GCS、数据库）
- 支持分布式场景

---

## 与其他组件的关系

### 与 Session 的关系

- Artifact 存储在会话上下文中
- 文件版本信息记录在 `EventActions.ArtifactDelta`
- 支持会话级别的文件隔离

### 与 Event 的关系

- Tool 保存文件时，版本信息写入 Event
- Event 记录了文件变更历史
- 支持事件溯源和审计

### 与 Tool 的关系

- Tool 通过 `tool.Context.Artifacts()` 访问
- 可在工具执行中读写文件
- 增强工具的数据处理能力

### 与 Agent 的关系

- Agent 通过 `InvocationContext.Artifacts()` 访问
- 可在 Agent 回调中管理文件
- 支持构建具有文件处理能力的智能代理

---

## 限制与改进方向

### 当前限制

1. **内存存储**：
   - `InMemoryService` 仅适合开发测试
   - 无法持久化
   - 不支持分布式场景

2. **文件大小限制**：
   - 内存实现受限于可用内存
   - GCS 实现受限于网络带宽

3. **无文件类型限制**：
   - 不限制文件类型和大小
   - 缺少上传验证和过滤

### 改进方向

1. **多种存储后端**：
   - 数据库存储（PostgreSQL、MongoDB）
   - 对象存储（AWS S3、Azure Blob）
   - 分布式文件系统

2. **增强安全**：
   - 文件类型白名单
   - 文件大小限制
   - 病毒扫描

3. **性能优化**：
   - 大文件分块上传
   - 断点续传
   - 缓存机制

4. **元数据管理**：
   - 文件描述、标签
   - 创建时间、修改时间
   - 访问权限控制

---

## 总结

Artifact 服务是 ADK Go 中实现文件资源管理的关键组件：

1. **核心功能**：存储、加载、删除、版本控制文件资源
2. **设计理念**：版本控制、双重作用域、有序存储、API 简化
3. **使用场景**：Tool 生成文件、读取历史文件、用户偏好、版本管理
4. **扩展性**：接口清晰，可替换底层存储实现

通过 Artifact 服务，Agent 和 Tool 可以方便地管理文件资源，实现更丰富的数据处理能力。版本控制机制保证了文件的历史可追溯性，用户作用域支持跨会话共享文件。