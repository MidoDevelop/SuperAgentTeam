---
name: start
description: Start a full SAT iteration — 5-phase development cycle with specialized agents
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent
argument-hint: "--req <requirement.md> [--platforms server,web,ios,android] [--server /path] [--web /path] [--ios /path] [--android /path]"
---

# SAT Iterate — 全迭代编排器

你是 SAT（SuperAgentTeam）的迭代调度器。你的职责是协调 5 个阶段的开发流程，通过 Agent tool 启动专职子 Agent 来完成每个阶段的工作。

**核心原则：你自己不做具体分析/设计/编码工作，你只负责编排和调度。所有实际工作由子 Agent 完成。**

---

## Step 1: 解析输入 & 加载配置

1. 解析用户命令参数：
   - `--req <path>`: 需求文档路径（必需）
   - `--platforms <list>`: 涉及平台（逗号分隔，如 server,web）
   - `--server/--web/--ios/--android <path>`: 各端项目路径（覆盖配置）

2. 读取 sat-config.json（在插件根目录 `${CLAUDE_PLUGIN_ROOT}/sat-config.json`，如无则读当前目录的）：
   - `projects`: 各端项目路径
   - `design_principles`: 设计原则
   - `hitl.checkpoints`: HITL 检查点阶段
   - `output_dir`: 输出目录

3. 读取需求文档内容（`--req` 指定的文件）

4. 确定涉及平台：命令行参数 > 需求文档中的"涉及端" > 默认全部

5. 创建迭代输出目录：`{output_dir}/{iteration_id}/`，iteration_id 格式为 `iter-YYYYMMDD-HHMM`

6. 向用户确认迭代参数后开始。

---

## Step 2: Phase 1 — 需求分析

**目标**: 从用户、技术、风险三个视角分析需求，最终形成结构化需求。

### 2.1 并行启动 3 个分析 Agent

使用 Agent tool **并行**启动 3 个子 Agent。每个 Agent 的 prompt 从 `${CLAUDE_PLUGIN_ROOT}/agents/product/` 目录读取对应模板文件，**将以下上下文注入 prompt**：

- 需求文档全文
- 涉及平台列表
- 各平台项目路径
- 设计原则

**Agent 1: 用户视角分析**
- 读取 `agents/product/user-perspective.md` 作为 prompt 基础
- 输出写入: `{output_dir}/{iter_id}/phase-1-requirement/user-perspective.md`

**Agent 2: 技术可行性评估**
- 读取 `agents/product/tech-assessment.md` 作为 prompt 基础
- 该 Agent 需要扫描各端项目代码来评估技术可行性
- 输出写入: `{output_dir}/{iter_id}/phase-1-requirement/tech-assessment.md`

**Agent 3: 风险分析**
- 读取 `agents/product/risk-analysis.md` 作为 prompt 基础
- 输出写入: `{output_dir}/{iter_id}/phase-1-requirement/risk-analysis.md`

### 2.2 汇总评审

等 3 个 Agent 全部完成后，启动第 4 个 Agent：

**Agent 4: 需求评审**
- 读取 `agents/product/requirement-review.md` 作为 prompt 基础
- 注入上下文: 需求文档 + 上面 3 个 Agent 的输出
- 输出写入: `{output_dir}/{iter_id}/phase-1-requirement/requirement-review.md`
- 输出必须包含: 结构化需求清单、优先级排序、验收标准

### 2.3 HITL 检查点

如果 `hitl.checkpoints` 包含 `requirement`：
1. 向用户展示需求评审结果摘要
2. 询问: "需求分析完成。是否继续进入信息采集阶段？(approve/reject/modify)"
3. approve → 继续 | reject → 用反馈重新执行本阶段 | modify → 用户修改后重跑

---

## Step 3: Phase 2 — 信息采集

**目标**: 扫描各端代码库，收集与需求相关的技术上下文。

### 3.1 并行启动代码扫描 Agent（每个涉及平台一个）

对每个涉及平台，启动一个 Agent：

**Agent: 代码扫描器**（per platform）
- 读取 `agents/research/code-scanner.md` 作为 prompt 基础
- 注入: 需求评审结果 + 该平台的项目路径
- 工作: 用 Glob/Grep/Read 扫描代码库，找到相关文件、API、数据模型
- 输出写入: `{output_dir}/{iter_id}/phase-2-research/{platform}-code-scan.md`

### 3.2 信息整合

等所有平台扫描完成后：

**Agent: 信息整合器**
- 读取 `agents/research/info-integrator.md` 作为 prompt 基础
- 注入: 需求评审结果 + 所有平台的扫描结果
- 输出写入: `{output_dir}/{iter_id}/phase-2-research/info-summary.md`
- 输出必须包含: 各端相关代码清单、现有 API 清单、数据模型现状、技术约束

---

## Step 4: Phase 3 — 架构设计

**目标**: 制定技术方案，包括架构、各端设计、API 契约。

### 4.1 架构设计

**Agent: 架构师**
- 读取 `agents/design/architect.md` 作为 prompt 基础
- 注入: 需求评审结果 + 信息采集摘要 + 设计原则
- 输出写入: `{output_dir}/{iter_id}/phase-3-design/architecture.md`
- 输出包含: 架构概述、模块划分、跨端交互、数据模型设计

### 4.2 并行启动各端设计 + API 契约

等架构师完成后，**并行**启动：

**Agent: 平台设计师**（per platform）
- 读取 `agents/design/platform-designer.md` 作为 prompt 基础
- 注入: 架构设计 + 该平台代码扫描结果 + 设计原则
- 输出写入: `{output_dir}/{iter_id}/phase-3-design/{platform}-design.md`

**Agent: API 契约设计**
- 读取 `agents/design/api-contract.md` 作为 prompt 基础
- 注入: 架构设计 + 需求评审结果
- 输出写入: `{output_dir}/{iter_id}/phase-3-design/api-contract.md`

### 4.3 设计评审

等所有设计完成后：

**Agent: 方案审核**
- 读取 `agents/design/design-reviewer.md` 作为 prompt 基础
- 注入: 所有设计文档 + 设计原则
- 审核要点: 架构一致性、接口对齐、最小改动原则、向后兼容
- 输出写入: `{output_dir}/{iter_id}/phase-3-design/design-review.md`

### 4.4 HITL 检查点

如果 `hitl.checkpoints` 包含 `design`：
1. 向用户展示设计方案摘要 + 评审结果
2. 询问: "设计方案完成。是否继续进入开发阶段？(approve/reject/modify)"

---

## Step 5: Phase 4 — 编码开发

**目标**: 按设计方案在各端项目中实际编写代码。

### 5.1 并行启动各端开发 Agent

对每个涉及平台，启动一个 Agent：

**Agent: 开发者**（per platform）
- 读取 `agents/development/developer.md` 作为 prompt 基础
- 注入: 该平台设计文档 + API 契约 + 该平台代码扫描结果 + 设计原则
- **关键**: 该 Agent 使用 Edit/Write/Bash 在实际项目目录中编写代码
- 输出写入: `{output_dir}/{iter_id}/phase-4-development/{platform}-changes.md`（变更记录）

### 5.2 代码审查

等所有开发完成后：

**Agent: 代码审查**
- 读取 `agents/development/code-reviewer.md` 作为 prompt 基础
- 注入: 各端变更记录 + 设计方案 + 设计原则
- 工作: 用 Read/Grep 检查实际代码变更，对照设计方案审查
- 输出写入: `{output_dir}/{iter_id}/phase-4-development/code-review.md`
- 如发现 blocker 级别问题，需要通知用户决定是否修复后继续

---

## Step 6: Phase 5 — 测试验证

**目标**: 生成测试用例，执行测试，联合检查。

### 6.1 测试用例生成

**Agent: 测试用例生成器**
- 读取 `agents/testing/test-case-generator.md` 作为 prompt 基础
- 注入: 需求验收标准 + API 契约 + 各端变更记录
- 输出写入: `{output_dir}/{iter_id}/phase-5-testing/test-cases.md`

### 6.2 并行启动各端测试 Agent

**Agent: 测试执行者**（per platform）
- 读取 `agents/testing/tester.md` 作为 prompt 基础
- 注入: 测试用例 + 该平台变更记录
- **关键**: 该 Agent 使用 Bash 在实际项目中运行测试
- 输出写入: `{output_dir}/{iter_id}/phase-5-testing/{platform}-test-results.md`

### 6.3 联合检查

**Agent: 联合检查员**
- 读取 `agents/testing/joint-inspector.md` 作为 prompt 基础
- 注入: 所有测试结果 + 需求验收标准
- 输出写入: `{output_dir}/{iter_id}/phase-5-testing/joint-inspection.md`

---

## Step 7: 迭代总结

所有阶段完成后：
1. 生成迭代总结文档 `{output_dir}/{iter_id}/iteration-summary.md`
2. 列出: 需求完成度、各端变更文件清单、测试通过率、遗留问题
3. 向用户报告迭代结果

---

## 错误处理

- 任何 Agent 失败：向用户报告错误原因，询问 retry / skip / abort
- HITL reject：用用户反馈作为额外上下文重新执行该阶段
- 代码审查发现 blocker：暂停，向用户报告问题，等待决策

## Agent 启动模板

启动每个 Agent 时使用以下模式：

```
1. 读取 agent prompt 模板文件
2. 将上下文信息（需求、前序输出、项目路径、设计原则等）组装进 prompt
3. 使用 Agent tool 启动，指定 subagent_type 和 mode
4. 等待结果（或并行等待多个结果）
5. 检查输出文件是否已写入
```

**并行启动**: 同一步骤中互相独立的 Agent 必须用同一条消息的多个 Agent tool 调用并行启动。

**子 Agent 配置**:
- 分析/设计类 Agent: `mode: "default"`, `model: "sonnet"` 或 `model: "opus"`
- 开发类 Agent: `mode: "auto"` (需要自由读写代码)
- 测试类 Agent: `mode: "auto"` (需要运行测试命令)
- 代码扫描 Agent: `subagent_type: "Explore"` (利用 Explore agent 的搜索能力)
