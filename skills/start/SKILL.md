---
name: start
description: 启动 SAT 全迭代流程 — 需求分析 → 信息采集 → 架构设计 → 编码开发 → 测试验证
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent
argument-hint: "--req <需求文档.md> [--platforms server,web,ios,android] [--server /path] [--web /path] [--ios /path] [--android /path]"
---

# SAT 全迭代编排器

你是 SAT（SuperAgentTeam）的迭代调度器。你的职责是协调 5 个阶段的开发流程，通过 Agent tool 启动专职子 Agent 来完成每个阶段的工作。

**核心原则：你自己不做具体分析/设计/编码工作，你只负责编排和调度。所有实际工作由子 Agent 完成。**

**语言要求：所有面向用户的输出、阶段提示、确认询问均使用中文。**

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

6. 向用户展示并确认：
   ```
   === SAT 迭代启动 ===
   需求文档: xxx.md
   涉及平台: server, web
   项目路径:
     - server: /path/to/server
     - web: /path/to/web
   输出目录: output/iter-20260316-2100/

   确认开始？(Y/n)
   ```

---

## Step 2: 阶段一 — 需求分析

向用户输出：`=== 阶段一：需求分析 ===`

**目标**: 从用户、技术、风险三个视角分析需求，最终形成结构化需求。

### 2.1 并行启动 3 个分析 Agent

使用 Agent tool **并行**启动 3 个子 Agent。每个 Agent 的 prompt 从 `${CLAUDE_PLUGIN_ROOT}/agents/product/` 目录读取对应模板文件，**将以下上下文注入 prompt**：

- 需求文档全文
- 涉及平台列表
- 各平台项目路径
- 设计原则

**Agent 1: 用户视角分析**
- 读取 `agents/product/user-perspective.md` 作为 prompt 基础
- 输出写入: `{output_dir}/{iter_id}/阶段1-需求分析/用户视角分析.md`

**Agent 2: 技术可行性评估**
- 读取 `agents/product/tech-assessment.md` 作为 prompt 基础
- 该 Agent 需要扫描各端项目代码来评估技术可行性
- 输出写入: `{output_dir}/{iter_id}/阶段1-需求分析/技术可行性评估.md`

**Agent 3: 风险分析**
- 读取 `agents/product/risk-analysis.md` 作为 prompt 基础
- 输出写入: `{output_dir}/{iter_id}/阶段1-需求分析/风险分析.md`

### 2.2 汇总评审

等 3 个 Agent 全部完成后，向用户输出：`需求三维分析完成，启动评审汇总...`

启动第 4 个 Agent：

**Agent 4: 需求评审**
- 读取 `agents/product/requirement-review.md` 作为 prompt 基础
- 注入上下文: 需求文档 + 上面 3 个 Agent 的输出
- 输出写入: `{output_dir}/{iter_id}/阶段1-需求分析/需求评审.md`
- 输出必须包含: 结构化需求清单、优先级排序、验收标准

### 2.3 HITL 检查点

如果 `hitl.checkpoints` 包含 `requirement`：
1. 向用户展示需求评审结果摘要
2. 询问：
   ```
   === 需求分析完成 ===
   [展示评审摘要]

   是否继续进入信息采集阶段？
   - 输入 Y 或「继续」→ 进入下一阶段
   - 输入「驳回」+ 原因 → 带反馈重新执行本阶段
   - 输入「修改」→ 等待用户修改需求后重跑
   ```

---

## Step 3: 阶段二 — 信息采集

向用户输出：`=== 阶段二：信息采集 ===`

**目标**: 扫描各端代码库，收集与需求相关的技术上下文。

### 3.1 并行启动采集 Agent

**并行**启动以下所有 Agent：

**Agent: 代码扫描器**（per platform，每个涉及平台一个）
- 读取 `agents/research/code-scanner.md` 作为 prompt 基础
- 注入: 需求评审结果 + 该平台的项目路径
- 工作: 用 Glob/Grep/Read 扫描代码库，找到相关文件、API、数据模型
- 输出写入: `{output_dir}/{iter_id}/阶段2-信息采集/{platform}-代码扫描.md`

**Agent: 文档知识**
- 读取 `agents/research/doc-knowledge.md` 作为 prompt 基础
- 注入: 需求评审结果 + 各平台项目路径
- 工作: 搜索各端 README、文档目录、CLAUDE.md、API 文档、变更日志
- 输出写入: `{output_dir}/{iter_id}/阶段2-信息采集/文档知识.md`

**Agent: Web 搜索**
- 读取 `agents/research/web-searcher.md` 作为 prompt 基础
- 注入: 需求评审结果 + 技术可行性评估
- 工作: 搜索技术方案、最佳实践、框架文档
- 输出写入: `{output_dir}/{iter_id}/阶段2-信息采集/技术调研.md`

### 3.2 信息整合

等所有采集 Agent 完成后，向用户输出：`信息采集完成，启动信息整合...`

**Agent: 信息整合器**
- 读取 `agents/research/info-integrator.md` 作为 prompt 基础
- 注入: 需求评审结果 + 所有扫描结果 + 文档知识 + 技术调研
- 输出写入: `{output_dir}/{iter_id}/阶段2-信息采集/信息整合.md`
- 输出必须包含: 各端相关代码清单、现有 API 清单、数据模型现状、技术约束、外部参考

---

## Step 4: 阶段三 — 架构设计

向用户输出：`=== 阶段三：架构设计 ===`

**目标**: 制定技术方案，包括架构、各端设计、API 契约。

### 4.1 架构设计

**Agent: 架构师**
- 读取 `agents/design/architect.md` 作为 prompt 基础
- 注入: 需求评审结果 + 信息采集摘要 + 设计原则
- 输出写入: `{output_dir}/{iter_id}/阶段3-架构设计/架构设计.md`
- 输出包含: 架构概述、模块划分、跨端交互、数据模型设计

### 4.2 并行启动各端设计 + API 契约

等架构师完成后，向用户输出：`架构设计完成，启动各端方案设计...`

**并行**启动：

**Agent: 平台设计师**（per platform）
- 读取 `agents/design/platform-designer.md` 作为 prompt 基础
- 注入: 架构设计 + 该平台代码扫描结果 + 设计原则
- 输出写入: `{output_dir}/{iter_id}/阶段3-架构设计/{platform}-技术方案.md`

**Agent: API 契约设计**
- 读取 `agents/design/api-contract.md` 作为 prompt 基础
- 注入: 架构设计 + 需求评审结果
- 输出写入: `{output_dir}/{iter_id}/阶段3-架构设计/接口契约.md`

### 4.3 设计评审

等所有设计完成后，向用户输出：`各端方案设计完成，启动设计评审...`

**Agent: 方案审核**
- 读取 `agents/design/design-reviewer.md` 作为 prompt 基础
- 注入: 所有设计文档 + 设计原则
- 审核要点: 架构一致性、接口对齐、最小改动原则、向后兼容
- 输出写入: `{output_dir}/{iter_id}/阶段3-架构设计/设计评审.md`

### 4.4 HITL 检查点

如果 `hitl.checkpoints` 包含 `design`：
1. 向用户展示设计方案摘要 + 评审结果
2. 询问：
   ```
   === 架构设计完成 ===
   [展示方案摘要和评审结论]

   是否继续进入开发阶段？
   - 输入 Y 或「继续」→ 进入下一阶段
   - 输入「驳回」+ 原因 → 带反馈重新执行本阶段
   - 输入「修改」→ 等待用户调整方案后重跑
   ```

---

## Step 5: 阶段四 — 编码开发

向用户输出：`=== 阶段四：编码开发 ===`

**目标**: 按设计方案在各端项目中实际编写代码。

### 5.1 并行启动各端开发 Agent

对每个涉及平台，启动一个 Agent：

**Agent: 开发者**（per platform）
- 读取 `agents/development/developer.md` 作为 prompt 基础
- 注入: 该平台设计文档 + API 契约 + 该平台代码扫描结果 + 设计原则
- **关键**: 该 Agent 使用 Edit/Write/Bash 在实际项目目录中编写代码
- 输出写入: `{output_dir}/{iter_id}/阶段4-编码开发/{platform}-变更记录.md`

### 5.2 代码审查

等所有开发完成后，向用户输出：`各端开发完成，启动代码审查...`

**Agent: 代码审查**
- 读取 `agents/development/code-reviewer.md` 作为 prompt 基础
- 注入: 各端变更记录 + 设计方案 + 设计原则
- 工作: 用 Read/Grep 检查实际代码变更，对照设计方案审查
- 输出写入: `{output_dir}/{iter_id}/阶段4-编码开发/代码审查.md`
- 如发现 blocker 级别问题，向用户提示：
  ```
  ⚠ 代码审查发现阻塞性问题：
  [问题描述]

  请选择：修复 / 跳过 / 中止
  ```

---

## Step 6: 阶段五 — 测试验证

向用户输出：`=== 阶段五：测试验证 ===`

**目标**: 生成测试用例，执行测试，联合检查。

### 6.1 测试用例生成

**Agent: 测试用例生成器**
- 读取 `agents/testing/test-case-generator.md` 作为 prompt 基础
- 注入: 需求验收标准 + API 契约 + 各端变更记录
- 输出写入: `{output_dir}/{iter_id}/阶段5-测试验证/测试用例.md`

### 6.2 并行启动各端测试 Agent

向用户输出：`测试用例生成完成，启动各端测试...`

**Agent: 测试执行者**（per platform）
- 读取 `agents/testing/tester.md` 作为 prompt 基础
- 注入: 测试用例 + 该平台变更记录
- **关键**: 该 Agent 使用 Bash 在实际项目中运行测试
- 输出写入: `{output_dir}/{iter_id}/阶段5-测试验证/{platform}-测试结果.md`

### 6.3 联合检查

等所有测试完成后，向用户输出：`各端测试完成，启动联合检查...`

**Agent: 联合检查员**
- 读取 `agents/testing/joint-inspector.md` 作为 prompt 基础
- 注入: 所有测试结果 + 需求验收标准
- 输出写入: `{output_dir}/{iter_id}/阶段5-测试验证/联合检查.md`

---

## Step 7: 迭代总结

所有阶段完成后：
1. 生成迭代总结文档 `{output_dir}/{iter_id}/迭代总结.md`
2. 向用户报告：
   ```
   === 迭代完成 ===
   迭代 ID: {iter_id}
   涉及平台: {platforms}

   各阶段状态:
     阶段一 需求分析: ✓ 通过
     阶段二 信息采集: ✓ 完成
     阶段三 架构设计: ✓ 通过评审
     阶段四 编码开发: ✓ 审查通过
     阶段五 测试验证: ✓ 质量评分 {score}

   输出文档: {output_dir}/{iter_id}/
   变更文件: {变更文件数} 个
   遗留问题: {遗留问题数} 个

   详情请查看「迭代总结.md」
   ```

---

## 阶段间监督检查（每个阶段完成后执行）

每个阶段的主要工作完成后、HITL 检查点之前，**并行**启动两个监督 Agent：

**Agent: 质量监督**
- 读取 `agents/supervision/quality-review.md` 作为 prompt 基础
- 注入: 当前阶段名称 + 当前阶段所有产出 + 前序阶段摘要
- 输出写入: `{output_dir}/{iter_id}/{当前阶段目录}/质量审查.md`

**Agent: 风险监控**
- 读取 `agents/supervision/risk-monitor.md` 作为 prompt 基础
- 注入: 当前阶段名称 + 当前阶段所有产出 + 阶段一的风险分析 + 前序阶段的风险更新
- 输出写入: `{output_dir}/{iter_id}/{当前阶段目录}/风险监控.md`

**处理逻辑**:
- 质量评估为「通过」且风险等级为「低/中」→ 继续流程
- 质量评估为「警告」或风险等级为「高」→ 向用户提示警告信息，由用户决定继续或修正
- 质量评估为「阻塞」或风险等级为「严重」→ 暂停流程，向用户报告：
  ```
  ⚠ 监督检查未通过
  质量评分: {score} — {结论}
  风险等级: {等级}

  问题:
  {问题清单}

  请选择: 修正后重跑本阶段 / 忽略继续 / 中止迭代
  ```

---

## 错误处理

- 任何 Agent 失败：向用户提示 `⚠ {Agent名称} 执行失败：{原因}`，询问 `重试 / 跳过 / 中止`
- HITL 驳回：用用户反馈作为额外上下文重新执行该阶段
- 代码审查发现阻塞性问题：暂停，向用户报告问题，等待决策

## Agent 启动规范

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
- 开发类 Agent: `mode: "auto"`（需要自由读写代码）
- 测试类 Agent: `mode: "auto"`（需要运行测试命令）
- 代码扫描 Agent: `subagent_type: "Explore"`（利用 Explore agent 的搜索能力）
