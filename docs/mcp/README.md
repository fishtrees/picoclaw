# MCP (Model Context Protocol) 模块

PicoClaw 提供了完整的 MCP (Model Context Protocol) 支持，允许与各种外部 MCP 服务器集成，扩展工具系统能力。

## 概述

MCP 是一个开放协议，允许 AI 助手与外部工具和数据源进行交互。PicoClaw 的 MCP 模块实现了：

- **多服务器管理**：同时连接和管理多个 MCP 服务器
- **多种传输协议**：支持 stdio、SSE、HTTP 传输方式
- **工具自动发现**：自动获取服务器提供的工具并注册到工具系统
- **环境变量支持**：支持从配置和 `.env` 文件加载环境变量

## 模块结构

```
pkg/
├── mcp/
│   ├── manager.go       # MCP 管理器核心实现
│   └── manager_test.go  # 管理器测试
├── tools/
│   ├── mcp_tool.go      # MCP 工具包装器
│   └── mcp_tool_test.go # 工具包装器测试
└── config/
    └── config.go        # MCP 配置定义
```

## 核心组件

### 1. Manager (管理器)

MCP 管理器负责管理所有 MCP 服务器连接。

**位置**: [pkg/mcp/manager.go](../../pkg/mcp/manager.go)

**主要功能**:
- 创建和管理多个服务器连接
- 自动发现服务器提供的工具
- 工具调用和结果处理
- 资源清理

### 2. MCPTool (工具包装器)

将 MCP 服务器提供的工具适配到 PicoClaw 的工具系统。

**位置**: [pkg/tools/mcp_tool.go](../../pkg/tools/mcp_tool.go)

**主要功能**:
- 实现 Tool 接口
- 工具名称规范化（64 字符限制）
- 参数 schema 转换
- 执行工具并处理结果

## 快速开始

### 配置示例

```json
{
  "tools": {
    "mcp": {
      "enabled": true,
      "servers": {
        "filesystem": {
          "enabled": true,
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
        }
      }
    }
  }
}
```

### 传输方式

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| `stdio` | 通过命令行进程通信 | 本地 MCP 服务器 |
| `sse` | Server-Sent Events | 远程 MCP 服务器 |
| `http` | HTTP 请求 | 远程 MCP 服务器 |

## 文档索引

- [配置指南](./configuration.md) - 详细的配置说明和示例
- [API 参考](./api-reference.md) - Go API 文档

## 相关链接

- [Model Context Protocol 官方文档](https://modelcontextprotocol.io/)
- [MCP Go SDK](https://github.com/modelcontextprotocol/go-sdk)
