# Harness Engineering（驾驭工程）

## 一、背景：为什么是「驾驭工程」？

大模型本身能力是「通用智力」，但它有几个致命短板：
- **不可控**：输出不稳定，容易幻觉
- **无状态**：多轮对话后上下文腐化，忘记关键约束
- **难交付**：能对话不代表能完成端到端任务

**提示词工程（Prompt Engineering）**解决了「一次性说清」的问题，**上下文工程（Context Engineering）**解决了「给够信息」的问题，但**没有一个系统性方案解决「持续、稳定、可交付」的问题**。

Harness Engineering 就是 answer to this gap。它不给模型增加参数，而是给模型**套上一个工程化的控制外壳（Harness）**，把大模型从「聊天工具」升级为「可靠的生产力单元」。

2026 年 OpenAI 博客的推广，本质上是把这套实践从 Anthropic/Claude Code 等少数团队的工程秘技，变成了行业公认的标准范式。

---

## 二、三者关系：从「说清」到「给够」再到「持续做对」

| 层级 | 工程形态 | 核心解决的问题 | 类比 |
|------|----------|----------------|------|
| L1 | **提示词工程** | 一次性把需求说清楚，让模型不乱说 | 写一份好 Brief |
| L2 | **上下文工程** | 在窗口限制内，动态给模型喂对的信息 | 做一份好资料包 |
| L3 | **驾驭工程** | 构建全流程管控机制，让模型持续稳定交付 | 搭一套项目管理体系 |

**关键区分：**
- **提示词工程**是「静态设计」——角色、格式、Few-shot 都在输入前定好。
- **上下文工程**是「动态管理」——召回(Retrieval)、压缩(Compression)、组装(Assembly)，解决窗口溢出和腐化问题。
- **Harness Engineering**是「系统编排」——不仅管输入，还管**执行流程、反馈循环、记忆沉淀**。

> **公式：Agent = 大模型 + Harness**

这里的 Harness 不是简单的 wrapper，而是一个包含**编排、执行、反馈、记忆**四层能力的工程系统。

---

## 三、Harness 的核心架构：四层能力

### 1. 编排层（Orchestration）
把复杂任务拆成可管理的子任务，决定：
- 什么由模型做，什么由代码/工具做
- 任务执行的顺序、分支、并发
- 何时调用外部工具（MCP、API、Shell）

**本质**：用大模型做「决策者」，用代码做「执行者」，不让模型干它不擅长的脏活累活。

### 2. 执行层（Execution）
保障每个子任务能落地：
- 工具调用沙箱化（如 Claude Code 的受限 Shell）
- 输出格式校验（JSON Schema、类型检查）
- 失败重试、超时控制、幂等性保障

**本质**：让模型的输出从「文本」变成「可执行指令」，并确保执行安全。

### 3. 反馈层（Feedback）
建立闭环，让系统知道做得对不对：
- **即时反馈**：编译错误、测试失败、Linter 报错 → 自动回传模型修正
- **人类反馈**：用户确认、拒绝、修改 → 更新行为策略
- **自我验证**：让模型先输出计划，再执行，再自检（Plan-Execute-Verify 模式）

**本质**：把「一次性生成」变成「迭代收敛」。

### 4. 记忆层（Memory）
解决模型无状态问题，沉淀长期知识：
- **工作记忆**：当前任务的中间状态、变量
- **长期记忆**：项目规范、用户偏好、历史决策（如 `CLAUDE.md`、项目 Spec）
- **语义记忆**：通过 RAG 检索相关代码片段、文档

**本质**：让模型「越用越懂你和项目」。

---

## 四、落地实践：两个可直接参考的范式

### 范式 A：Claude Code + CLAUDE.md
Anthropic 的 Claude Code 是 Harness Engineering 的标杆产品：
- **Harness 体现**：它不是一个裸聊天框，而是一个有文件系统访问权、Shell 执行权、Git 操作权的受限 Agent，所有操作都在用户确认或可配置自动化的规则下进行。
- **CLAUDE.md**：这是「长期记忆」的核心载体。把项目背景、架构约定、代码规范、常用命令写进去，Claude Code 每次请求前都会读取，相当于给模型植入了项目的「组织记忆」。

### 范式 B：Spec-Kit（规范驱动开发）
这是更高阶的做法：
- 用结构化 Spec（YAML/Markdown）定义任务目标、验收标准、技术约束
- Harness 读取 Spec 后，自动拆解为子任务，调用模型和工具逐步完成
- 每一步都与 Spec 对齐，确保不偏离需求

**本质**：把「需求文档」变成「可执行契约」，Harness 负责监督和履约。

---

## 五、对程序员意味着什么？

**判断：程序员的工作重心将从写代码转向写规则和技能。**

这可以拆解成三个层面的迁移：

### 1. 从「写逻辑」到「写规则」
以前：if/else、循环、状态机，代码表达所有业务逻辑。
未来：用自然语言/结构化文档描述「什么是对的、什么是错的、边界在哪里」， Harness 负责把这些规则翻译给大模型并监督执行。

### 2. 从「做功能」到「搭系统」
以前：程序员的核心产出是代码文件。
未来：核心产出是「Agent 系统」——大模型 + 工具链 + 反馈机制 + 记忆存储。代码只是 Harness 的一部分。

### 3. 从「个体手艺」到「工程规范」
提示词工程太依赖个人技巧，换个人效果可能差很多。
Harness Engineering 强调的是**可复用、可交接、可规模化**的工程能力。这正是「存量程序员」的主战场——不是拼谁 Prompt 写得花，而是拼谁能让 AI 在复杂业务里**稳定交付**。

---

## 六、CLAUDE.md 模板

Claude Code / Cursor / 其他 Agent 工具都能读取的项目记忆文件。建议放在**项目根目录**。

```markdown
# CLAUDE.md - 项目上下文

## 1. 项目简介
- **项目名称**：[名称]
- **核心目标**：[一句话说清楚这个系统解决什么问题]
- **目标用户**：[谁用]
- **当前阶段**：[MVP / 迭代期 / 维护期]

## 2. 技术栈
- **语言/框架**：[如 TypeScript + Next.js / Java + Spring Boot]
- **数据库**：[如 PostgreSQL / MongoDB]
- **关键依赖**：[列出 3-5 个核心库]
- **部署方式**：[Docker / Vercel / K8s]

## 3. 目录结构约定
```
src/
├── components/     # 纯 UI 组件，无业务逻辑
├── pages/          # 路由页面
├── hooks/          # 自定义 React Hooks
├── lib/            # 工具函数、API 客户端
├── services/       # 核心业务逻辑
├── types/          # TypeScript 类型定义
└── tests/          # 测试文件（与源文件同名）
```

## 4. 编码规范
- **命名**：组件用 PascalCase，函数/变量用 camelCase，常量用 UPPER_SNAKE_CASE
- **类型**：所有函数参数和返回值必须显式声明类型
- **错误处理**：不用 `any`，异步操作必须 try/catch 并返回统一错误格式
- **注释**：复杂逻辑必须写「为什么」而不是「做什么」
- **测试**：新增功能必须附带单元测试

## 5. 常见任务 SOP
### 新增 API 接口
1. 在 `src/types/` 定义请求/响应类型
2. 在 `src/services/` 实现业务逻辑
3. 在 `src/pages/api/` 注册路由
4. 补充测试和文档

### 新增 UI 组件
1. 在 `src/components/` 创建组件文件
2. props 必须定义 interface
3. 在 Storybook 或同级目录补充示例

## 6. 禁止事项（安全边界）
- ❌ 不要修改 `.env` 文件
- ❌ 不要执行删除数据库/表的命令
- ❌ 不要修改 CI/CD 配置（`.github/workflows`）除非用户明确要求
- ❌ 不要引入未经确认的新依赖

## 7. 沟通偏好
- 执行任务前先输出简要计划
- 遇到不确定的需求，先确认再执行
- 完成后总结改动点和潜在风险
```

---

## 七、Harness 系统最小可行架构（MVP）

### 核心模块图

```
┌─────────────────────────────────────────┐
│              User Input                 │
│         (需求 / 命令 / 问题)             │
└────────────────┬────────────────────────┘
                 ▼
┌─────────────────────────────────────────┐
│           Orchestrator                  │
│         编排层：任务拆解 + 路由           │
│   (Plan → Decide → Route to Tool/LLM)   │
└────────────────┬────────────────────────┘
                 ▼
┌─────────────────────────────────────────┐
│           Executor                      │
│         执行层：工具调用 + 代码运行        │
│   (Shell / File / API / Code Runner)    │
└────────────────┬────────────────────────┘
                 ▼
┌─────────────────────────────────────────┐
│           Validator                     │
│         反馈层：结果校验 + 错误回传        │
│   (Syntax Check / Test / Diff Review)   │
└────────┬───────────────────┬────────────┘
         │                   │
    通过 ▼              失败 ▼
┌──────────────┐    ┌──────────────┐
│   Memory     │◄───│  Retry Loop  │
│  记忆沉淀     │    │  自动重试    │
└──────────────┘    └──────────────┘
```

### MVP 代码实现（Python 伪代码）

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
import json

@dataclass
class Task:
    id: str
    description: str
    tool: Optional[str]  # "llm", "shell", "file_read", "file_write"
    input_data: Dict
    output_data: Optional[Dict] = None
    status: str = "pending"  # pending, running, success, failed

@dataclass
class ProjectMemory:
    """长期记忆：项目规范 + 历史决策"""
    claude_md: str
    history_decisions: List[Dict]

class Orchestrator:
    """编排层：把用户需求拆成任务链"""
    
    def __init__(self, llm_client):
        self.llm = llm_client
    
    def plan(self, user_input: str, memory: ProjectMemory) -> List[Task]:
        prompt = f"""
        根据以下项目规范和用户需求，拆解为可执行的任务列表。
        每个任务必须指定工具：llm / shell / file_read / file_write
        
        项目规范：
        {memory.claude_md}
        
        用户需求：{user_input}
        
        输出格式（严格 JSON）：
        [
          {{"id": "1", "description": "...", "tool": "file_read", "input_data": {{"path": "..."}}}},
          {{"id": "2", "description": "...", "tool": "llm", "input_data": {{"prompt": "..."}}}},
          {{"id": "3", "description": "...", "tool": "file_write", "input_data": {{"path": "...", "content": "..."}}}}
        ]
        """
        response = self.llm.chat(prompt)
        tasks = json.loads(response)
        return [Task(**t) for t in tasks]

class Executor:
    """执行层：实际运行任务"""
    
    def __init__(self, llm_client):
        self.llm = llm_client
        self.tools = {
            "shell": self._run_shell,
            "file_read": self._read_file,
            "file_write": self._write_file,
            "llm": self._call_llm,
        }
    
    def run(self, task: Task) -> Dict:
        if task.tool not in self.tools:
            raise ValueError(f"Unknown tool: {task.tool}")
        result = self.tools[task.tool](task.input_data)
        task.output_data = result
        task.status = "success"
        return result
    
    def _run_shell(self, data):
        import subprocess
        cmd = data["command"]
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
        return {
            "stdout": result.stdout,
            "stderr": result.stderr,
            "returncode": result.returncode
        }
    
    def _read_file(self, data):
        with open(data["path"], "r", encoding="utf-8") as f:
            return {"content": f.read()}
    
    def _write_file(self, data):
        with open(data["path"], "w", encoding="utf-8") as f:
            f.write(data["content"])
        return {"status": "written", "path": data["path"]}
    
    def _call_llm(self, data):
        response = self.llm.chat(data["prompt"])
        return {"content": response}

class Validator:
    """反馈层：校验执行结果，决定是否重试"""
    
    def __init__(self, llm_client):
        self.llm = llm_client
    
    def check(self, task: Task):
        # 基础校验
        if task.tool == "shell":
            if task.output_data["returncode"] != 0:
                return False, f"Shell error: {task.output_data['stderr']}"
        
        if task.tool == "file_write":
            # 让 LLM 检查写入内容是否符合规范
            prompt = f"""
            检查以下代码/内容是否符合项目规范。如有问题，列出具体问题。
            内容：{task.output_data}
            """
            review = self.llm.chat(prompt)
            if "不符合" in review or "error" in review.lower():
                return False, review
        
        return True, "OK"

class Harness:
    """主控：把四层能力串起来"""
    
    def __init__(self, llm_client, claude_md_path: str):
        self.llm = llm_client
        self.orchestrator = Orchestrator(llm_client)
        self.executor = Executor(llm_client)
        self.validator = Validator(llm_client)
        self.memory = self._load_memory(claude_md_path)
    
    def _load_memory(self, path: str) -> ProjectMemory:
        with open(path, "r", encoding="utf-8") as f:
            claude_md = f.read()
        return ProjectMemory(claude_md=claude_md, history_decisions=[])
    
    def run(self, user_input: str, max_retries: int = 3):
        print(f"🎯 收到需求: {user_input}")
        
        # 1. 编排：拆解任务
        tasks = self.orchestrator.plan(user_input, self.memory)
        print(f"📋 拆解为 {len(tasks)} 个子任务")
        
        # 2. 顺序执行（简单版，可扩展为 DAG）
        for task in tasks:
            print(f"▶️ 执行任务: {task.description}")
            
            retries = 0
            while retries <= max_retries:
                try:
                    self.executor.run(task)
                    passed, msg = self.validator.check(task)
                    
                    if passed:
                        print(f"✅ 通过校验")
                        break
                    else:
                        print(f"⚠️ 校验失败: {msg}")
                        task.input_data["feedback"] = msg
                        retries += 1
                        
                except Exception as e:
                    print(f"❌ 执行异常: {e}")
                    retries += 1
            
            if retries > max_retries:
                print(f"🛑 任务 {task.id} 最终失败，需要人工介入")
                return {"status": "failed", "task": task.id}
        
        # 3. 记忆沉淀：记录本次决策
        self.memory.history_decisions.append({
            "input": user_input,
            "tasks": [{"id": t.id, "status": t.status} for t in tasks]
        })
        
        print("🎉 全部任务完成")
        return {"status": "success", "tasks": tasks}
```

### 这个 MVP 能做什么？

| 场景 | Harness 行为 |
|------|-------------|
| 用户说「给这个项目加个登录接口」| 读取 `CLAUDE.md` → 计划：读目录结构 → 设计类型 → 写 service → 写 API 路由 → 写测试 → 逐一执行并校验 |
| 代码编译报错 | Executor 运行 `npm run build`，Validator 捕获 stderr → 反馈给 LLM 自动修复 → 重试 |
| 上下文溢出 | Orchestrator 在 Plan 阶段只加载必要文件，不一次性塞全部代码 |

### 从 MVP 到生产级的升级路径

```
MVP 阶段          →  进阶阶段           →  生产级
─────────────────────────────────────────────────────────
顺序执行           →  DAG 依赖调度       →  多 Agent 协作
本地 Shell         →  沙箱化执行环境      →  K8s / 安全容器
简单 LLM 校验      →  单元测试 + Lint    →  CI 流水线集成
文件存储记忆        →  Vector DB (RAG)   →  知识图谱 + 决策树
```

---

## 八、核心结论

> **提示词工程解决「说清楚」，上下文工程解决「给够信息」，驾驭工程解决「持续做对并交付」。**

AI 开发的本质，不是让大模型变得更聪明，而是为它搭建一个**可控、可观测、可迭代**的工程外壳。谁掌握 Harness Engineering，谁就掌握了把「通用智力」转化为「生产工具」的钥匙。

---

*记录时间：2026-04-14*
