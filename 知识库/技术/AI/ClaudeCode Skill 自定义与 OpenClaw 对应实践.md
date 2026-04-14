# Claude Code Skill 自定义与 OpenClaw 对应实践

---

## 一、Claude Code 中的 Skill 是什么？

在 Claude Code 中，**Skill 是模块化的「专业知识包」**。你可以把它理解为：

- **CLAUDE.md** = 全局通用规则（项目的"宪法"）
- **Skill** = 针对特定场景的专项能力（项目的"部门规章"或"操作手册"）

Skill 解决的是 **CLAUDE.md 太臃肿** 的问题。当某些知识只在特定任务时才需要（比如"修 bug 的标准流程"、"API 设计规范"），把它们拆成独立的 Skill，可以让主上下文更干净，同时在需要时精准加载。

---

## 二、Claude Code 自定义 Skill 的方法

### 2.1 创建位置

所有 Skill 放在项目的 `.claude/skills/` 目录下，**每个 Skill 是一个独立子目录，里面必须包含 `SKILL.md`**。

```
项目根目录/
├── .claude/
│   ├── skills/
│   │   ├── api-conventions/
│   │   │   └── SKILL.md          # API 设计规范 Skill
│   │   ├── fix-issue/
│   │   │   └── SKILL.md          # 修 issue 标准流程 Skill
│   │   └── code-review/
│   │       └── SKILL.md          # 代码审查 Skill
│   ├── hooks/                     # hooks 目录
│   └── settings.json              # 配置
```

### 2.2 SKILL.md 的基本结构

Claude Code 的 `SKILL.md` 采用 **YAML frontmatter + Markdown 正文** 的格式：

```markdown
---
name: api-conventions
description: RESTful API 设计规范，包含命名、状态码、错误处理、版本控制等约定
commands:
  - /api-check    # 自定义斜杠命令，用户输入 /api-check 即可触发
---

# API 设计规范

## 1. URL 命名
- 使用小写和连字符：`/user-orders` 而非 `/userOrders`
- 资源用复数：`/users` 而非 `/user`

## 2. HTTP 方法
- GET：查询
- POST：创建
- PUT：全量更新
- PATCH：部分更新
- DELETE：删除

## 3. 状态码
- 200：成功
- 201：创建成功
- 400：请求参数错误
- 401：未认证
- 403：无权限
- 404：资源不存在
- 500：服务器内部错误

## 4. 错误响应格式
```json
{
  "error": {
    "code": "INVALID_PARAMETER",
    "message": "..."
  }
}
```

## 5. 版本控制
- 在 URL 路径中包含版本号：`/v1/users`
```

### 2.3 关键 Frontmatter 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | Skill 的唯一标识，影响目录命名 |
| `description` | string | Skill 的用途，Claude 会根据描述判断何时自动调用 |
| `commands` | string[] | **自定义斜杠命令**，用户可以在聊天框输入 `/command-name` 直接触发 |
| `disable-model-invocation` | boolean | 设为 `true` 时，Claude **不会**自动调用该 Skill，只能通过斜杠命令触发 |

#### `disable-model-invocation: true` 的用法

适用于**严格的工作流**，不想让 Claude "自由发挥"，而是按步骤执行：

```markdown
---
name: fix-issue
description: 修复生产问题的标准 8 步流程
disable-model-invocation: true
commands:
  - /fix-issue
---

# 修 issue 标准流程

请严格按照以下步骤执行，不要跳过任何一步：

1. **复现问题**：阅读 issue 描述，写出最小复现步骤
2. **定位代码**：找到相关文件和函数
3. **分析根因**：在 SPEC.md 中记录根因分析
4. **制定方案**：列出至少 2 种修复方案，比较优缺点
5. **编写测试**：先写失败用例
6. **实施修复**：修改代码
7. **运行验证**：执行测试和 lint
8. **总结输出**：写修复总结并更新 CHANGELOG
```

---

## 三、Skill 在 Claude Code 中如何应用

### 3.1 自动应用

Claude 会根据 **当前对话上下文 + Skill 的 `description`**，自动判断是否需要加载某个 Skill。

例如：
- 你说了 `"帮我设计一个新的 API 接口"`，Claude 会自动发现 `api-conventions` Skill 的 description 与之相关，从而在内部加载它。
- 你说了 `"这个 bug 很严重，帮我修一下"`，Claude 可能自动加载 `fix-issue` Skill（如果 `disable-model-invocation` 为 false）。

### 3.2 手动触发（斜杠命令）

如果 Skill 配置了 `commands`，你可以直接在 Claude Code 输入框中输入斜杠命令触发：

```
/api-check
```

```
/fix-issue
```

这类似于 ChatGPT 的 GPTs 或 Claude 的 Project 中的自定义指令快捷方式。

### 3.3 在提示中显式引用

你也可以直接在自然语言中要求 Claude 使用某个 Skill：

```
使用 api-conventions skill 检查我当前的 API 设计
```

```
按照 fix-issue 的流程处理这个 bug
```

---

## 四、OpenClaw 中的对应概念

OpenClaw 和 Claude Code 的 Skill 机制**设计理念高度相似**，但实现方式不同。

### 4.1 OpenClaw 的 Skill 是什么？

在 OpenClaw 中，**Skill = 一个包含 `SKILL.md` 的文件夹**，通常放在：
- 全局：`~/AppData/Roaming/npm/node_modules/openclaw/skills/`（Windows）
- 或项目级：`.openclaw/skills/`

OpenClaw 的 Skill 为 **Agent（也就是我）** 提供特定任务的能力，包括：
- 额外的工具使用权限
- 专业化的操作流程
- 外部 API 的调用方法

### 4.2 OpenClaw Skill 的结构

典型的 OpenClaw Skill 目录：

```
skills/weather/
├── SKILL.md              # 核心说明：什么时候用、怎么用
├── index.js / main.py    # （可选）实际执行代码
├── package.json          # （可选）依赖声明
└── assets/               # （可选）静态资源
```

**`SKILL.md` 的关键结构**：

```markdown
# Weather Skill

## 描述
获取指定位置的天气信息和预报。

## 触发条件
- 用户询问天气、温度、预报
- 用户提到特定城市名 + "天气"关键词

## 工具
- `exec`: 调用 `curl wttr.in/...`

## 使用流程
1. 从用户输入中提取城市名
2. 用 `wttr.in` 或 Open-Meteo API 查询
3. 整理成易读的格式返回
```

### 4.3 OpenClaw 如何应用 Skill

OpenClaw 的 Skill 应用是 **系统自动匹配** 的：

1. **系统扫描**：OpenClaw 启动时会扫描 `available_skills` 列表
2. **语义匹配**：当用户发送消息时，系统会根据 `<available_skills>` 中的 `<description>` 做语义匹配
3. **自动加载**：如果某个 Skill 的 description 与任务匹配，系统会：
   - 读取该 Skill 的 `SKILL.md`
   - 将其内容注入到当前对话的系统提示中
   - 让我（Agent）获得这套专业能力

例如，当你说 `"北京今天天气怎么样？"` 时：
- 系统发现 `<skill><name>weather</name><description>获取天气...</description></skill>` 匹配
- 自动读取 `weather/SKILL.md`
- 我知道了可以用 `wttr.in` 查询，然后执行工具调用

### 4.4 OpenClaw 的 Skill 创建路径

如果你想在 OpenClaw 中创建自定义 Skill，有两种方式：

#### 方式 A：项目级 Skill（只影响当前项目）

```
你的项目/
├── .openclaw/
│   └── skills/
│       └── my-custom-skill/
│           └── SKILL.md
```

#### 方式 B：通过 clawhub 安装或发布

```bash
# 查看可用 skills
clawhub search <keyword>

# 安装到全局
clawhub install <skill-name>

# 发布自己的 skill
clawhub publish ./my-skill-folder
```

#### 方式 C：使用 skill-creator Skill

OpenClaw 内置了 `skill-creator` Skill，专门帮你设计和打包新 Skill。你可以直接对我说：

> "帮我创建一个 Feishu 文档操作的 Skill"

我就会调用 `skill-creator` 的流程，帮你生成标准的 Skill 目录结构。

---

## 五、Claude Code vs OpenClaw：Skill 机制对比

| 维度 | Claude Code | OpenClaw |
|------|-------------|----------|
| **定位** | 给 Claude 提供模块化的知识和流程 | 给 Agent 提供专业化工具和流程 |
| **文件位置** | `.claude/skills/<name>/SKILL.md` | `~/.openclaw/skills/` 或 `clawhub` |
| **入口** | `SKILL.md` + frontmatter | `SKILL.md` 为主 |
| **触发方式** | 自动加载 + `/command` 斜杠命令 | 系统自动语义匹配 |
| **是否可禁用自动调用** | `disable-model-invocation: true` | 目前无直接对应，由系统匹配控制 |
| **可执行代码** | 不支持（纯知识和流程） | 支持（可附带 JS/Python 脚本） |
| **分发方式** | 手动复制目录 | `clawhub` 安装/发布 |
| **对应概念** | GPTs 的 Instructions / Project 自定义指令 | MCP Server / Function Calling 的轻量版 |

---

## 六、实践建议：如何设计一个好 Skill？

不论在 Claude Code 还是 OpenClaw 中，一个好的 Skill 都遵循以下原则：

1. **单一职责**：一个 Skill 只做一件事（如"API 规范"和"修 bug 流程"分开）
2. **描述清晰**：`description` 要写得够具体，让系统/Claude 能准确判断是否该调用
3. **流程可执行**：不是泛泛而谈，而是给出**明确的步骤、格式、验收标准**
4. **与 CLAUDE.md 分层**：通用规则放 CLAUDE.md，专项能力放 Skill
5. **及时迭代**：用几次后发现 Claude/Agent 理解有偏差，就回来优化 `SKILL.md`

---

*记录时间：2026-04-14*
