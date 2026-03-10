# MCP API 参考

## 包路径

```
github.com/sipeed/picoclaw/pkg/mcp
github.com/sipeed/picoclaw/pkg/tools
```

## Manager 类型

MCP 管理器负责管理所有 MCP 服务器连接。

### 定义

```go
// pkg/mcp/manager.go
type Manager struct {
    // 包含私有字段
}
```

### 方法

#### NewManager

创建新的 MCP 管理器实例。

```go
func NewManager() *Manager
```

**返回值**:
- `*Manager`: 新的管理器实例

**示例**:
```go
manager := mcp.NewManager()
```

---

#### LoadFromConfig

从完整配置加载 MCP 服务器。

```go
func (m *Manager) LoadFromConfig(ctx context.Context, cfg *config.Config) error
```

**参数**:
- `ctx context.Context`: 上下文，用于控制超时和取消
- `cfg *config.Config`: 完整的配置对象

**返回值**:
- `error`: 加载过程中的错误

**示例**:
```go
cfg, _ := config.LoadConfig("config.json")
err := manager.LoadFromConfig(ctx, cfg)
```

---

#### LoadFromMCPConfig

从 MCP 配置加载服务器（最小依赖版本）。

```go
func (m *Manager) LoadFromMCPConfig(
    ctx context.Context,
    mcpCfg config.MCPConfig,
    workspacePath string,
) error
```

**参数**:
- `ctx context.Context`: 上下文
- `mcpCfg config.MCPConfig`: MCP 配置
- `workspacePath string`: 工作区路径，用于解析相对路径

**返回值**:
- `error`: 加载错误

**示例**:
```go
mcpCfg := config.MCPConfig{
    Enabled: true,
    Servers: map[string]config.MCPServerConfig{
        "my-server": {
            Enabled: true,
            Command: "npx",
            Args:    []string{"-y", "@modelcontextprotocol/server-filesystem", "/tmp"},
        },
    },
}
err := manager.LoadFromMCPConfig(ctx, mcpCfg, "/home/user/workspace")
```

---

#### ConnectServer

连接到单个 MCP 服务器。

```go
func (m *Manager) ConnectServer(
    ctx context.Context,
    name string,
    cfg config.MCPServerConfig,
) error
```

**参数**:
- `ctx context.Context`: 上下文
- `name string`: 服务器名称（标识符）
- `cfg config.MCPServerConfig`: 服务器配置

**返回值**:
- `error`: 连接错误

**示例**:
```go
err := manager.ConnectServer(ctx, "github", config.MCPServerConfig{
    Enabled: true,
    Command: "npx",
    Args:    []string{"-y", "@modelcontextprotocol/server-github"},
    Env:     map[string]string{"GITHUB_TOKEN": "ghp_xxx"},
})
```

---

#### CallTool

调用指定服务器上的工具。

```go
func (m *Manager) CallTool(
    ctx context.Context,
    serverName, toolName string,
    arguments map[string]any,
) (*mcp.CallToolResult, error)
```

**参数**:
- `ctx context.Context`: 上下文
- `serverName string`: 服务器名称
- `toolName string`: 工具名称
- `arguments map[string]any`: 工具参数

**返回值**:
- `*mcp.CallToolResult`: 调用结果
- `error`: 调用错误

**示例**:
```go
result, err := manager.CallTool(ctx, "filesystem", "read_file", map[string]any{
    "path": "/tmp/test.txt",
})
if err != nil {
    log.Fatal(err)
}
fmt.Println(result.Content)
```

---

#### GetServers

获取所有已连接的服务器。

```go
func (m *Manager) GetServers() map[string]*ServerConnection
```

**返回值**:
- `map[string]*ServerConnection`: 服务器名称到连接的映射

**示例**:
```go
servers := manager.GetServers()
for name, conn := range servers {
    fmt.Printf("Server: %s, Tools: %d\n", name, len(conn.Tools))
}
```

---

#### GetServer

获取指定名称的服务器连接。

```go
func (m *Manager) GetServer(name string) (*ServerConnection, bool)
```

**参数**:
- `name string`: 服务器名称

**返回值**:
- `*ServerConnection`: 服务器连接
- `bool`: 是否存在

**示例**:
```go
conn, ok := manager.GetServer("filesystem")
if !ok {
    fmt.Println("Server not found")
}
```

---

#### GetAllTools

获取所有服务器的所有工具。

```go
func (m *Manager) GetAllTools() map[string][]*mcp.Tool
```

**返回值**:
- `map[string][]*mcp.Tool`: 服务器名称到工具列表的映射

**示例**:
```go
allTools := manager.GetAllTools()
for serverName, tools := range allTools {
    for _, tool := range tools {
        fmt.Printf("[%s] %s: %s\n", serverName, tool.Name, tool.Description)
    }
}
```

---

#### Close

关闭所有服务器连接。

```go
func (m *Manager) Close() error
```

**返回值**:
- `error`: 关闭过程中的错误

**示例**:
```go
defer manager.Close()
```

---

## ServerConnection 类型

表示与单个 MCP 服务器的连接。

### 定义

```go
type ServerConnection struct {
    Name    string              // 服务器名称
    Client  *mcp.Client         // MCP 客户端
    Session *mcp.ClientSession  // 客户端会话
    Tools   []*mcp.Tool         // 服务器提供的工具列表
}
```

## MCPTool 类型

将 MCP 工具适配到 PicoClaw 工具系统的包装器。

### 定义

```go
// pkg/tools/mcp_tool.go
type MCPTool struct {
    // 包含私有字段
}
```

### 构造函数

#### NewMCPTool

创建新的 MCP 工具包装器。

```go
func NewMCPTool(manager MCPManager, serverName string, tool *mcp.Tool) *MCPTool
```

**参数**:
- `manager MCPManager`: MCP 管理器接口
- `serverName string`: 服务器名称
- `tool *mcp.Tool`: 原始 MCP 工具

**返回值**:
- `*MCPTool`: 工具包装器实例

---

### 方法

#### Name

返回规范化的工具名称。

```go
func (t *MCPTool) Name() string
```

**说明**:
- 格式: `mcp_{server}_{tool}`
- 自动处理特殊字符和长度限制（最大 64 字符）
- 必要时添加哈希后缀确保唯一性

**示例**:
```go
tool := NewMCPTool(manager, "my-server", mcpTool)
fmt.Println(tool.Name()) // 输出: mcp_my_server_some_tool
```

---

#### Description

返回工具描述。

```go
func (t *MCPTool) Description() string
```

**说明**: 描述格式为 `[MCP:{server}] {original_description}`

**示例**:
```go
fmt.Println(tool.Description()) // 输出: [MCP:my-server] Read file contents
```

---

#### Parameters

返回工具参数的 JSON Schema。

```go
func (t *MCPTool) Parameters() map[string]any
```

**返回值**:
- `map[string]any`: JSON Schema 格式的参数定义

**示例**:
```go
params := tool.Parameters()
// {
//   "type": "object",
//   "properties": {
//     "path": {"type": "string", "description": "File path"}
//   },
//   "required": ["path"]
// }
```

---

#### Execute

执行工具调用。

```go
func (t *MCPTool) Execute(ctx context.Context, args map[string]any) *ToolResult
```

**参数**:
- `ctx context.Context`: 上下文
- `args map[string]any`: 工具参数

**返回值**:
- `*ToolResult`: 执行结果

**示例**:
```go
result := tool.Execute(ctx, map[string]any{
    "path": "/tmp/test.txt",
})
if result.IsError {
    fmt.Println("Error:", result.ForLLM)
} else {
    fmt.Println("Result:", result.ForLLM)
}
```

---

## MCPManager 接口

定义 MCP 管理器的最小接口，便于测试和模拟。

```go
type MCPManager interface {
    CallTool(
        ctx context.Context,
        serverName, toolName string,
        arguments map[string]any,
    ) (*mcp.CallToolResult, error)
}
```

## 配置类型

### MCPConfig

```go
type MCPConfig struct {
    ToolConfig                      // 包含 Enabled bool
    Servers map[string]MCPServerConfig `json:"servers,omitempty"`
}
```

### MCPServerConfig

```go
type MCPServerConfig struct {
    Enabled bool              `json:"enabled"`
    Command string            `json:"command"`
    Args    []string          `json:"args,omitempty"`
    Env     map[string]string `json:"env,omitempty"`
    EnvFile string            `json:"env_file,omitempty"`
    Type    string            `json:"type,omitempty"`      // stdio, sse, http
    URL     string            `json:"url,omitempty"`
    Headers map[string]string `json:"headers,omitempty"`
}
```

## 使用示例

### 完整工作流

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/sipeed/picoclaw/pkg/config"
    "github.com/sipeed/picoclaw/pkg/mcp"
    "github.com/sipeed/picoclaw/pkg/tools"
)

func main() {
    ctx := context.Background()

    // 1. 创建管理器
    manager := mcp.NewManager()
    defer manager.Close()

    // 2. 配置服务器
    mcpCfg := config.MCPConfig{
        ToolConfig: config.ToolConfig{Enabled: true},
        Servers: map[string]config.MCPServerConfig{
            "filesystem": {
                Enabled: true,
                Command: "npx",
                Args:    []string{"-y", "@modelcontextprotocol/server-filesystem", "/tmp"},
            },
        },
    }

    // 3. 加载配置
    if err := manager.LoadFromMCPConfig(ctx, mcpCfg, ""); err != nil {
        log.Fatal(err)
    }

    // 4. 获取所有工具
    allTools := manager.GetAllTools()
    for serverName, mcpTools := range allTools {
        for _, mcpTool := range mcpTools {
            // 5. 包装为 PicoClaw 工具
            tool := tools.NewMCPTool(manager, serverName, mcpTool)
            fmt.Printf("Tool: %s\n", tool.Name())
            fmt.Printf("Description: %s\n", tool.Description())
        }
    }

    // 6. 调用工具
    result, err := manager.CallTool(ctx, "filesystem", "list_directory", map[string]any{
        "path": "/tmp",
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Result:", result.Content)
}
```
