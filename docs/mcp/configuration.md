# MCP 配置指南

## 配置结构

MCP 配置位于 `tools.mcp` 配置节点下，包含全局启用开关和服务器配置映射。

### 配置字段

```go
// MCPConfig 定义所有 MCP 服务器的配置
type MCPConfig struct {
    ToolConfig                    // 包含 Enabled 字段
    Servers map[string]MCPServerConfig  // 服务器配置映射
}

// MCPServerConfig 定义单个 MCP 服务器的配置
type MCPServerConfig struct {
    Enabled bool                   // 是否启用此服务器
    Command string                 // 可执行命令 (stdio)
    Args    []string               // 命令参数
    Env     map[string]string      // 环境变量
    EnvFile string                 // 环境变量文件路径
    Type    string                 // 传输类型: stdio, sse, http
    URL     string                 // SSE/HTTP 端点 URL
    Headers map[string]string      // HTTP 请求头 (sse/http)
}
```

## 传输类型

### stdio (默认)

通过标准输入/输出与本地 MCP 服务器进程通信。

```json
{
  "tools": {
    "mcp": {
      "enabled": true,
      "servers": {
        "filesystem": {
          "enabled": true,
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
          "env": {
            "NODE_ENV": "production"
          }
        }
      }
    }
  }
}
```

**字段说明**:
- `command`: 可执行文件路径或命令名称
- `args`: 命令行参数数组
- `env`: 传递给进程的环境变量
- `env_file`: `.env` 格式的环境变量文件路径（相对于 workspace）

### SSE (Server-Sent Events)

通过 SSE 协议与远程 MCP 服务器通信。

```json
{
  "tools": {
    "mcp": {
      "enabled": true,
      "servers": {
        "remote-api": {
          "enabled": true,
          "type": "sse",
          "url": "https://api.example.com/mcp/sse",
          "headers": {
            "Authorization": "Bearer YOUR_API_KEY",
            "X-Custom-Header": "value"
          }
        }
      }
    }
  }
}
```

**字段说明**:
- `type`: 必须设置为 `"sse"`
- `url`: SSE 端点的完整 URL
- `headers`: 自定义 HTTP 请求头（可选）

### HTTP

通过 HTTP 请求与远程 MCP 服务器通信。

```json
{
  "tools": {
    "mcp": {
      "enabled": true,
      "servers": {
        "http-api": {
          "enabled": true,
          "type": "http",
          "url": "https://api.example.com/mcp",
          "headers": {
            "Authorization": "Bearer YOUR_TOKEN"
          }
        }
      }
    }
  }
}
```

## 环境变量

### 直接配置

在配置中直接设置环境变量：

```json
{
  "tools": {
    "mcp": {
      "servers": {
        "my-server": {
          "command": "python",
          "args": ["-m", "my_mcp_server"],
          "env": {
            "API_KEY": "secret-key",
            "DEBUG": "true"
          }
        }
      }
    }
  }
}
```

### 从文件加载

使用 `.env` 文件管理敏感信息：

**配置**:
```json
{
  "tools": {
    "mcp": {
      "servers": {
        "my-server": {
          "command": "python",
          "args": ["-m", "my_mcp_server"],
          "env_file": ".env.mcp"
        }
      }
    }
  }
}
```

**`.env.mcp` 文件内容**:
```bash
# 这是注释
API_KEY=your-secret-key
DATABASE_URL=postgres://localhost/mydb
DEBUG=true
```

**优先级**: `env` 配置 > `env_file` 文件 > 系统环境变量

## 自动检测

如果未指定 `type`，系统会自动检测：

1. 如果设置了 `url`，使用 `sse` 传输
2. 如果设置了 `command`，使用 `stdio` 传输
3. 否则返回错误

## 完整配置示例

```json
{
  "tools": {
    "mcp": {
      "enabled": true,
      "servers": {
        "filesystem": {
          "enabled": true,
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]
        },
        "github": {
          "enabled": true,
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-github"],
          "env_file": ".env.github"
        },
        "postgres": {
          "enabled": true,
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-postgres"],
          "env": {
            "POSTGRES_CONNECTION_STRING": "postgresql://user:pass@localhost/db"
          }
        },
        "remote-tool": {
          "enabled": true,
          "type": "sse",
          "url": "https://mcp.example.com/sse",
          "headers": {
            "Authorization": "Bearer token123"
          }
        },
        "disabled-server": {
          "enabled": false,
          "command": "npx",
          "args": ["-y", "some-mcp-server"]
        }
      }
    }
  }
}
```

## 常用 MCP 服务器

| 服务器 | npm 包 | 说明 |
|--------|--------|------|
| Filesystem | `@modelcontextprotocol/server-filesystem` | 文件系统操作 |
| GitHub | `@modelcontextprotocol/server-github` | GitHub API |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | PostgreSQL 数据库 |
| SQLite | `@modelcontextprotocol/server-sqlite` | SQLite 数据库 |
| Puppeteer | `@modelcontextprotocol/server-puppeteer` | 浏览器自动化 |
| Slack | `@modelcontextprotocol/server-slack` | Slack 集成 |

## 故障排除

### 服务器连接失败

1. 检查 `command` 路径是否正确
2. 确认 `npx` 或其他运行时已安装
3. 检查环境变量是否正确设置
4. 查看日志输出了解详细错误信息

### 工具未显示

1. 确认服务器 `enabled: true`
2. 检查服务器是否支持 tools 能力
3. 查看连接日志确认工具列表已获取

### 环境变量未生效

1. 检查 `env_file` 路径是否正确（相对于 workspace）
2. 确认 `.env` 文件格式正确
3. 注意优先级：配置中的 `env` 会覆盖 `env_file`
