# Cron 定时任务指南

PicoClaw 的 cron 功能提供强大的任务调度能力，支持多种调度模式和任务类型。

## 概述

cron 系统允许您调度执行以下类型的任务：
- 发送简单的通知消息
- 执行由 Agent 处理的复杂任务
- 运行系统命令

## 核心概念

### 数据结构

```go
// 调度配置
type CronSchedule struct {
    Kind    string  // "at", "every", "cron"
    AtMS    *int64  // 特定时间戳（用于 "at"）
    EveryMS *int64  // 间隔毫秒数（用于 "every"）
    Expr    string  // Cron 表达式（用于 "cron"）
    TZ      string  // 时区
}

// 任务负载
type CronPayload struct {
    Kind    string  // "agent_turn"
    Message string  // 消息内容
    Command string  // 要执行的命令（可选）
    Deliver bool    // 直接投递到频道还是由 Agent 处理
    Channel string  // 目标频道
    To      string  // 目标用户/聊天 ID
}

// 任务状态
type CronJobState struct {
    NextRunAtMS *int64 // 下次运行时间戳
    LastRunAtMS *int64 // 上次运行时间戳
    LastStatus  string // 最后状态
    LastError   string // 最后错误信息
}

// 完整的 Cron 任务
type CronJob struct {
    ID             string       // 任务唯一 ID
    Name           string       // 任务名称
    Enabled        bool         // 是否启用
    Schedule       CronSchedule // 调度配置
    Payload        CronPayload  // 任务负载
    State          CronJobState // 任务状态
    CreatedAtMS    int64        // 创建时间戳
    UpdatedAtMS    int64        // 更新时间戳
    DeleteAfterRun bool         // 执行后删除
}
```

## 调度类型

| 类型 | 描述 | 使用场景 |
|------|------|----------|
| `at` | 一次性任务，在特定时间执行 | 提醒、延迟执行 |
| `every` | 周期性任务，固定间隔执行 | 健康检查、定期报告 |
| `cron` | 标准 cron 表达式 | 工作时间任务、周报 |

### 调度示例

```
# 一次性任务
"schedule": {"kind": "at", "at_ms": 1709875200000}

# 每 5 分钟
"schedule": {"kind": "every", "every_ms": 300000}

# 每天上午 9:00
"schedule": {"kind": "cron", "expr": "0 9 * * *"}

# 每个工作日下午 6:00
"schedule": {"kind": "cron", "expr": "0 18 * * 1-5"}
```

## 任务类型

### 1. 简单消息（直接投递）

当 `deliver=true` 时，消息直接发送到指定频道。

```json
{
  "action": "add",
  "name": "每日提醒",
  "message": "该站起来活动一下了！",
  "cron_expr": "0 10 * * *",
  "deliver": true,
  "channel": "general"
}
```

### 2. 复杂任务（Agent 处理）

当 `deliver=false` 时，消息由 Agent 处理，可使用所有工具能力。

```json
{
  "action": "add",
  "name": "每日报告",
  "message": "生成今日工作总结报告",
  "cron_expr": "0 17 * * 1-5",
  "deliver": false,
  "channel": "reports"
}
```

Agent 可以：
- 执行 shell 命令
- 读写文件
- 搜索网络
- 调用 API
- 进行复杂推理
- 完成多步骤工作流

### 3. 系统命令执行

执行 shell 命令并返回输出：

```json
{
  "action": "add",
  "name": "系统健康检查",
  "command": "df -h && free -m",
  "cron_expr": "0 */6 * * *",
  "deliver": false,
  "channel": "ops"
}
```

## 配置项

| 配置项 | 类型 | 默认值 | 描述 |
|--------|------|--------|------|
| `exec_timeout_minutes` | int | 5 | 命令执行超时时间（分钟），0 表示不限制 |

### 配置示例

```json
{
  "tools": {
    "cron": {
      "exec_timeout_minutes": 10
    }
  }
}
```

## 执行流程

1. **任务检查**（每秒执行）
   - 遍历所有启用的任务
   - 检查是否有任务到达执行时间

2. **任务执行**（异步）
   - 复制任务到 goroutine 中执行
   - 避免阻塞主调度循环

3. **状态更新**
   - 记录执行时间和状态
   - 计算下次执行时间
   - 持久化到存储

4. **结果处理**
   - 消息类型：通过 MessageBus 发送
   - 命令类型：通过 ExecTool 执行并返回结果

## 持久化

- 任务存储在 `{workspace}/cron/jobs.json`
- 使用原子写入确保数据安全
- 文件权限设置为 600（仅可读写）

## 任务管理

| 操作 | 描述 |
|------|------|
| `add` | 创建新的定时任务 |
| `list` | 列出所有任务及其状态 |
| `enable` | 启用已禁用的任务 |
| `disable` | 禁用已启用的任务 |
| `delete` | 删除任务 |

## 命令行使用

```bash
# 添加周期性任务
picoclaw cron add --name "系统检查" --message "检查系统状态" --every 3600 --deliver true

# 添加 cron 表达式任务
picoclaw cron add --name "每日报告" --message "生成每日报告" --cron "0 9 * * *" --deliver false

# 添加一次性任务
picoclaw cron add --name "会议提醒" --message "1小时后有会议" --at "2024-03-10T14:00:00" --deliver true

# 列出所有任务
picoclaw cron list

# 禁用任务
picoclaw cron disable --job_id "task-id"

# 启用任务
picoclaw cron enable --job_id "task-id"

# 删除任务
picoclaw cron delete --job_id "task-id"
```

## 安全特性

- 命令执行超时控制
- 工作区路径限制
- 支持路径白名单/黑名单
- 危险命令拦截（继承自 ExecTool）

## 最佳实践

1. **使用描述性名称**：便于识别任务用途
2. **设置合理的超时时间**：避免长时间运行的命令占用资源
3. **调度前先测试**：验证任务行为后再部署
4. **复杂任务使用 `deliver=false`**：充分利用 Agent 的能力
5. **一次性任务设置 `delete_after_run`**：自动清理

## 故障排除

### 任务不执行

1. 检查任务是否已启用
2. 验证 cron 表达式是否正确
3. 检查时区设置
4. 查看日志中的错误信息

### 命令超时

1. 在配置中增加 `exec_timeout_minutes`
2. 优化命令以加快执行速度
3. 考虑拆分为多个任务

### 任务状态不更新

1. 检查 `jobs.json` 的文件权限
2. 验证磁盘空间是否充足
3. 检查是否存在并发访问问题
