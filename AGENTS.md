# Agent Development Kit (ADK) for Go - Agent Guidelines

本文档为在 ADK Go 代码库中工作的代码代理提供指导方针。

## 项目概览

- **项目类型**: Go 语言项目 (Agent Development Kit)
- **模块路径**: `google.golang.org/adk`
- **Go 版本**: 1.24.4+
- **主要依赖**: Google Cloud AI/Storage, Gemini, Gorilla Mux, OpenTelemetry

## 构建、测试和Lint命令

### 基本构建命令
```bash
# 构建所有包
go build -mod=readonly -v ./...

# 清理构建缓存
go clean -cache

# 下载依赖
go mod download
```

### 测试命令

#### 运行所有测试
```bash
go test -race -mod=readonly -v -count=1 -shuffle=on ./...
```

#### 运行单个测试文件
```bash
go test -mod=readonly -v -count=1 ./agent
go test -mod=readonly -v -count=1 ./agent/llmagent
```

#### 运行特定测试函数
```bash
go test -mod=readonly -v -count=1 ./agent -run TestAgentCallbacks
go test -mod=readonly -v -count=1 ./agent/llmagent -run TestLLMAgent_Run
```

#### HTTP Recording测试
```bash
# 自动生成HTTP记录
go test -httprecord=Test ./model/gemini
go test -httprecord=testdata/.*\.httprr ./model/gemini
```

#### 覆盖率测试
```bash
go test -mod=readonly -v -count=1 -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html
```

### Lint和格式检查

#### 运行所有linters
```bash
golangci-lint run
```

#### 使用项目特定的golangci-lint配置
```bash
golangci-lint run --config=.golangci.yml
```

#### 单独运行格式化工具
```bash
goimports -w .
gofumpt -w .
```

## 代码风格指南

### 遵循的规范
- **主要规范**: [Google Go Style Guide](https://google.github.io/styleguide/go/index)
- **格式化工具**: gofumpt (extra-rules), goimports
- **本地前缀**: `google.golang.org/adk`

### 文件头部要求
所有 `.go` 文件必须包含标准版权头：

```go
// Copyright 2025 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
```

### 命名约定

#### 包命名
- 使用小写、单数形式的包名
- 避免下划线和混合大小写
- 示例: `agent`, `model`, `session`, `tool`

#### 函数和变量命名
- 使用驼峰命名法 (camelCase)
- 导出函数使用大写字母开头
- 非导出函数使用小写字母开头
- 避免缩写，保持清晰描述性

#### 常量命名
- 使用驼峰命名法
- 如果是导出常量，使用大写开头
- 示例: `MaxRetries`, `DefaultTimeout`

#### 结构体和接口
- 结构体: `AgentConfig`, `LLMAgent`
- 接口: 以 "er" 结尾或描述性名称，如 `Agent`, `Tool`, `Formatter`

### Import组织规范

#### 导入顺序
1. 标准库导入
2. 第三方库导入
3. 项目内部导入 (`google.golang.org/adk`)

#### Import分组和格式化
```go
import (
    "context"
    "iter"
    "testing"

    "github.com/google/go-cmp/cmp"
    "github.com/google/go-cmp/cmp/cmpopts"
    "google.golang.org/genai"

    "google.golang.org/adk/model"
    "google.golang.org/adk/session"
)
```

### 错误处理

#### 错误创建和返回
- 使用 `errors.New` 或 `fmt.Errorf`
- 提供有意义的错误消息
- 避免在错误消息中暴露敏感信息

#### 错误检查
```go
if err != nil {
    return fmt.Errorf("failed to create agent: %w", err)
}
```

#### 自定义错误类型
```go
type ConfigError struct {
    Field   string
    Message string
}

func (e *ConfigError) Error() string {
    return fmt.Sprintf("config error in %s: %s", e.Field, e.Message)
}
```

### 并发编程

#### Goroutine管理
- 总是使用 `sync.WaitGroup` 或 context 来管理 goroutines
- 使用 `t.Parallel()` 进行测试并发
- 避免 goroutine 泄漏

#### Channel使用
- 明确指定 channel 方向
- 使用 buffered channels 避免死锁
- 总是检查 channel 是否已关闭

### 测试最佳实践

#### 测试结构
```go
func TestFunction(t *testing.T) {
    t.Parallel()
    
    tests := []struct {
        name string
        // 测试用例字段
    }{
        {
            name: "descriptive test name",
            // 测试逻辑
        },
    }
    
    for _, tt := range tests {
        tt := tt // 避免闭包问题
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            // 测试逻辑
        })
    }
}
```

#### Mock和Fixture
- 使用 mock 对象隔离外部依赖
- 利用 HTTP recording 测试 (`go test -httprecord=...`)
- 创建可重用的测试 fixtures

#### 测试覆盖率
- 目标: 新代码覆盖率 >80%
- 覆盖所有代码路径，包括错误处理
- 测试边界条件和边缘案例

### 代码审查要求

#### PR要求
- 每个 PR 必须关联一个 Issue
- 遵循小而集中的更改原则
- 包含测试计划和 E2E 测试证据
- 遵循 Google Go Style Guide

#### 测试要求
- **单元测试**: 覆盖新功能、错误条件和边缘案例
- **E2E测试**: 对于集成功能提供手动测试证明
- **文档更新**: 用户Facing更改需要同步更新文档

### 性能和内存

#### 性能优化
- 使用 `sync.Pool` 重用对象
- 避免不必要的内存分配
- 使用 buffered I/O
- 考虑内存对齐和缓存友好性

#### 内存管理
- 及时释放不需要的资源
- 使用 `defer` 确保资源清理
- 避免内存泄漏

### 安全最佳实践

- 总是验证输入参数
- 避免泄露敏感信息在错误消息中
- 使用安全的随机数生成器
- 遵循最小权限原则

### Linters配置

项目使用 `.golangci.yml` 配置，包含：
- `goheader`: 版权头检查
- `goimports`: 导入格式化
- `gofumpt`: Go 代码格式化
- `staticcheck`: 静态分析
- 特殊规则: 排除 `internal/httprr/` 目录的某些检查

### 贡献流程

1. **Issue First**: 大型更改需要先创建 Issue 讨论
2. **PR审查**: 所有更改需要 PR 审查
3. **测试**: 必须包含测试证明
4. **文档**: 相关文档更新同步进行
5. **对齐**: 与 adk-python 项目保持一致性

### 注意事项

- `internal/httprr` 包有特殊的许可条款和linting豁免
- 项目使用 HTTP recording 进行端到端测试
- 优先与 adk-python 保持 API 一致性
- 所有外部依赖必须经过安全审查