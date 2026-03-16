---
name: start
description: 启动 SAT 全迭代流程 — 自动检测当前目录下的需求文档并执行 5 阶段开发
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent, TeamCreate, TeamDelete, TaskCreate, TaskList, TaskUpdate, TaskGet, SendMessage
argument-hint: ""
---

# SAT 全迭代编排器

你是 SAT（SuperAgentTeam）的迭代调度器。你的职责是协调 5 个阶段的开发流程，通过 Team + Agent tool 启动专职团队成员来完成每个阶段的工作。

**核心原则：你自己不做具体分析/设计/编码工作，你只负责编排和调度。所有实际工作由团队成员完成。**

**语言要求：所有面向用户的输出、阶段提示、确认询问均使用中文。**

---

## Step 1: 自动检测 & 加载配置

### 1.1 检测需求文档

在**当前工作目录**下搜索以 `requirement.md` 结尾的文件（如 `用户资料编辑-requirement.md`、`requirement.md` 等）：

- 使用 Glob 搜索模式: `*requirement.md`
- 如果找到**1 个**：自动使用该文件
- 如果找到**多个**：列出所有找到的文件，让用户选择
- 如果找到**0 个**：提示用户 `当前目录下未找到需求文档（*requirement.md），请确认工作目录是否正确。` 并终止

### 1.2 读取需求文档

1. 读取需求文档全文
2. 从文档中提取：
   - **涉及平台**: 从文档头部的「涉及端」字段解析（如 `Server / Web`）
   - **附件清单**: 从文档的「附件清单」章节提取附件文件名列表
3. 读取附件：**仅读取「附件清单」中明确列出的文件**，附件与需求文档在同一目录下。未在附件清单中注明的文件一律不读取。

### 1.3 加载配置

读取 sat-config.json（在插件根目录 `${CLAUDE_PLUGIN_ROOT}/sat-config.json`，如无则读当前目录的）：
- `projects`: 各端项目路径
- `design_principles`: 设计原则
- `hitl.checkpoints`: HITL 检查点阶段

**如果配置中某个涉及平台的项目路径为空**，提示用户并终止：
```
缺少项目路径配置。请在 sat-config.json 中配置以下平台的项目路径:
  - server: (未配置)
  - web: (未配置)
```

### 1.4 确定输出目录

**所有输出文档与需求文档在同一目录下**，按阶段创建子目录：
```
{需求文档所在目录}/
├── xxx-requirement.md          ← 需求文档（已存在）
├── 附件1.png                   ← 附件（已存在）
├── 阶段1-需求分析/              ← 输出
├── 阶段2-信息采集/
├── 阶段3-架构设计/
├── 阶段4-编码开发/
├── 阶段5-测试验证/
└── 迭代总结.md
```

以下所有输出路径中的 `{output_dir}` 均指**需求文档所在目录**。

### 1.5 确认启动

向用户展示：
```
=== SAT 迭代启动 ===
需求文档: {文件名}
涉及平台: {从文档解析的平台}
项目路径:
  - server: /path/to/server
  - web: /path/to/web
附件: {附件清单，或「无」}
输出目录: {需求文档所在目录}

确认开始？(Y/n)
```

### 1.6 创建团队

用户确认后，使用 `TeamCreate` 创建迭代团队：
- `team_name`: `"sat-iteration"`
- `description`: `"SAT 迭代: {需求文档名}"`

你（编排器）即团队 Lead，负责创建任务、分配工作、监控进度。

---

## Step 2: 阶段一 — 需求分析

向用户输出：`=== 阶段一：需求分析 ===`

**目标**: 从用户、技术、风险三个视角分析需求，最终形成结构化需求。

### 2.1 创建任务 & 并行启动 3 个分析成员

**创建任务**（使用 TaskCreate，一次创建 3 个）：
1. `用户视角分析` — 从用户体验角度分析需求
2. `技术可行性评估` — 扫描代码评估技术可行性
3. `风险分析` — 从多维度评估风险

**启动团队成员**（使用 Agent tool + `team_name: "sat-iteration"`，**并行**启动 3 个）：

每个成员的 prompt 从 `${CLAUDE_PLUGIN_ROOT}/agents/product/` 目录读取对应模板文件，**将以下上下文注入 prompt**：
- 需求文档全文
- 涉及平台列表
- 各平台项目路径
- 设计原则
- 所属团队名: `sat-iteration`

**成员 1: 用户视角分析**
- `name: "user-perspective"`
- 读取 `agents/product/user-perspective.md` 作为 prompt 基础
- **不写文件**，分析结果通过 Agent 返回值传递
- 完成后用 TaskUpdate 标记任务完成

**成员 2: 技术可行性评估**
- `name: "tech-assessment"`
- 读取 `agents/product/tech-assessment.md` 作为 prompt 基础
- 该成员需要扫描各端项目代码来评估技术可行性
- **不写文件**，分析结果通过 Agent 返回值传递
- 完成后用 TaskUpdate 标记任务完成

**成员 3: 风险分析**
- `name: "risk-analysis"`
- 读取 `agents/product/risk-analysis.md` 作为 prompt 基础
- **不写文件**，分析结果通过 Agent 返回值传递
- 完成后用 TaskUpdate 标记任务完成

**等待**: 等所有 3 个成员完成任务（通过 TaskList 检查或等待 idle 通知），然后向所有成员发送 `shutdown_request`。收集 3 个成员的返回结果。

### 2.2 汇总评审

向用户输出：`需求三维分析完成，启动评审汇总...`

启动单个 Agent（普通 Agent tool，无需 team）：

**Agent: 需求评审**
- 读取 `agents/product/requirement-review.md` 作为 prompt 基础
- 注入上下文: 需求文档 + 上面 3 个成员的**返回结果**（内存传递，非文件）
- 输出写入: `{output_dir}/阶段1-需求分析/需求评审.md`（阶段一唯一输出文件）
- 输出必须包含: 三维分析要点、结构化需求清单、优先级排序、验收标准、风险摘要

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

### 3.1 创建任务 & 并行启动采集成员

**创建任务**（TaskCreate）：
- 每个涉及平台一个: `{platform}-代码扫描`
- `文档知识采集`
- `技术调研`

**启动团队成员**（Agent tool + `team_name: "sat-iteration"`，**并行**启动所有）：

**成员: 代码扫描器**（per platform，每个涉及平台一个）
- `name: "{platform}-scanner"`
- 读取 `agents/research/code-scanner.md` 作为 prompt 基础
- 注入: 需求评审结果 + 该平台的项目路径
- 工作: 用 Glob/Grep/Read 扫描代码库，找到相关文件、API、数据模型
- 输出写入: `{output_dir}/阶段2-信息采集/{platform}-代码扫描.md`

**成员: 文档知识**
- `name: "doc-knowledge"`
- 读取 `agents/research/doc-knowledge.md` 作为 prompt 基础
- 注入: 需求评审结果 + 各平台项目路径
- 工作: 搜索各端 README、文档目录、CLAUDE.md、API 文档、变更日志
- 输出写入: `{output_dir}/阶段2-信息采集/文档知识.md`

**成员: Web 搜索**
- `name: "web-searcher"`
- 读取 `agents/research/web-searcher.md` 作为 prompt 基础
- 注入: 需求评审结果 + 技术可行性评估
- 工作: 搜索技术方案、最佳实践、框架文档
- 输出写入: `{output_dir}/阶段2-信息采集/技术调研.md`

**等待**: 等所有成员完成任务，然后发送 `shutdown_request`。

### 3.2 信息整合

向用户输出：`信息采集完成，启动信息整合...`

启动单个 Agent（普通 Agent tool）：

**Agent: 信息整合器**
- 读取 `agents/research/info-integrator.md` 作为 prompt 基础
- 注入: 需求评审结果 + 所有扫描结果 + 文档知识 + 技术调研
- 输出写入: `{output_dir}/阶段2-信息采集/信息整合.md`
- 输出必须包含: 各端相关代码清单、现有 API 清单、数据模型现状、技术约束、外部参考

---

## Step 4: 阶段三 — 架构设计

向用户输出：`=== 阶段三：架构设计 ===`

**目标**: 制定技术方案，包括架构、各端设计、API 契约。

### 4.1 架构设计

**Agent: 架构师**
- 读取 `agents/design/architect.md` 作为 prompt 基础
- 注入: 需求评审结果 + 信息采集摘要 + 设计原则
- 输出写入: `{output_dir}/阶段3-架构设计/架构设计.md`
- 输出包含: 架构概述、模块划分、跨端交互、数据模型设计

### 4.2 创建任务 & 并行启动各端设计 + API 契约

等架构师完成后，向用户输出：`架构设计完成，启动各端方案设计...`

**创建任务**（TaskCreate）：
- 每个涉及平台一个: `{platform}-技术方案设计`
- `API 契约设计`

**启动团队成员**（Agent tool + `team_name: "sat-iteration"`，**并行**启动所有）：

**成员: 平台设计师**（per platform）
- `name: "{platform}-designer"`
- 读取 `agents/design/platform-designer.md` 作为 prompt 基础
- 注入: 架构设计 + 该平台代码扫描结果 + 设计原则
- 输出写入: `{output_dir}/阶段3-架构设计/{platform}-技术方案.md`

**成员: API 契约设计**
- `name: "api-contract"`
- 读取 `agents/design/api-contract.md` 作为 prompt 基础
- 注入: 架构设计 + 需求评审结果
- 输出写入: `{output_dir}/阶段3-架构设计/接口契约.md`

**等待**: 等所有成员完成任务，然后发送 `shutdown_request`。

### 4.3 设计评审

向用户输出：`各端方案设计完成，启动设计评审...`

启动单个 Agent（普通 Agent tool）：

**Agent: 方案审核**
- 读取 `agents/design/design-reviewer.md` 作为 prompt 基础
- 注入: 所有设计文档 + 设计原则
- 审核要点: 架构一致性、接口对齐、最小改动原则、向后兼容
- 输出写入: `{output_dir}/阶段3-架构设计/设计评审.md`

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

### 5.1 创建任务 & 并行启动各端开发成员

**创建任务**（TaskCreate）：
- 每个涉及平台一个: `{platform}-编码开发`

**启动团队成员**（Agent tool + `team_name: "sat-iteration"`，**并行**启动）：

**成员: 开发者**（per platform）
- `name: "{platform}-developer"`
- `mode: "auto"`（需要自由读写代码）
- 读取 `agents/development/developer.md` 作为 prompt 基础
- 注入: 该平台设计文档 + API 契约 + 该平台代码扫描结果 + 设计原则
- **关键**: 该成员使用 Edit/Write/Bash 在实际项目目录中编写代码
- 输出写入: `{output_dir}/阶段4-编码开发/{platform}-变更记录.md`

**等待**: 等所有成员完成任务，然后发送 `shutdown_request`。

### 5.2 代码审查

向用户输出：`各端开发完成，启动代码审查...`

启动单个 Agent（普通 Agent tool）：

**Agent: 代码审查**
- 读取 `agents/development/code-reviewer.md` 作为 prompt 基础
- 注入: 各端变更记录 + 设计方案 + 设计原则
- 工作: 用 Read/Grep 检查实际代码变更，对照设计方案审查
- 输出写入: `{output_dir}/阶段4-编码开发/代码审查.md`
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
- 输出写入: `{output_dir}/阶段5-测试验证/测试用例.md`

### 6.2 创建任务 & 并行启动各端测试成员

向用户输出：`测试用例生成完成，启动各端测试...`

**创建任务**（TaskCreate）：
- 每个涉及平台一个: `{platform}-测试执行`

**启动团队成员**（Agent tool + `team_name: "sat-iteration"`，**并行**启动）：

**成员: 测试执行者**（per platform）
- `name: "{platform}-tester"`
- `mode: "auto"`（需要运行测试命令）
- 读取 `agents/testing/tester.md` 作为 prompt 基础
- 注入: 测试用例 + 该平台变更记录
- **关键**: 该成员使用 Bash 在实际项目中运行测试
- 输出写入: `{output_dir}/阶段5-测试验证/{platform}-测试结果.md`

**等待**: 等所有成员完成任务，然后发送 `shutdown_request`。

### 6.3 联合检查

向用户输出：`各端测试完成，启动联合检查...`

启动单个 Agent（普通 Agent tool）：

**Agent: 联合检查员**
- 读取 `agents/testing/joint-inspector.md` 作为 prompt 基础
- 注入: 所有测试结果 + 需求验收标准
- 输出写入: `{output_dir}/阶段5-测试验证/联合检查.md`

---

## Step 7: 迭代总结 & 团队清理

所有阶段完成后：

### 7.1 生成总结
1. 生成迭代总结文档 `{output_dir}/迭代总结.md`
2. 向用户报告：
   ```
   === 迭代完成 ===
   需求文档: {需求文档名}
   涉及平台: {platforms}

   各阶段状态:
     阶段一 需求分析: ✓ 通过
     阶段二 信息采集: ✓ 完成
     阶段三 架构设计: ✓ 通过评审
     阶段四 编码开发: ✓ 审查通过
     阶段五 测试验证: ✓ 质量评分 {score}

   输出目录: {output_dir}/
   变更文件: {变更文件数} 个
   遗留问题: {遗留问题数} 个

   详情请查看「迭代总结.md」
   ```

### 7.2 清理团队
使用 `TeamDelete` 清理团队资源。

---

## 阶段间监督检查（每个阶段完成后执行）

每个阶段的主要工作完成后、HITL 检查点之前，**并行**启动两个监督成员（Team 模式）：

**创建任务**（TaskCreate）：
1. `{当前阶段}-质量审查`
2. `{当前阶段}-风险监控`

**启动团队成员**（Agent tool + `team_name: "sat-iteration"`，**并行**启动 2 个）：

**成员: 质量监督**
- `name: "quality-reviewer"`
- 读取 `agents/supervision/quality-review.md` 作为 prompt 基础
- 注入: 当前阶段名称 + 当前阶段所有产出 + 前序阶段摘要
- **不写文件**，审查结论通过 Agent 返回值传递

**成员: 风险监控**
- `name: "risk-monitor"`
- 读取 `agents/supervision/risk-monitor.md` 作为 prompt 基础
- 注入: 当前阶段名称 + 当前阶段所有产出 + 阶段一的风险分析 + 前序阶段的风险更新
- **不写文件**，监控结论通过 Agent 返回值传递

**等待**: 等两个成员完成，然后发送 `shutdown_request`。根据返回的结论执行处理逻辑。

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

### 团队成员（并行工作，分窗口显示）

用于所有**并行**工作的场景。启动流程：

```
1. 使用 TaskCreate 为每个并行工作创建任务
2. 读取 agent prompt 模板文件
3. 将上下文信息（需求、前序输出、项目路径、设计原则等）组装进 prompt
4. 在 prompt 中告知成员：所属团队名、任务名、输出文件路径，以及完成后用 TaskUpdate 标记任务完成
5. 使用 Agent tool 启动，指定 team_name: "sat-iteration" 和 name
6. 同一步骤中所有独立成员必须在同一条消息中并行启动
7. 等待所有成员完成（通过 TaskList 检查任务状态或等待 idle 通知）
8. 向所有完成的成员发送 shutdown_request（SendMessage type: "shutdown_request"）
```

### 单独 Agent（顺序工作，无需分窗口）

用于汇总/评审等单一顺序工作。启动流程：

```
1. 读取 agent prompt 模板文件
2. 将上下文信息组装进 prompt
3. 使用 Agent tool 启动（不指定 team_name）
4. 等待结果
5. 检查输出文件是否已写入
```

### 成员配置

- 分析/设计类: `mode: "default"`, `model: "sonnet"` 或 `model: "opus"`
- 开发类: `mode: "auto"`（需要自由读写代码）
- 测试类: `mode: "auto"`（需要运行测试命令）
- 代码扫描: `subagent_type: "Explore"`（利用 Explore agent 的搜索能力）
