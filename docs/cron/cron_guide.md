# Cron Task Scheduling Guide

PicoClaw's cron feature provides powerful task scheduling capabilities, supporting various scheduling modes and task types.

## Overview

The cron system allows you to schedule tasks that can:
- Send simple notification messages
- Execute complex tasks processed by the Agent
- Run system commands

## Core Concepts

### Data Structures

```go
// Schedule configuration
type CronSchedule struct {
    Kind    string  // "at", "every", "cron"
    AtMS    *int64  // Specific timestamp (for "at")
    EveryMS *int64  // Interval in milliseconds (for "every")
    Expr    string  // Cron expression (for "cron")
    TZ      string  // Timezone
}

// Task payload
type CronPayload struct {
    Kind    string  // "agent_turn"
    Message string  // Message content
    Command string  // Command to execute (optional)
    Deliver bool    // Direct delivery to channel or Agent processing
    Channel string  // Target channel
    To      string  // Target user/chat ID
}

// Task state
type CronJobState struct {
    NextRunAtMS *int64 // Next run timestamp
    LastRunAtMS *int64 // Last run timestamp
    LastStatus  string // Last status
    LastError   string // Last error message
}

// Complete cron job
type CronJob struct {
    ID             string       // Unique job ID
    Name           string       // Job name
    Enabled        bool         // Whether enabled
    Schedule       CronSchedule // Schedule configuration
    Payload        CronPayload  // Task payload
    State          CronJobState // Job state
    CreatedAtMS    int64        // Creation timestamp
    UpdatedAtMS    int64        // Update timestamp
    DeleteAfterRun bool         // Delete after execution
}
```

## Schedule Types

| Type | Description | Use Case |
|------|-------------|----------|
| `at` | One-time task at specific time | Reminders, delayed execution |
| `every` | Periodic task with fixed interval | Health checks, periodic reports |
| `cron` | Standard cron expression | Business hour tasks, weekly reports |

### Schedule Examples

```
# One-time task
"schedule": {"kind": "at", "at_ms": 1709875200000}

# Every 5 minutes
"schedule": {"kind": "every", "every_ms": 300000}

# Every day at 9:00 AM
"schedule": {"kind": "cron", "expr": "0 9 * * *"}

# Every weekday at 6:00 PM
"schedule": {"kind": "cron", "expr": "0 18 * * 1-5"}
```

## Task Types

### 1. Simple Message (Direct Delivery)

When `deliver=true`, the message is sent directly to the specified channel.

```json
{
  "action": "add",
  "name": "Daily Reminder",
  "message": "Time to stand up and stretch!",
  "cron_expr": "0 10 * * *",
  "deliver": true,
  "channel": "general"
}
```

### 2. Complex Task (Agent Processing)

When `deliver=false`, the message is processed by the Agent with full tool capabilities.

```json
{
  "action": "add",
  "name": "Daily Report",
  "message": "Generate today's work summary report",
  "cron_expr": "0 17 * * 1-5",
  "deliver": false,
  "channel": "reports"
}
```

The Agent can:
- Execute shell commands
- Read/write files
- Search the web
- Call APIs
- Perform complex reasoning
- Complete multi-step workflows

### 3. System Command Execution

Execute shell commands and return output:

```json
{
  "action": "add",
  "name": "System Health Check",
  "command": "df -h && free -m",
  "cron_expr": "0 */6 * * *",
  "deliver": false,
  "channel": "ops"
}
```

## Configuration

| Config | Type | Default | Description |
|--------|------|---------|-------------|
| `exec_timeout_minutes` | int | 5 | Command execution timeout in minutes, 0 for no limit |

### Configuration Example

```json
{
  "tools": {
    "cron": {
      "exec_timeout_minutes": 10
    }
  }
}
```

## Execution Flow

1. **Task Check** (every second)
   - Iterate through all enabled tasks
   - Check if any task has reached execution time

2. **Task Execution** (async)
   - Copy task to goroutine for execution
   - Avoid blocking the main scheduler loop

3. **State Update**
   - Record execution time and status
   - Calculate next execution time
   - Persist to storage

4. **Result Handling**
   - Message type: Send via MessageBus
   - Command type: Execute via ExecTool and return results

## Persistence

- Tasks are stored in `{workspace}/cron/jobs.json`
- Uses atomic write for data safety
- File permissions set to 600 (read/write only)

## Task Management

| Action | Description |
|--------|-------------|
| `add` | Create a new cron job |
| `list` | List all jobs and their status |
| `enable` | Enable a disabled job |
| `disable` | Disable an enabled job |
| `delete` | Remove a job |

## CLI Usage

```bash
# Add periodic task
picoclaw cron add --name "System Check" --message "Check system status" --every 3600 --deliver true

# Add cron expression task
picoclaw cron add --name "Daily Report" --message "Generate daily report" --cron "0 9 * * *" --deliver false

# Add one-time task
picoclaw cron add --name "Meeting Reminder" --message "Meeting in 1 hour" --at "2024-03-10T14:00:00" --deliver true

# List all tasks
picoclaw cron list

# Disable a task
picoclaw cron disable --job_id "task-id"

# Enable a task
picoclaw cron enable --job_id "task-id"

# Delete a task
picoclaw cron delete --job_id "task-id"
```

## Security Features

- Command execution timeout control
- Workspace path restrictions
- Support for path whitelist/blacklist
- Dangerous command blocking (inherited from ExecTool)

## Best Practices

1. **Use descriptive names**: Make it easy to identify task purposes
2. **Set appropriate timeouts**: Avoid long-running commands blocking resources
3. **Test before scheduling**: Verify task behavior before deploying
4. **Use `deliver=false` for complex tasks**: Leverage Agent's full capabilities
5. **Set `delete_after_run` for one-time tasks**: Automatic cleanup

## Troubleshooting

### Task not executing

1. Check if the task is enabled
2. Verify the cron expression is correct
3. Check timezone settings
4. Review logs for error messages

### Command timeout

1. Increase `exec_timeout_minutes` in configuration
2. Optimize command for faster execution
3. Consider splitting into multiple tasks

### Task state not updating

1. Check file permissions on `jobs.json`
2. Verify disk space availability
3. Check for concurrent access issues
