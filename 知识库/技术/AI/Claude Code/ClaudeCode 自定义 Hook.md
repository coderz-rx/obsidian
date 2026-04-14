> 核心目标：从零开始创建一个自定义 Hook，实现「禁止 Claude 修改敏感文件，并在拦截时给出中文提示」。

---

## 一、自定义 Hook 的核心流程

自定义一个 Hook 需要走完以下 5 步：

```
确定事件类型 → 编写脚本（处理 stdin/stdout） → 赋予执行权限
        ↓
配置 settings.json → 测试验证
```

| 步骤 | 关键动作 |
|------|----------|
| **1. 定事件** | 想拦截选 `PreToolUse`，想善后选 `PostToolUse`，想通知选 `Notification` |
| **2. 写脚本** | 从 `stdin` 读 JSON（用 `cat` / `jq`），根据逻辑返回 stdout |
| **3. 给权限** | `chmod +x` 让脚本可执行 |
| **4. 配 JSON** | 在 `.claude/settings.json` 的对应事件数组里注册 |
| **5. 测结果** | 用 `/hooks` 检查加载，用实际场景测试，用手动 `echo '{"...":...}' \| hook.sh` 快速验证 |

---

## 二、实战案例：保护敏感文件

### 2.1 需求

当 Claude 尝试修改 `.env`、`package-lock.json`、或 `migrations/` 目录下的文件时，自动拦截并告诉它：这个文件受保护，不能改。

### 2.2 确定事件类型

要「拦截」操作，必须在工具执行**之前**触发，所以事件类型选 **`PreToolUse`**。

### 2.3 编写 Hook 脚本

创建一个可执行脚本，让它：
1. 从 `stdin` 读取 Claude 传入的 JSON
2. 提取要修改的文件路径
3. 判断是否在黑名单里
4. 如果是，输出 `block` 决策；否则正常退出（不阻止）

#### 创建脚本文件

```bash
mkdir -p .claude/hooks
touch .claude/hooks/protect-sensitive-files.sh
```

#### 脚本内容

```bash
#!/bin/bash

# 1. 从 stdin 读取 JSON 输入
INPUT=$(cat)

# 2. 提取工具名和文件路径
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty')
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# 3. 只拦截 Edit 和 Write 操作
if [[ "$TOOL_NAME" != "Edit" && "$TOOL_NAME" != "Write" ]]; then
  exit 0
fi

# 4. 判断文件是否在保护列表中
if echo "$FILE_PATH" | grep -qE '(\.env|package-lock\.json|migrations/.*|\.github/workflows/.*)'; then
  # 5. 返回 block 决策（必须输出严格的 JSON）
  echo '{
    "hookEventName": "PreToolUse",
    "decision": {
      "behavior": "block"
    },
    "permissionDecisionReason": "该文件属于受保护文件（.env / package-lock.json / migrations / CI配置），请勿修改。如有需要，请手动编辑或明确授权。"
  }'
  exit 1
fi

# 6. 非敏感文件，放行
exit 0
```

#### 赋予执行权限

```bash
chmod +x .claude/hooks/protect-sensitive-files.sh
```

### 2.4 配置到 settings.json

在**项目根目录**创建或编辑 `.claude/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "type": "command",
        "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-sensitive-files.sh"
      }
    ]
  }
}
```

**关于 `$CLAUDE_PROJECT_DIR`**：
这是 Claude Code 自动注入的环境变量，指向当前项目的绝对路径。用它拼接脚本路径，可以避免相对路径在不同工作目录下的问题。

### 2.5 测试验证

#### 方法 A：用 `/hooks` 查看是否加载
在 Claude Code 中输入：
```
/hooks
```
如果看到 `PreToolUse` 下面有你的 `protect-sensitive-files.sh`，说明配置已生效。

#### 方法 B：直接触发拦截
让 Claude 尝试修改一个受保护文件：
```
帮我在 .env 里加一行 DEBUG=true
```
如果 Hook 正常工作，Claude 会：
1. 规划使用 `Write` 或 `Edit` 工具修改 `.env`
2. Hook 触发，读取到 `.env` 在黑名单中
3. Hook 返回 `block`
4. Claude 收到拦截，输出类似：
   > "我尝试修改 `.env`，但被项目规则阻止了：该文件属于受保护文件... 请勿修改。"

#### 方法 C：手动模拟 stdin
在终端里直接测试脚本逻辑：
```bash
echo '{"tool_name":"Edit","tool_input":{"file_path":".env"}}' | .claude/hooks/protect-sensitive-files.sh
```
应该输出：
```json
{"hookEventName":"PreToolUse","decision":{"behavior":"block"},"permissionDecisionReason":"该文件属于受保护文件..."}
```

再测一个安全文件：
```bash
echo '{"tool_name":"Edit","tool_input":{"file_path":"src/index.ts"}}' | .claude/hooks/protect-sensitive-files.sh
```
应该**没有任何输出**，退出码为 0（放行）。

---

## 三、进阶：让 Hook 更智能

### 3.1 返回 `ask` 而不是绝对 `block`

如果你不想一刀切，可以改成：匹配敏感文件时，返回 `ask`，让 Claude 弹窗询问你是否允许。

把脚本里的输出块改成：
```bash
echo '{
  "hookEventName": "PreToolUse",
  "decision": {
    "behavior": "ask"
  },
  "permissionDecisionReason": "即将修改敏感文件 '$FILE_PATH'，建议先确认是否必要。"
}'
exit 1
```

### 3.2 把拦截记录到审计日志

在 block 的同时，把事件记到日志文件：
```bash
echo "$FILE_PATH blocked at $(date)" >> .claude/hooks/protection-audit.log
```

### 3.3 动态加载保护规则

不 hardcode 正则，而是把保护规则写在一个 `.claude/hooks/protected-files.txt` 里：
```text
\.env
package-lock\.json
migrations/
```
然后脚本里用 `grep -f` 匹配：
```bash
if echo "$FILE_PATH" | grep -qEf .claude/hooks/protected-files.txt; then
  # block
fi
```
这样不用改脚本就能调整规则。

---

## 四、另一个简单案例：PostToolUse 自动格式化

如果你只想做一个**不拦截、只善后**的 Hook，比如「Claude 改完代码后自动用 Prettier 格式化」：

### 直接写在 settings.json 里，不需要单独脚本

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "type": "command",
        "command": "jq -r '.tool_input.file_path // empty' | xargs -r npx prettier --write"
      }
    ]
  }
}
```

### 加上 `if` 过滤，只对 Edit/Write 生效

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "type": "command",
        "if": "tool_name == 'Edit' or tool_name == 'Write'",
        "command": "jq -r '.tool_input.file_path // empty' | xargs -r npx prettier --write"
      }
    ]
  }
}
```

---

## 五、Hook 脚本编写要点

### 5.1 输入（stdin）

当 Hook 触发时，Claude Code 会通过 stdin 传入一个 JSON 对象。以 `PreToolUse` 为例：

```json
{
  "hook_event_name": "PreToolUse",
  "session_id": "uuid",
  "cwd": "/path/to/project",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "src/index.ts",
    "old_string": "...",
    "new_string": "..."
  }
}
```

### 5.2 输出（stdout）

**普通 hooks**（如 `PostToolUse`、`SessionStart`）：
- 退出码 0 表示成功
- stdout 内容可能注入 Claude 上下文（视事件类型而定）

**决策类 hooks**（`PreToolUse`、`PermissionRequest`）：

必须输出严格合法的 JSON：

```json
{
  "hookEventName": "PreToolUse",
  "decision": {
    "behavior": "block"
  },
  "permissionDecisionReason": "..."
}
```

| 行为 | 含义 |
|------|------|
| `allow` | 放行，继续执行 |
| `block` | 拦截，终止该工具调用 |
| `ask` | 弹窗让用户确认 |

### 5.3 常见调试技巧

| 问题 | 排查方法 |
|------|----------|
| Hook 没触发 | `/hooks` 查看是否加载，检查 JSON 语法、事件名拼写 |
| 脚本不执行 | 检查是否 `chmod +x`，路径是否正确 |
| JSON 解析失败 | 确保 stdout 没有额外日志、shell 提示符、或多余换行 |
| 决策不生效 | 检查 `hookEventName` 是否和事件名一致，`decision` 结构是否正确 |

---

## 六、更多自定义 Hook 灵感

| 场景 | 事件 | 实现思路 |
|------|------|----------|
| 自动批准白名单命令 | `PermissionRequest` | `if` 过滤 `git status` / `npm run lint`，返回 `allow` |
| 会话开始时注入 Git 信息 | `SessionStart` | `git log --oneline -3` 输出到 stdout |
| Bash 命令审计 | `PostToolUse` | `if` 过滤 `tool_name == 'Bash'`，把 `tool_input.command` 追加到日志 |
| 配合 direnv 自动加载环境 | `CwdChanged` | `direnv export bash >> "$CLAUDE_ENV_FILE"` |
| 配置文件变更审计 | `ConfigChange` | `jq` 提取 `source` 和 `file_path` 记录到审计日志 |

---

*记录时间：2026-04-14*
