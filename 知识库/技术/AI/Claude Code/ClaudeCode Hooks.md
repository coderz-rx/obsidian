# Claude Code Hooks 使用指南

> 来源：https://code.claude.com/docs/zh-CN/hooks-guide

Hooks 是 Claude Code 的自动化工作流机制。当 Claude 编辑文件、完成任务、需要用户输入或执行特定操作时，会自动触发你预定义的 shell 命令。与 CLAUDE.md 中的软指令不同，Hooks 是**确定性的约束和验证**，不会被 LLM 忽略。

---

## 一、核心定位：Hook vs CLAUDE.md

| 特性 | CLAUDE.md | Hooks |
|------|-----------|-------|
| 性质 | 指导性建议 | 确定性规则 |
| 执行方式 | LLM 自主决定是否遵守 | 事件触发，自动执行 |
| 用途 | 项目规范、编码风格 | 格式化、验证、拦截、通知 |
| 可靠性 | 可能被上下文稀释 | 100% 触发 |

---

## 二、配置位置

Hooks 配置在 `settings.json` 中，支持三层作用域：

| 路径 | 作用域 |
|------|--------|
| `~/.claude/settings.json` | 全局，所有 Claude 会话生效 |
| `./.claude/settings.json` | 项目级，仅当前项目生效 |
| 任意子目录 `.claude/settings.json` | 子目录级（monorepo 场景） |

> 用 `/hooks` 命令可快速查看当前生效的所有 hooks。

---

## 三、基本配置格式

```json
{
  "hooks": {
    "Notification": [
      {
        "type": "command",
        "command": "osascript -e 'display notification \"Claude Code needs your attention\" with title \"Claude Code\"'"
      }
    ],
    "PostToolUse": [
      {
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
      }
    ]
  }
}
```

**每个 hook 对象的通用字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | 目前主要是 `"command"` |
| `command` | string | 要执行的 shell 命令 |
| `matcher` | string | （可选）正则表达式，过滤触发条件 |
| `if` | string | （可选）按工具名/参数过滤的条件表达式 |

---

## 四、完整事件类型列表

| 事件名 | 触发时机 | 特殊能力 |
|--------|----------|----------|
| `SessionStart` | 会话开始或恢复时 | stdout 输出注入 Claude 上下文 |
| `UserPromptSubmit` | 用户提交提示前 | stdout 输出注入 Claude 上下文 |
| `PreToolUse` | **工具执行前** | 可 `block` / `allow` / `ask` |
| `PermissionRequest` | 出现权限请求对话框时 | 可自动 `allow` |
| `PermissionDenied` | 权限被 auto mode 拒绝时 | 可返回 `{retry: true}` 让模型重试 |
| `PostToolUse` | 工具执行成功后 | 常用于格式化、验证 |
| `PostToolUseFailure` | 工具执行失败后 | 错误恢复、日志记录 |
| `Notification` | Claude 发送通知时 | 桌面通知、消息推送 |
| `SubagentStart` | 子代理启动时 | 监控子代理行为 |
| `SubagentStop` | 子代理结束时 | 汇总子代理结果 |
| `TaskCreated` | 通过 TaskCreate 创建任务时 | 任务追踪 |
| `TaskCompleted` | 任务标记为完成时 | 后续自动化 |
| `Stop` | Claude 完成本轮响应时 | 总结、清理 |
| `StopFailure` | 因 API 错误结束轮次时 | 错误上报（输出和退出码被忽略） |
| `TeammateIdle` | Agent team 队友即将空闲时 | 团队协调 |
| `InstructionsLoaded` | CLAUDE.md / rules 文件加载时 | 动态规则注入 |
| `ConfigChange` | 配置文件在会话中变更时 | 审计、日志 |
| `CwdChanged` | 工作目录变更时（如执行 `cd`） | 配合 direnv 重新加载环境 |
| `FileChanged` | **被监视的文件**在磁盘上变更时 | 需配合 `matcher` 使用 |
| `WorktreeCreate` | 创建 worktree 时 | 替换默认 git 行为 |
| `WorktreeRemove` | 移除 worktree 时 | 清理逻辑 |
| `PreCompact` | 上下文压缩前 | 保留关键信息 |
| `PostCompact` | 上下文压缩后 | 恢复被压缩的状态 |
| `Elicitation` | MCP server 请求用户输入时 | 拦截或辅助处理 |
| `ElicitationResult` | 用户响应 MCP 请求后 | 结果预处理 |
| `SessionEnd` | 会话终止时 | 最终清理 |

---

## 五、输入输出机制

Hooks 是一个**通过 stdin/stdout 与 Claude Code 交互**的脚本/命令。

### 5.1 输入（stdin）

当 hook 触发时，Claude Code 会通过 stdin 传入一个 JSON 对象。通用字段包括：

```json
{
  "hook_event_name": "PreToolUse",
  "session_id": "uuid",
  "cwd": "/path/to/project",
  "source": "user",
  "file_path": "relative/path/to/file",
  "tool_name": "Edit",
  "tool_input": { "file_path": "src/index.ts", "old_string": "...", "new_string": "..." }
}
```

**不同事件的特有字段：**
- `UserPromptSubmit`：`prompt`（用户输入的原始文本）
- `SessionStart`：`source`（`startup` / `resume` / `clear` / `compact`）
- `PreToolUse` / `PostToolUse`：`tool_name`, `tool_input`
- `PermissionRequest`：`permission_type`, `permission_details`
- `ConfigChange`：`source`（`user_settings` / `project_settings` / `local_settings` / `policy_settings` / `skills`）
- `Notification`：`source`（`permission_prompt` / `idle_prompt` / `auth_success` / `elicitation_dialog`）

### 5.2 输出（stdout）

**普通 hooks**（`SessionStart`, `UserPromptSubmit`, `PostToolUse` 等）：
- 退出码 0：成功
- stdout 内容会直接注入 Claude 的上下文（对 `SessionStart` 和 `UserPromptSubmit` 特别有效）

**决策类 hooks**（`PreToolUse`, `PermissionRequest`）：

`PreToolUse` 返回示例：
```json
{
  "hookEventName": "PreToolUse",
  "decision": {
    "behavior": "block"
  },
  "permissionDecisionReason": "This file is protected and should not be modified by AI."
}
```

- `behavior`: `"allow"` | `"block"` | `"ask"`
- `permissionDecisionReason`: 解释原因，Claude 会把这个原因展示给用户或用于后续决策

`PermissionRequest` 返回示例：
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow"
    }
  }
}
```

`PermissionDenied` 返回示例：
```json
{ "retry": true }
```

> **注意**：`PostToolUse` 和 `Stop` hooks **不支持**返回 `{"decision": "block"}`。只有 `PreToolUse` 和 `PermissionRequest` 能做拦截决策。

---

## 六、过滤机制：让 Hook 只针对特定场景触发

### 6.1 `matcher` 正则过滤

用于 `FileChanged` 等事件，指定要监视的文件名模式：

```json
{
  "hooks": {
    "FileChanged": [
      {
        "type": "command",
        "matcher": "\\.envrc|\\.env",
        "command": "direnv export bash \u003e\u003e \"$CLAUDE_ENV_FILE\""
      }
    ]
  }
}
```

### 6.2 `if` 条件过滤

用于 `PreToolUse`, `PostToolUse` 等工具事件，按工具名称和参数过滤：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "type": "command",
        "if": "tool_name == 'Bash'",
        "command": "jq -r '.tool_input.command' \u003e\u003e ~/claude-bash-audit.log"
      }
    ]
  }
}
```

---

## 七、常用实战示例

### 示例 1：Claude 需要输入时发送桌面通知

**macOS：**
```json
{
  "hooks": {
    "Notification": [
      {
        "type": "command",
        "command": "osascript -e 'display notification \"Claude Code needs your attention\" with title \"Claude Code\"'"
      }
    ]
  }
}
```

**Linux：**
```json
{
  "hooks": {
    "Notification": [
      {
        "type": "command",
        "command": "notify-send 'Claude Code' 'Claude Code needs your attention'"
      }
    ]
  }
}
```

**Windows (PowerShell)：**
```json
{
  "hooks": {
    "Notification": [
      {
        "type": "command",
        "command": "powershell.exe -Command \"[System.Reflection.Assembly]::LoadWithPartialName('System.Windows.Forms'); [System.Windows.Forms.MessageBox]::Show('Claude Code needs your attention', 'Claude Code')\""
      }
    ]
  }
}
```

### 示例 2：文件编辑后自动格式化（Prettier）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
      }
    ]
  }
}
```

### 示例 3：阻止修改受保护文件

**步骤 1**：创建脚本 `.claude/hooks/protect-files.sh`

```bash
#!/bin/bash
# 读取 stdin 中的 JSON，提取文件路径
FILE=$(jq -r '.tool_input.file_path')

# 检查是否是受保护文件
if echo "$FILE" | grep -qE "(migrations/|\.env|package-lock\.json)"; then
  echo '{"hookEventName": "PreToolUse", "decision": {"behavior": "block"}, "permissionDecisionReason": "This file is protected."}'
  exit 1
fi
```

**步骤 2**：赋予执行权限
```bash
chmod +x .claude/hooks/protect-files.sh
```

**步骤 3**：配置 hook
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "type": "command",
        "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh"
      }
    ]
  }
}
```

### 示例 4：会话开始时注入团队上下文

```json
{
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": "echo 'Reminder: use Bun, not npm. Run bun test before committing. Current sprint: auth refactor.'"
      }
    ]
  }
}
```

也可以注入动态内容：
```json
{
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": "git log --oneline -5"
      }
    ]
  }
}
```

### 示例 5：审计配置变更

```json
{
  "hooks": {
    "ConfigChange": [
      {
        "type": "command",
        "command": "jq -c '{timestamp: now | todate, source: .source, file: .file_path}' \u003e\u003e ~/claude-config-audit.log"
      }
    ]
  }
}
```

### 示例 6：自动批准特定权限请求

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "type": "command",
        "command": "echo '{\"hookSpecificOutput\": {\"hookEventName\": \"PermissionRequest\", \"decision\": {\"behavior\": \"allow\"}}}'"
      }
    ]
  }
}
```

> ⚠️ **警告**：全局自动批准所有权限会降低安全性。建议配合 `matcher` 或 `if` 只对安全的命令（如 `npm run lint`、`git status`）自动放行。

### 示例 7：目录切换时自动加载 direnv 环境

```json
{
  "hooks": {
    "CwdChanged": [
      {
        "type": "command",
        "command": "direnv export bash \u003e\u003e \"$CLAUDE_ENV_FILE\""
      }
    ],
    "FileChanged": [
      {
        "type": "command",
        "matcher": "\\.envrc|\\.env",
        "command": "direnv export bash \u003e\u003e \"$CLAUDE_ENV_FILE\""
      }
    ]
  }
}
```

> `$CLAUDE_ENV_FILE` 是一个临时文件，Claude Code 会在执行 Bash 命令前自动 source 它，从而让 direnv 的环境变量生效。

### 示例 8：记录每个 Bash 命令到审计日志

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "type": "command",
        "if": "tool_name == 'Bash'",
        "command": "jq -r '.tool_input.command' \u003e\u003e ~/claude-bash-audit.log"
      }
    ]
  }
}
```

---

## 八、调试与排错

### 8.1 Hook 没触发
- 用 `/hooks` 检查配置是否被正确加载
- 检查 JSON 语法是否有效
- 确认事件名称拼写正确（大小写敏感）
- 检查 `matcher` / `if` 条件是否过于严格

### 8.2 Stop hook 永远运行
- `Stop` 事件在 Claude 每轮响应结束时触发。如果你看到它"一直运行"，可能是因为 Claude 在循环中不断地结束和开始新轮次。检查是否有其他 hook 导致了无限循环。

### 8.3 JSON 验证失败
- 确保 hook 的 stdout 输出是**严格有效的 JSON**（没有多余的日志、换行、或 shell 提示符）
- 决策类 hook 必须包含正确的 `hookEventName` 和 `decision` 结构

### 8.4 调试技巧
- 先把 hook 命令在终端中手动跑一遍，确认它能正常工作
- 在 hook 脚本中加 `echo "..." \u003e\u003e /tmp/hook-debug.log` 来追踪执行过程
- 使用 `jq` 处理 stdin 时，先用 `cat \u003e /tmp/hook-input.json` 把输入保存下来看结构

---

## 九、限制与注意事项

1. **Hooks 和权限模式**：
   - 在 `auto mode` 下，Hooks 仍然生效。
   - `PermissionRequest` hook 可以覆盖 auto mode 的决策。
   - `deny` 决策会被 `PreToolUse` hook 拦截，不会到达 auto mode 分类器。

2. **执行环境**：
   - Hooks 在 Claude Code 的 shell 环境中执行，继承当前工作目录和环境变量。
   - 长时间运行的 hook 会阻塞 Claude 的响应，建议保持轻量。

3. **不要循环触发**：
   - 一个 `PostToolUse` hook 执行了 Bash/Edit，可能再次触发另一个 `PostToolUse`，注意避免无限循环。

4. **权限脚本**：
   - 在 macOS/Linux 上，自定义脚本必须 `chmod +x` 才能被直接执行。

---

## 十、进阶：基于 Prompt 的 Hooks 和基于 Agent 的 Hooks

Claude Code 还支持更高级的 hook 类型：

- **Prompt-based hooks**：让 Claude LLM 本身作为 hook 的处理器，对输入做自然语言判断后再返回决策。
- **Agent-based hooks**：把决策委托给专门的 subagent 去做复杂分析。
- **HTTP hooks**：向远程 HTTP endpoint 发送请求，实现外部系统集成。

这些高级类型需要更复杂的配置，具体可参考官方文档的进阶章节。

---

*记录时间：2026-04-14*
