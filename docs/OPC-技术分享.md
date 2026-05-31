# Claude Code Agent 编排实战：如何构建 12 人 AI 开发团队

> 基于 [Claude Standard Dev Team](https://github.com/xuanbingbingo/claude-standard-dev-team) 项目的实践与思考

---

## 目录

1. [Claude Code 简介](#一claude-code-简介)
2. [Claude Code Agent 机制深度解析](#二claude-code-agent-机制深度解析)
3. [多 Agent 编排原理](#三多-agent-编排原理)
4. [Claude Standard Dev Team 项目解析](#四claude-standard-dev-team-项目解析)
5. [实践与思考](#五实践与思考)

---

## 一、Claude Code 简介

### 1.1 Claude Code 是什么？

**Claude Code** 是 Anthropic 官方推出的 AI 编程助手，深度集成 Claude 大模型能力。

**产品形态**：
- **CLI 命令行**：终端中使用，支持全系统
- **IDE 插件**：VS Code、JetBrains 系列
- **Web 端**：claude.ai/code

**定位**：不只是代码补全工具，而是**能独立完成复杂开发任务的 AI Agent**。

### 1.2 Claude Code 核心能力

| 能力 | 说明 | 示例 |
|------|------|------|
| **代码理解与编辑** | 读取、修改、重构代码 | "帮我重构这个函数" |
| **Shell 执行** | 运行命令、脚本、测试 | "运行测试并修复失败的用例" |
| **Agent 机制** | 启动子代理处理专业任务 | 自动派遣专门的 agent |
| **Skill 机制** | 加载剧本，执行复杂流程 | 自定义工作流 |
| **MCP 协议** | 连接外部工具和数据源 | 数据库、API、文件系统 |

### 1.3 Claude Code vs 其他 AI 编程工具

| 维度 | Claude Code | Cursor | GitHub Copilot |
|------|-------------|--------|----------------|
| **底层模型** | Claude 4/3.5 | Claude/GPT | GPT-4 |
| **Agent 能力** | ✅ 原生支持 | ⚠️ 有限支持 | ❌ 无 |
| **自定义 Agent** | ✅ 完整支持 | ⚠️ 部分 | ❌ 无 |
| **命令行操作** | ✅ 原生支持 | ⚠️ 有限 | ❌ 无 |
| **MCP 协议** | ✅ 支持 | ❌ 不支持 | ❌ 不支持 |
| **Skill 剧本** | ✅ 支持 | ❌ 不支持 | ❌ 不支持 |

**关键差异**：Claude Code 的 Agent 机制使其能够**编排多个专业 AI 角色协作完成复杂任务**。

### 1.4 为什么选择 Claude Code？

```
传统 AI 编程工具：补全代码片段

Claude Code：
    ├── 理解项目结构
    ├── 执行命令
    ├── 启动专业 Agent
    ├── 协调多 Agent 协作
    └── 完成端到端开发任务
```

---

## 二、Claude Code Agent 机制深度解析

### 2.1 什么是 Agent？

**定义**：Agent（子代理）是一个**独立的 LLM 会话**，拥有特定的角色定义和工具集。

```
主会话 (Main Session)
    │
    ├── 可以直接执行工具（Read/Write/Bash...）
    │
    └── 可以启动 Agent (Subagent)
            │
            ├── 独立的 LLM 会话
            ├── 独立的上下文窗口
            ├── 独立的工具集
            └── 完成后返回结果给主会话
```

### 2.2 Agent 的核心特性

#### 特性一：角色专业化

每个 Agent 都有明确的职责边界，定义在 `.md` 文件中：

```yaml
# agents/backend-architect.md
---
name: backend-architect
description: 后端工程师。当需要实现 API 接口、业务逻辑时激活。
---

你是一名后端工程师，职责：
1. 严格按 API_CONTRACT.md 实现接口
2. 字段名、路径、方法不得偏差
3. 遇到歧义上报不自决

可使用的工具：Read, Write, Edit, Bash, Grep, Glob
```

#### 特性二：工具隔离

Agent 可以拥有**独立的工具集**，不是所有 agent 都能访问所有工具：

```yaml
# 某些 agent 可能没有写权限
tools:
  - Read      # 读取文件
  - Grep      # 搜索内容
  - Glob      # 搜索文件
  # 没有 Write/Edit，只读分析
```

#### 特性三：独立上下文

Agent 运行在**独立的 LLM 会话**中：
- 不会污染主会话的上下文
- 有自己的 token 预算
- 完成后只返回结果摘要，不返回中间过程

```
主会话 token 消耗：
    ├── 自己的操作：少量
    └── Agent 返回的摘要：少量

Agent 内部的 token 消耗：
    └── 独立计算，不影响主会话
```

### 2.3 Agent 的平台限制（关键！）

**限制：Subagent 不能 spawn 其他 subagent**

```
❌ 不允许：
Main Session → Agent A → Agent B → Agent C

✅ 允许：
Main Session → Agent A
Main Session → Agent B
Main Session → Agent C
```

**实测证据**（来自 Claude Standard Dev Team 项目）：
- orchestrator 若作为 subagent 启动
- Task 工具会被运行时强制剥离
- 无法调度其他 agent

**应对方案**：使用 **Skill** 代替 Agent 作为总指挥

### 2.4 Skill vs Agent

| 维度 | Agent | Skill |
|------|-------|-------|
| **执行主体** | 独立 LLM 会话 | 主会话 |
| **能否调度 Agent** | ❌ 不能 | ✅ 可以 |
| **上下文** | 独立隔离 | 共享主会话 |
| **Token 消耗** | 独立计算 | 计入主会话 |
| **适用场景** | 专业任务执行 | 编排调度、流程控制 |

```
Skill（剧本）
    │
    └── 主会话 load 后直接执行
            │
            ├── 可以用 Task 工具调度多个 Agent
            └── 这就是"总指挥"的正确实现方式
```

### 2.5 Agent 文件结构

```
~/.claude/agents/
├── product-manager.md
├── software-architect.md
├── backend-architect.md
├── frontend-developer.md
├── testing-evidence-collector.md
└── ...

每个 .md 文件结构：
---
name: agent-name                    # Agent 名称
description: 触发条件描述            # 什么时候激活
---

# 角色定义

你是一名 [角色]，职责是 [职责]。

## 输入
- 读取 xxx 文件

## 输出
- 生成 xxx 文件

## 约束
- 必须遵守 xxx 规范
- 遇到 xxx 情况上报不自决
```

### 2.6 Skill 文件结构

```
~/.claude/skills/
└── standard-team/
    └── SKILL.md

SKILL.md 结构：
---
name: standard-team
description: 触发条件描述
---

# 剧本内容

主会话 load 后按剧本执行：
1. Phase 0: 创建目录
2. Phase 1: Task → product-manager
3. Phase 2: Task → software-architect
...
```

---

## 三、多 Agent 编排原理

### 3.1 为什么要多 Agent 编排？

**问题**：一个 AI 干所有事 → 角色混乱、质量不可控

```
单 Agent 模式：
User → AI → 代码（全干，角色混乱）

问题：
- 前后端接口对不上
- 自己写自己测
- 改一处坏一片
```

**解决**：专业分工 + 协作流程

```
多 Agent 模式：
User → Orchestrator → PM → Architect → Developer → QA → 代码

优势：
- 职责清晰
- 质量可控
- 可追溯
```

### 3.2 编排的核心要素

#### 要素一：契约（Contract）

所有 Agent 对着同一份契约工作：

| 契约文件 | 内容 | 作用 |
|----------|------|------|
| `API_CONTRACT.md` | 接口定义 | 前后端对齐 |
| `DB_SCHEMA.md` | 数据结构 | DB 与代码对齐 |
| `TECH_SPEC.md` | 技术规范 | 全局约束 |

**契约即"宪法"**：一旦确定，所有 Agent 必须遵守。

#### 要素二：流程（Workflow）

定义清晰的阶段和检查点：

```
Phase 0   初始化
Phase 1   需求分析    → ⏸ 人工检查点
Phase 2   契约生成    → ⏸ 人工检查点（最关键）
Phase 3   任务拆解
Phase 4   数据库实现
Phase 5   后端实现    → Dev-QA Loop
Phase 6   前端实现    → Dev-QA Loop
Phase 7   安全审计
Phase 8   代码评审
Phase 9   部署配置
Phase 10  最终验收
Phase 11  文档输出
```

#### 要素三：验证（Validation）

独立的 QA Agent：
- **不参与实现**
- **只对照契约验证**
- **PASS/FAIL 决策**

```
实现 Agent：写代码
    ↓
QA Agent：独立验证（不知道实现细节）
    ↓
PASS → 完成
FAIL → 打回重做
```

### 3.3 编排模式对比

| 模式 | 说明 | 优势 | 劣势 | 适用场景 |
|------|------|------|------|----------|
| **串行** | 逐个 Agent 执行 | 质量可控 | 效率低 | 质量优先 |
| **并行** | 多个 Agent 同时执行 | 效率高 | 冲突风险 | 效率优先 |
| **混合** | 部分串行 + 部分并行 | 平衡 | 复杂度高 | 大型项目 |

**Claude Standard Dev Team 选择串行**：质量 > 速度

### 3.4 Dev-QA Loop 机制

这是多 Agent 编排的核心创新——**每个任务都有独立的 QA 闭环**。

```
FOR 每个任务:
    │
    ├─ STEP 1: 实现 Agent 执行
    │           • 读取契约
    │           • 实现代码
    │           • 返回完成状态
    │
    ├─ STEP 2: QA Agent 验证（独立会话！）
    │           • 读取契约
    │           • 检查实现代码
    │           • 对照契约验证
    │           • 返回 PASS/FAIL
    │
    └─ STEP 3: 决策
                ├─ PASS → 标记完成，下一任务
                ├─ FAIL（< 3次）→ 打回重做
                └─ FAIL（>= 3次）→ 暂停，报告用户
```

**关键洞察**：

QA Agent 是**独立的 LLM session**：
- 不知道实现者内部怎么写的
- 只对照契约验证产出代码
- 这种"评审者不是实现者"的设计，比自测可靠 10 倍

### 3.5 打回机制

```
问题分类：
    │
    ├── 契约有问题 → 打回 architect 修改契约
    │
    ├── 实现有问题 → 打回对应实现 agent
    │
    └── 评审有问题 → 打回对应 agent 修复

重试上限：
    ├── 任务级：3 次
    ├── 契约级：2 次
    └── 超限 → 暂停，等待用户介入
```

---

## 四、Claude Standard Dev Team 项目解析

### 4.1 项目定位

**一套面向 Claude Code 的 12 人 AI 开发团队配置**

```
12 个专业 Agent + 1 个总指挥 Skill = 虚拟开发团队
```

**核心价值**：
- 让 OPC（一人公司）拥有虚拟开发团队
- 从需求到上线全流程自动化
- 30-90 分钟完成中型应用开发

### 4.2 团队架构

```
                ┌─────────────────────────────┐
                │  standard-team (skill)      │
                │  总指挥剧本，主会话执行       │
                └────────────┬────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   规划层 2人            实现层 5人           质量层 4人
   ──────────           ──────────           ──────────
   product-manager      database-optimizer   testing-evidence-collector
   software-architect   backend-architect    security-engineer
                        ui-designer          code-reviewer
                        frontend-developer   reality-checker
                        devops-automator
                                          文档层 1人
                                          ──────────
                                          technical-writer
```

### 4.3 各 Agent 职责详解

| 层级 | Agent | 职责 | 输入 | 输出 |
|------|-------|------|------|------|
| **规划** | product-manager | 需求分析、PRD 编写 | 用户需求 | `PRD.md` |
| **规划** | software-architect | 技术选型、契约生成 | `PRD.md` | `API_CONTRACT.md`, `DB_SCHEMA.md` |
| **实现** | ui-designer | 设计规范 | `PRD.md` | `DESIGN_SYSTEM.md` |
| **实现** | database-optimizer | 数据库迁移 | `DB_SCHEMA.md` | `migrations/` |
| **实现** | backend-architect | 后端接口实现 | `API_CONTRACT.md` | 后端代码 |
| **实现** | frontend-developer | 前端 UI 实现 | `API_CONTRACT.md` | 前端代码 |
| **实现** | devops-automator | 部署配置 | 项目代码 | `Dockerfile` |
| **质量** | testing-evidence-collector | 任务级 QA | 实现代码 | PASS/FAIL |
| **质量** | security-engineer | 安全审计 | 项目代码 | `SECURITY_REPORT.md` |
| **质量** | code-reviewer | 代码评审 | 项目代码 | `REVIEW_REPORT.md` |
| **质量** | reality-checker | 最终验收 | 全部产出 | `ACCEPTANCE_REPORT.md` |
| **文档** | technical-writer | 文档编写 | 全部产出 | `README.md` |

### 4.4 11 阶段工作流详解

```
Phase 0   主会话创建项目目录
Phase 1   → product-manager      → PRD.md
   ⏸ 人工检查点 #1：确认功能范围
Phase 2   → software-architect   → API_CONTRACT.md / DB_SCHEMA.md
   ⏸ 人工检查点 #2：确认接口契约（最关键！）
Phase 2.5 → ui-designer          → DESIGN_SYSTEM.md
Phase 3   主会话拆解任务清单
Phase 4   → database-optimizer   → migrations/
Phase 5   → backend-architect    ┐ Dev-QA Loop
          → testing-evidence-collector ┘ 逐任务验证
Phase 6   → frontend-developer   ┐ Dev-QA Loop
          → testing-evidence-collector ┘ 逐任务验证
Phase 7   → security-engineer    → SECURITY_REPORT.md
Phase 8   → code-reviewer        → REVIEW_REPORT.md
Phase 9   → devops-automator     → Dockerfile + 部署检查
Phase 10  → reality-checker      → 最终验收
Phase 11  → technical-writer     → README + API_DOC
```

**为什么只在 2 个地方暂停？**

| 检查点 | 原因 |
|--------|------|
| Phase 1 后 | PRD 确认后才能生成契约，改 PRD = 重跑全部 |
| Phase 2 后 | 契约确认后才能开发，改契约 = 重跑 Phase 4-6 |

其他阶段有自动 Dev-QA Loop，能自愈，不需要人工介入。

### 4.5 关键设计决策

#### 决策一：总指挥用 Skill 不用 Agent

```
❌ 错误设计：
orchestrator (Agent) → Task → 其他 Agent
    ↓
Task 工具被剥离，无法调度

✅ 正确设计：
主会话 load skill → Task → 多个 Agent
    ↓
主会话有 Task 工具，可以调度
```

#### 决策二：QA Agent 必须独立

```
❌ 自己测自己：
backend-architect → 写代码 → 自己检查 → "OK"
    ↓
实际有 bug，但自己看不见

✅ 独立验证：
backend-architect → 写代码
    ↓
testing-evidence-collector → 独立验证 → FAIL
    ↓
打回重做
```

#### 决策三：零容忍硬编码

```typescript
// ❌ 禁止 - 部署时必定 404
axios.get('/api/v1/users')

// ✅ 正确 - 使用环境变量
const API_BASE = import.meta.env.VITE_API_BASE ?? ''
axios.get(`${API_BASE}/api/v1/users`)
```

**事故背景**：子路径部署时，所有硬编码 `/api/...` 的请求全部 404。

---

## 五、实践与思考

### 5.1 如何自定义 Agent

创建 `~/.claude/agents/my-agent.md`：

```markdown
---
name: my-custom-agent
description: 当需要 xxx 时激活
---

# 角色定义

你是一名 [角色]，职责是 [职责]。

## 工具权限
- Read: 读取文件
- Write: 写入文件
- Edit: 编辑文件
- Bash: 执行命令

## 输入
- 读取 xxx 文件

## 输出
- 生成 xxx 文件

## 约束
- 必须遵守 xxx 规范
- 遇到 xxx 情况上报不自决
- 不得 xxx
```

### 5.2 如何自定义 Skill

创建 `~/.claude/skills/my-skill/SKILL.md`：

```markdown
---
name: my-workflow
description: 当用户说 "用 xxx 流程" 时激活
---

# 工作流程剧本

主会话 load 本剧本后按以下步骤执行：

## Phase 1: 初始化
- 创建目录结构

## Phase 2: 执行 Agent
- Task → my-agent

## Phase 3: 验证
- 检查产出文件

## 人工检查点
- Phase 2 后暂停，等用户确认
```

### 5.3 适用场景

**适合**：
- 中型应用开发（前后端 + DB + 部署）
- 需要质量保障的项目
- 有复杂协作流程的需求
- OPC / 独立开发者

**不适合**：
- 小工具、原型验证
- 单文件脚本
- 简单 CRUD

### 5.4 经验教训

每条规则背后都是真实事故：

| 事故 | 规则 | 教训 |
|------|------|------|
| 子路径部署全部 404 | API 路径禁止硬编码 | 部署环境多变，必须用环境变量 |
| 后端改字段前端不知道 | 契约变更必须同步前端 | 改契约是大事，必须广播 |
| AI 自己测说 OK 实际有 bug | QA Agent 必须独立 | 自己测自己是靠不住的 |
| 重试无限循环 | 打回有上限 | 死循环浪费资源，必须有人工介入 |

**核心教训**：质量 > 速度，宁可慢一点串行跑，也不要让两个 agent 同时改代码。

### 5.5 未来展望

**Agent 编排的演进方向**：
- 从"一把梭"到"专业分工"
- 从"自己测自己"到"独立验证"
- 从"边写边改"到"契约驱动"
- 编排模式的标准化

**对开发者的影响**：
- OPC 可以拥有虚拟团队
- 开发者角色从"写代码"转向"指挥 AI"
- 核心能力变成：需求理解、架构设计、质量把控
- 一个人 = 一个团队的战斗力

---

## 参考资料

- [Claude Standard Dev Team - GitHub](https://github.com/xuanbingbingo/claude-standard-dev-team)
- [Claude Code 官方文档](https://claude.com/claude-code)
- [Claude Code Agent 机制](https://docs.claude.com/en/docs/claude-code/sub-agents)
- [MCP 协议说明](https://modelcontextprotocol.io/)

---

## 附录

### A. 项目文件结构

```
claude-standard-dev-team/
├── agents/                    # 12 个成员 agent
│   ├── product-manager.md
│   ├── software-architect.md
│   ├── backend-architect.md
│   ├── frontend-developer.md
│   ├── testing-evidence-collector.md
│   └── ...
├── skills/
│   └── standard-team/
│       └── SKILL.md          # 总指挥剧本
├── README.md
├── WORKFLOW.md               # 11 阶段详解
└── INSTALL.md
```

### B. Agent 文件示例

```markdown
# agents/backend-architect.md

---
name: backend-architect
description: 后端工程师。当需要实现 API 接口、业务逻辑、服务层代码时激活。由 orchestrator 在 Phase 5 调用，在 Dev-QA Loop 中逐任务实现接口。严格按照 API_CONTRACT.md 实现，字段名路径方法不得偏差，遇到歧义上报不自决。
---

你是一名后端工程师...

## 职责
1. 读取 API_CONTRACT.md（必须第一步）
2. 严格按照契约实现接口
3. 字段名、路径、HTTP 方法不得偏差

## 约束
- 禁止自行修改契约
- 遇到契约歧义写入 BACKEND_STATUS.md ISSUES 章节
```

---

> 文档基于 [Claude Standard Dev Team](https://github.com/xuanbingbingo/claude-standard-dev-team) 项目整理，适用于团队内部技术分享。
