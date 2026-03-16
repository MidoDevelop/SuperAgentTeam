# SuperAgentTeam

多 Agent 协作研发系统 — 基于 Claude Code Agent 能力的多端迭代开发框架。

**零 Python 运行时**。全部能力由 Claude Code 原生 Agent tool 驱动，每个 Agent 直接使用 Claude Code 的 Read/Write/Edit/Bash/Grep 操作真实代码。

## 工作原理

```
用户: /sat:start --req 需求文档.md --platforms server,web

SKILL.md (编排器) ─── Claude Code 自身作为调度器
    │
    ├── Agent tool → 用户视角分析 ──┐
    ├── Agent tool → 技术可行性评估 ──├── 并行
    ├── Agent tool → 风险分析 ──────┘
    │
    ├── Agent tool → 需求评审汇总
    │       ↓ HITL: 用户审批
    │
    ├── Agent tool → 代码扫描 (per 平台) ── 并行
    ├── Agent tool → 信息整合
    │
    ├── Agent tool → 架构设计
    ├── Agent tool → 各端方案 + API 契约 ── 并行
    ├── Agent tool → 设计评审
    │       ↓ HITL: 用户审批
    │
    ├── Agent tool → 各端开发 ── 并行，直接写代码
    ├── Agent tool → 代码审查
    │
    ├── Agent tool → 测试用例生成
    ├── Agent tool → 各端测试执行 ── 并行
    └── Agent tool → 联合检查 → 迭代总结
```

## 与 v1 的区别

| | v1 (Python 引擎) | v2 (Agent 原生) |
|--|---|---|
| 运行时 | 自研 Python Scheduler + WorkflowEngine | Claude Code 自身 |
| Agent 实现 | Python 类 + 自研 LLMProvider | Agent tool 子进程 |
| 工具调用 | 自研 ToolRegistry 代理 | Claude Code 原生工具 |
| 代码操作 | 间接（通过 LLM 生成代码文本） | 直接 Read/Write/Edit 真实文件 |
| 测试执行 | 模拟输出 | Bash 真实运行 |
| 文件数量 | ~50 个 Python 文件 | 19 个 Prompt 模板 |

## 快速开始

### 1. 安装插件

```bash
# 将本项目添加为本地 marketplace 后安装
claude plugin enable sat@local
```

### 2. 配置项目路径

编辑 `sat-config.json`：

```json
{
  "projects": {
    "server": "/path/to/server",
    "web": "/path/to/web"
  }
}
```

### 3. 执行迭代

```bash
/sat:start --req requirements/ITER-001.md --platforms server,web
```

## 目录结构

```
SuperAgentTeam/
├── .claude-plugin/plugin.json    # 插件元数据
├── sat-config.json               # 项目配置
├── skills/start/SKILL.md         # 编排器（唯一入口）
├── agents/                       # Agent 提示词模板
│   ├── product/                  # 需求分析团队 (4)
│   ├── research/                 # 信息采集团队 (4)
│   ├── design/                   # 架构设计团队 (4)
│   ├── development/              # 编码开发团队 (2)
│   ├── testing/                  # 测试验证团队 (3)
│   └── supervision/              # 监督团队 (2)
├── templates/requirement.md      # 需求文档模板
└── output/                       # 迭代输出目录
    └── iter-YYYYMMDD-HHMM/
        ├── 阶段1-需求分析/
        ├── 阶段2-信息采集/
        ├── 阶段3-架构设计/
        ├── 阶段4-编码开发/
        ├── 阶段5-测试验证/
        └── 迭代总结.md
```

## 设计原则

- **Claude Code 即运行时**: 不引入任何中间层，直接使用 CC 的 Agent/Read/Write/Bash
- **Prompt 即逻辑**: 编排逻辑在 SKILL.md 中，Agent 能力在提示词模板中
- **文件即通信**: Agent 间通过输出文件传递数据，无需消息总线
- **真实操作**: 开发 Agent 直接写代码，测试 Agent 直接跑测试

## License

MIT
