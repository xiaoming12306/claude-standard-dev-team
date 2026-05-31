# 如何让一个人拥有团队级的战斗力

> 基于 Claude Standard Dev Team 项目的实践与深度思考
> 适用场景：独立开发者、小型技术团队、想要提升个人产出的工程师
> 核心价值：用 12 个 AI Agent 实现一人即团队

---

## 目录

1. [开发者的日常痛点](#一开发者的日常痛点)
2. [解决思路：组建虚拟团队](#二解决思路组建虚拟团队)
3. [整体架构](#三整体架构)
4. [核心创新：Dev-QA Loop](#四核心创新dev-qa-loop)
5. [契约驱动设计](#五契约驱动设计)
6. [零容忍硬编码规则](#六零容忍硬编码规则)
7. [完整工作流程](#七完整工作流程)
8. [核心价值总结](#八核心价值总结)

---

## 一、开发者的日常痛点

### 1.1 你可能遇到过的场景

**场景一：硬编码翻车**

```
情况：项目要部署到子路径 /my-app/
问题：所有 /api/... 请求全部 404
原因：代码里写了 axios.get('/api/v1/users')
后果：紧急修复，重新部署
```

**场景二：接口字段变更**

```
后端：把 userId 改成 user_id
前端：还在用 userId
结果：接口调用失败，上线后才发现
```

**场景三：AI 自测不可靠**

```
AI：写完代码后自己检查
AI："测试通过了，应该没问题"
实际：有 bug，但 AI 有思维盲区，自己看不见
结果：用户反馈才发现问题
```

### 1.2 问题背后的根因

### 1.2 问题背后的根因

| 问题 | 根因 |
|------|------|
| 前后端对不上 | 一个 AI 全干，改了后端忘了前端 |
| 部署翻车 | 没有部署环境检查 |
| 质量不可控 | AI 自己测自己，靠不住 |
| 交付不可控 | AI 说"写完了"，实际没写完 |

**一句话总结**：一个 AI 干所有事 = 角色混乱 + 质量不可控

---

## 二、解决思路：一人即团队

### 2.1 类比传统团队分工

```
传统团队                     虚拟团队（AI Agent）
─────────────────────────────────────────────────
产品经理         →          product-manager
架构师           →          software-architect
前端工程师       →          frontend-developer
后端工程师       →          backend-architect
数据库工程师     →          database-optimizer
QA              →          testing-evidence-collector
安全工程师       →          security-engineer
代码评审         →          code-reviewer
运维            →          devops-automator
技术文档         →          technical-writer
```

### 2.2 核心思路

**让 AI 也按角色分工，而不是一个 AI 全干**

传统开发模式：
```
一个人 = 一个开发者
```

AI 辅助模式：
```
一个人 + AI = 1.5 个开发者
```

AI Agent 团队模式：
```
一个人 + 12 个 Agent = 团队级的战斗力
```

### 2.3 这不是要替代团队

**澄清**：这个方案的目标不是替代团队协作，而是：

- 帮助**独立开发者**提升产能
- 帮助**小团队**补齐角色缺口
- 帮助**想要提升个人产出**的工程师

对于大型团队项目，传统协作模式仍然是最优解。

---

## 三、整体架构

### 3.1 团队架构

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

### 3.2 各 Agent 职责

| 层级 | Agent | 职责 | 输入 | 输出 |
|------|-------|------|------|------|
| **规划** | product-manager | 需求分析 | 用户需求 | PRD.md |
| **规划** | software-architect | 契约生成 | PRD.md | API_CONTRACT.md, DB_SCHEMA.md |
| **实现** | ui-designer | 设计规范 | PRD.md | DESIGN_SYSTEM.md |
| **实现** | database-optimizer | 数据库迁移 | DB_SCHEMA.md | migrations/ |
| **实现** | backend-architect | 后端实现 | API_CONTRACT.md | 后端代码 |
| **实现** | frontend-developer | 前端实现 | API_CONTRACT.md | 前端代码 |
| **实现** | devops-automator | 部署配置 | 项目代码 | Dockerfile |
| **质量** | testing-evidence-collector | 任务级 QA | 实现代码 | PASS/FAIL |
| **质量** | security-engineer | 安全审计 | 项目代码 | SECURITY_REPORT.md |
| **质量** | code-reviewer | 代码评审 | 项目代码 | REVIEW_REPORT.md |
| **质量** | reality-checker | 最终验收 | 全部产出 | ACCEPTANCE_REPORT.md |
| **文档** | technical-writer | 文档编写 | 全部产出 | README.md |

### 3.3 关键设计：总指挥是 Skill 不是 Agent

**平台限制**：Subagent 不能 spawn 其他 subagent

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

**实测证据**：orchestrator 作为 Agent 启动时，Task 工具被运行时强制剥离。

---

## 四、核心创新：Dev-QA Loop

**这是与传统 AI 编程最大的区别。**

### 4.1 传统模式：AI 自己测自己

```
AI 写代码 → AI 自己检查 → "测试通过，应该 OK"
    ↓
实际有 bug，但 AI 有思维盲区
```

### 4.2 Dev-QA Loop 模式：独立验证

```
FOR 每个任务:
    │
    ├─ STEP 1: 实现 Agent 执行
    │           • 读取契约
    │           • 实现代码
    │
    ├─ STEP 2: QA Agent 验证（独立会话！）
    │           • 读取契约
    │           • 检查实现代码
    │           • 对照契约验证
    │           • 返回 PASS/FAIL
    │
    └─ STEP 3: 决策
                ├─ PASS → 下一任务
                ├─ FAIL（< 3次）→ 打回重做
                └─ FAIL（>= 3次）→ 暂停，报告用户
```

### 4.3 为什么比自测可靠 10 倍？

**心理学原理**：
- 实现者知道自己怎么写的 → 有思维盲区
- QA Agent 只看契约和代码 → 没有实现者的思维惯性
- "评审者不是实现者" — 人类团队的最佳实践，AI 也一样

**技术原理**：
- QA Agent 是**独立的 LLM session**
- 不知道实现者内部怎么写的
- 只对照契约验证产出代码

### 4.4 打回机制

```
问题分类：
    │
    ├── 契约有问题 → 打回 architect 修改契约（上限 2 次）
    │
    ├── 实现有问题 → 打回对应实现 agent（上限 3 次/任务）
    │
    └── 评审有问题 → 打回对应 agent 修复（上限 1-2 次）

超限 → 暂停，等待用户介入
```

**为什么有上限？**
- 避免无限循环浪费资源
- 多次失败说明问题严重，需要人工介入

---

## 五、契约驱动设计

### 5.1 先定契约，再写代码

**所有 Agent 对着同一份契约工作：**

| 契约文件 | 内容 | 作用 |
|----------|------|------|
| `API_CONTRACT.md` | 接口定义 | 前后端对齐 |
| `DB_SCHEMA.md` | 数据结构 | DB 与代码对齐 |
| `TECH_SPEC.md` | 技术规范 | 全局约束 |

**契约即"宪法"**：一旦确定，所有 Agent 必须遵守。

### 5.2 契约变更规则

```
架构师改 API_CONTRACT 后：
    │
    ├── 必须同步扫描前端 types.ts
    │
    └── 字段名/类型变更必须列出"前端需同步改的位置"
```

**为什么这么严格？**
- 改契约是大事
- 必须广播到所有相关方
- 避免前后端对不上

---

## 六、零容忍硬编码规则

**每条规则背后都是真实部署翻车事故。**

### 6.1 事故背景

```typescript
// AI 写的代码
axios.get('/api/v1/users')  // 看起来没问题

// 部署到子路径 /my-app/ 时
// 实际请求：https://example.com/api/v1/users
// 正确应该：https://example.com/my-app/api/v1/users
// 结果：全部 404
```

### 6.2 正确做法

```typescript
// ✅ 使用环境变量
const API_BASE = import.meta.env.VITE_API_BASE ?? ''
axios.get(`${API_BASE}/api/v1/users`)
```

### 6.3 Phase 9 强制检查

1. 确认 `frontend/.env.production` 存在且包含 `VITE_API_BASE=/{APP_PATH}`
2. grep 扫描 `frontend/src/` 确认无硬编码 `/api/` 调用路径

**检查不通过 → 不允许进入下一阶段**

---

## 七、完整工作流程

### 7.1 11 阶段流程

| Phase | 执行者 | 产出 | 人工检查 |
|-------|--------|------|----------|
| 0 | 主会话 | 创建目录结构 | - |
| 1 | product-manager | PRD.md | ⏸ 确认功能范围 |
| 2 | software-architect | API/DB/TECH 契约 | ⏸ 确认接口契约（最关键）|
| 2.5 | ui-designer | 设计规范 | - |
| 3 | 主会话 | 任务清单 | - |
| 4 | database-optimizer | migrations/ | - |
| 5 | backend-architect + QA | 后端代码 | - |
| 6 | frontend-developer + QA | 前端代码 | - |
| 7 | security-engineer | 安全报告 | - |
| 8 | code-reviewer | 评审报告 | - |
| 9 | devops-automator | 部署配置 | - |
| 10 | reality-checker | 验收报告 | - |
| 11 | technical-writer | README/API_DOC | - |

### 7.2 为什么只在 2 处暂停？

| 检查点 | 原因 |
|--------|------|
| Phase 1 后 | 改 PRD = 重跑全部 |
| Phase 2 后 | 改契约 = 重跑 Phase 4-6 |

**其他阶段有 Dev-QA Loop 自愈，不需要人工介入。**

### 7.3 总耗时

- **约 30-90 分钟**（取决于项目规模）
- 仅需 2 次人工确认

---

## 八、核心价值总结

### 8.1 个人效能的跃升

```
传统开发模式：一个人 = 一个开发者

AI 辅助模式：一个人 + AI = 1.5 个开发者

AI Agent 团队模式：一个人 + 12 个 Agent = 团队级的战斗力
```

### 8.2 核心价值

| 价值 | 说明 |
|------|------|
| **专业分工** | 12 个 Agent 各司其职，职责清晰 |
| **质量闭环** | Dev-QA Loop 确保每个任务都有独立验证 |
| **可控交付** | 明确的阶段划分和人工检查点 |
| **实战经验** | 每条零容忍规则背后都是真实事故教训 |

### 8.3 适用场景

**适合**：
- ✅ 中型应用开发（前后端 + DB + 部署）
- ✅ 需要质量保障的项目
- ✅ 独立开发者、小型团队
- ✅ 想要提升个人产出的工程师

**不适合**：
- ❌ 小工具、单文件脚本
- ❌ 纯前端原型
- ❌ 一次性小工具

### 8.4 经验教训

| 事故 | 规则 | 教训 |
|------|------|------|
| 子路径部署全部 404 | API 路径禁止硬编码 | 部署环境多变，必须用环境变量 |
| 后端改字段前端不知道 | 契约变更必须同步前端 | 改契约是大事，必须广播 |
| AI 自己测说 OK 实际有 bug | QA Agent 必须独立 | 自己测自己是靠不住的 |
| 重试无限循环 | 打回有上限 | 死循环浪费资源，必须有人工介入 |

**核心教训**：质量 > 速度，宁可慢一点串行跑，也不要让两个 agent 同时改代码。

---

## 参考资料

- [Claude Standard Dev Team - GitHub](https://github.com/xuanbingbingo/claude-standard-dev-team)
- [Claude Code 官方文档](https://claude.com/claude-code)
- [Claude Code Agent 机制](https://docs.claude.com/en/docs/claude-code/sub-agents)

---

> 本分享稿基于 [Claude Standard Dev Team](https://github.com/xuanbingbingo/claude-standard-dev-team) 项目整理。
> 技术细节请参考原文档：[OPC-技术分享.md](./OPC-技术分享.md)
