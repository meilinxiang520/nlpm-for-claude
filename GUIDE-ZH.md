# NLPM 中文分析与使用指南

> Natural-Language Programming Manager — 自然语言编程质量管理器

---

## 一、项目定位

**NLPM = 自然语言编程的 ESLint**

传统 linter（ESLint、ruff、clippy）检查的是代码文件。NLPM 检查的是驱动 AI 行为的 Markdown 文件：技能（SKILL.md）、Agent 定义、命令、规则（.claude/rules/）、Hooks、提示词、CLAUDE.md、记忆文件。

核心洞察：**这些 Markdown 文件本质上是程序。** 写得好，AI 行为稳定可预测；写得差（模糊词泛滥、缺少示例、工具权限过大），AI 输出就飘忽不定。

NLPM 的工作就是像 linter 一样，把"NL 程序的坏味道"量化成扣分，给出可操作的修复建议。

---

## 二、技术架构

```
用户命令层（7 个命令）
        ↓ 调度
Agent 层（5 个专职 Agent）
  scanner      — haiku  — 机械式文件发现
  vague-scanner — haiku  — 机械式模糊词计数
  scorer       — sonnet — 100 分质量评分
  checker      — sonnet — 跨组件一致性检查
  tester       — sonnet — NL-TDD 规格测试
        ↑ 引用
Skills 层（12 个知识库）
  核心：rules / scoring / conventions / patterns / testing / security
  写作参考（按需加载）：writing-skills / writing-agents / writing-hooks /
                        writing-rules / writing-prompts / writing-plugins /
                        orchestration
```

**零依赖，纯 Markdown 插件。** 无需 Python、Node.js 或外部 API Key，安装即用。

---

## 三、50 条规则体系

NLPM 的评分完全由 50 条明确规则驱动，分布在 8 个领域：

### Universal（R01-R03）——适用所有制品

| 规则 | 核心要求 |
|------|---------|
| **R01** | 禁止无标准的模糊量词：appropriate / relevant / as needed / sufficient / properly / correctly 等，每个 -2 分，上限 -20 |
| **R02** | 每一行必须改变 Claude 的行为，否则删除（token 有限） |
| **R03** | 正向措辞："用 X" 而不是 "不要用 Y"（Pink Elephant 效应） |

### Skills（R04-R08）

| 规则 | 核心要求 |
|------|---------|
| **R04** | `description` 是触发器而非摘要，需含 3+ 具体动作短语 |
| **R05** | 单文件 500 行上限，超出即拆分子技能 |
| **R06** | 代码示例必须可运行，非伪代码 |
| **R07** | 有关联技能时必须写 Scope 说明并交叉引用 |
| **R08** | 模式优先于理论，用具体场景教，不讲抽象概念 |

### Agents（R09-R13）

| 规则 | 核心要求 |
|------|---------|
| **R09** | `<example>` 块强制要求，最少 2 个，格式：Context + user + assistant |
| **R10** | 模型选型与任务复杂度匹配：haiku=机械、sonnet=推理、opus=复杂判断 |
| **R11** | 工具最小权限：只声明 body 中实际用到的工具 |
| **R12** | 必须在 body 中定义输出格式 |
| **R13** | 系统提示结构：任务 → 步骤 → 边界 → 输出格式 |

### Commands（R14-R18）

| 规则 | 核心要求 |
|------|---------|
| **R14** | 多步骤工作流必须编号 |
| **R15** | 必须处理空输入（`$ARGUMENTS` 为空时的默认行为） |
| **R16** | 定义输出格式，不能写"展示结果" |
| **R17** | 指定所有错误路径（文件不存在、数据异常等） |
| **R18** | 接受输入的命令必须有 `argument-hint` |

### Rules 文件（R21-R26）

| 规则 | 核心要求 |
|------|---------|
| **R21** | 格式：**粗体命令式** + 不遵守会怎样 + 为什么 |
| **R22** | 必须可执行：能在代码审查中验证 |
| **R23** | 所有规则文件合计 <500 行 |
| **R24** | 不重复工具链已覆盖的检查（引用 eslint/ruff 而不是重写） |
| **R25** | 尽量设置 `paths:` 路径限定范围 |
| **R26** | 同一文件内不同规则不得相互矛盾 |

### Hooks（R27-R32）

| 规则 | 核心要求 |
|------|---------|
| **R27** | 事件名大小写敏感：`PreToolUse` 而非 `pretooluse` |
| **R28** | `type: command` 配 `command:` 字段，`type: prompt` 配 `prompt:` |
| **R29** | 引用的脚本必须存在 |
| **R30** | 用 `${CLAUDE_PLUGIN_ROOT}` 而非硬编码绝对路径 |
| **R31** | 默认失败开放（fail-open），只有关键安全门才失败关闭 |
| **R32** | PreToolUse 可阻止动作；PostToolUse 只能建议 |

### CLAUDE.md（R33-R39）

| 规则 | 核心要求 |
|------|---------|
| **R33** | 必须包含构建命令 |
| **R34** | 必须包含测试命令 |
| **R35** | 必须包含架构概览（目录用途表） |
| **R36** | `@path` 引入必须指向真实存在的文件 |
| **R37** | 不得有过期引用（已删除的文件/函数/API） |
| **R38** | 指令性内容 >60%（CLAUDE.md 是给 Claude 看的，不是 README） |
| **R39** | 与 `.claude/rules/` 无冲突 |

### Orchestration（R43-R47）

| 规则 | 核心要求 |
|------|---------|
| **R43** | 无数据依赖时并行，有依赖时串行 |
| **R44** | AI 输出经 QC 门控后再呈现给用户 |
| **R45** | 高成本 AI 阶段前必须估算并确认 token 费用 |
| **R46** | 用状态文件支持中断恢复（pending → running → completed/failed） |
| **R47** | 循环必须有最大重试次数（通常 3 次） |

---

## 四、自进化机制

`auditor/` 目录内置 6 个 GitHub Actions，构成一条自动进化流水线：

```
每周自动发现 (500+ star 的 Claude 插件仓库)
        ↓
安全扫描 (Critical 风险直接 block)
        ↓
NL 质量审计 (nlpm:score)
        ↓
自动提 PR (只修复已验证的 bug)
        ↓
追踪 PR 合并率
        ↓
写入 feedback/log.json
        ↓
驱动 NLPM 自身规则迭代 → 审计更精准
```

三个人工决策点（issue label）：`audit-ready` → `contribute-approved` → `case-study-ready`，既自动化又保留人类监督。

---

## 五、安装

```bash
# 对所有项目生效（推荐）
claude plugin install nlpm@xiaolai --scope user

# 只对当前项目生效
claude plugin install nlpm@xiaolai --scope project
```

---

## 六、命令速查

| 命令 | 参数 | 用途 |
|------|------|------|
| `/nlpm:ls` | 无 | 发现并列出所有 NL 制品 |
| `/nlpm:score` | `[路径]` 或 `--changed` | 100 分质量评分 |
| `/nlpm:check` | 无 | 跨组件一致性：断链/孤儿/矛盾 |
| `/nlpm:fix` | 无 | 自动修复机械性问题 |
| `/nlpm:trend` | 无 | 历史评分趋势（发现悄悄退步） |
| `/nlpm:test` | 无 | 运行 `.nlpm-test/` 中的 NL-TDD 规格 |
| `/nlpm:init` | 无 | 初始化 `.claude/nlpm.local.md` 配置 |

---

## 七、典型工作流

### 接手新项目

```
/nlpm:ls              # 摸底：有哪些 NL 制品
/nlpm:score           # 全量评分，找最低分文件
/nlpm:check           # 检查断链和矛盾
/nlpm:fix             # 自动修机械问题
```

### 写新 Agent

```
1. 先写规格文件：.nlpm-test/my-agent.spec.md
2. /nlpm:test         → RED（制品不存在）
3. 写 agents/my-agent.md
4. /nlpm:test         → 检查触发准确率、输出格式
5. /nlpm:score agents/my-agent.md
6. /nlpm:fix          → 修机械问题
7. 手动优化描述和示例，目标 80+
```

### 发版前检查

```
/nlpm:score --changed   # 只评分本次改动的文件
/nlpm:check             # 发现断链
/nlpm:trend             # 检测有没有文件悄悄退步
```

---

## 八、评分体系

分数从 100 开始，按问题扣减，下限 0，确定性计算（同一制品永远同分）。

| 分数段 | 等级 | 含义 |
|--------|------|------|
| 90-100 | Excellent | 生产就绪 |
| 80-89 | Good | 有小缺口 |
| 70-79 | Adequate | 达标但应改进 |
| 60-69 | Weak | 低于阈值 |
| <60 | Rewrite | 根本性问题 |

默认阈值 70，可配置。**目标分数建议 85+，不要追求 100**（最后 5-10 分收益递减）。

---

## 九、最高频失分项

### R01 模糊词（最常见，-2/个，上限 -20）

下列词出现即扣分：

`appropriate` `relevant` `as needed` `sufficient` `adequate` `reasonable`
`properly` `correctly` `some` `several` `various` `thoroughly` `gracefully`

**替换示例：**

```
❌ "Use appropriate error handling."
✅ "Return Result<T, AppError> from all API handlers.
    Map errors to HTTP status codes via From<AppError> for StatusCode."

❌ "Handle edge cases properly."
✅ "If input is empty string, return Err(ValidationError::EmptyInput).
    If input exceeds 1000 chars, return Err(ValidationError::TooLong)."
```

### R09 缺少 Example 块（-15）

Agent 没有 `<example>` 块是最大单项扣分。每个 example 必须含：
- **Context**：用户正在做什么
- **user**：用户说的话
- **assistant**：Agent 的回应（包含输出格式预览）

### R04 描述是摘要不是触发器（-15）

```
❌ description: "Helpful React skill for component development"
✅ description: "Use when debugging React re-renders, fixing hook dependency
    arrays, optimizing with useMemo/useCallback, or implementing error boundaries."
```

---

## 十、配置

运行 `/nlpm:init` 生成，或手动创建 `.claude/nlpm.local.md`：

```yaml
---
strictness: standard        # relaxed(60) / standard(70) / strict(80)
score_threshold: 70
rule_overrides:
  R09: { min_examples: 1 }  # 只要求 1 个 example（宽松）
  R05: { threshold: 600 }   # skill 最多 600 行
  R23: { budget: 800 }      # 规则文件总预算 800 行
---
```

---

## 十一、故障排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 分数看起来太低 | 模糊词叠加很快 | 看具体扣分项，逐条处理 |
| Writing skill 没有触发 | 关键词不匹配 | 说"write an agent definition"或"create a new skill" |
| Check 报告孤儿文件但实际有用 | Writing skills 是按需加载，不被 agent 引用 | 预期行为，忽略这类孤儿 |
| Trend 无历史数据 | 未初始化 | 先跑 `/nlpm:trend` 创建基线快照 |

---

> 参考：`skills/nlpm/rules/SKILL.md`（50 条完整规则）、`EXAMPLES.md`（真实改进案例）、`auditor/README.md`（自进化流水线）
