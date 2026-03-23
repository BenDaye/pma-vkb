# pma-vkb

PMA（Project Management Agent）+ Vibe Kanban 编排插件 —— 为 AI 编码 Agent 提供结构化的项目交付流程管理。

## 简介

pma-vkb 是一个 [Claude Code 插件](https://docs.anthropic.com/en/docs/claude-code)，为 AI Agent 在现有代码库中执行功能开发、Bug 修复、重构等任务时提供**严格的三阶段工作流**（调查 → 方案 → 实现），并通过文件驱动的任务/方案追踪体系确保过程可审计、可回溯。

当 [Vibe Kanban](https://vibekanban.com/) MCP 可用时，插件会自动升级为**多 Agent 并行编排**模式：将任务拆分为互不冲突的子任务，通过 VKB 分发到独立工作空间并行执行，最终由主工作空间完成合并。VKB 不可用时，无感降级为 PMA 单 Agent 模式。

## 安装

本插件通过 GitHub 仓库安装（未发布到官方市场），需要先添加 marketplace 再安装插件。

在 Claude Code 中执行：

```
/plugin marketplace add BenDaye/pma-vkb
/plugin install pma-vkb@pma-vkb
```

## 核心特性

- **三阶段门控**：调查（Investigation）→ 方案（Proposal）→ 实现（Implementation），每个阶段设有明确的准入/准出条件，杜绝未经审查的代码修改
- **文件驱动追踪**：`docs/task/` 和 `docs/plan/` 目录作为唯一事实来源，所有任务和方案均以 Markdown 文件管理，天然支持 Git 版本控制
- **VKB 并行编排**（可选）：自动检测 Vibe Kanban MCP，将可并行化任务拆分到独立工作空间，支持依赖 DAG、阻塞关系、进度轮询和自动合并
- **优雅降级**：VKB 不可用时自动切换为 PMA-only 模式，核心流程不受影响
- **中文优先**：所有生成的文档、任务文件、Changelog、Commit 信息均使用中文

## 工作流概览

```
用户请求
  │
  ▼
┌──────────────────────────┐
│  Phase 1: 调查            │
│  - 追踪调用链和符号引用     │
│  - 查找/创建任务文件        │
│  - 评估复杂度，必要时创建方案 │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│  Phase 2: 方案            │
│  - 输出: 现状/方案/风险/工作量│
│  - VKB 模式下追加子任务拆分  │
│  - STOP，等待用户确认       │
└──────────┬───────────────┘
           │  用户回复 `proceed`
           ▼
┌──────────────────────────────────────┐
│  Phase 3: 实现                        │
│  ┌─────────────┐  ┌────────────────┐ │
│  │ Path A: VKB │  │ Path B: PMA    │ │
│  │ 多 Agent 编排│  │ 单 Agent 实现   │ │
│  └─────────────┘  └────────────────┘ │
└──────────────────────────────────────┘
```

## 项目结构

```
pma-vkb/
├── .claude-plugin/
│   ├── plugin.json          # 插件元信息（名称、版本、描述）
│   └── marketplace.json     # GitHub 安装配置
├── skills/
│   └── pma-vkb/
│       ├── SKILL.md         # 核心 Skill 定义（完整工作流指令）
│       └── docs/
│           ├── task-format.md              # 任务文件格式规范
│           ├── plan-format.md              # 方案文件格式规范
│           └── subtask-prompt-template.md  # VKB 子任务 Prompt 模板
└── README.md
```

## 使用方式

安装后，当你在任何项目中要求 Agent 进行代码修改时（如"修复登录页的 Bug"、"添加用户认证功能"），该 Skill 会自动触发，引导 Agent 按照三阶段流程执行。

首次使用时，插件会在项目根目录自动初始化：
- `docs/task/index.md` — 任务索引
- `docs/plan/index.md` — 方案索引
- `docs/changelog.md` — 变更日志

### 关键交互点

| 用户输入 | 效果 |
|---------|------|
| 描述需求 | Agent 进入调查阶段，完成后输出方案 |
| `proceed` / `开始实现` | 确认方案，Agent 进入实现阶段 |
| `push` | 实现完成后推送到远程 |
| `继续监控` | VKB 编排轮询超限后继续 |
| `跳到合并` | 跳过未完成的子任务进入合并 |

## 文件格式参考

- [任务格式规范](skills/pma-vkb/docs/task-format.md) — 任务 ID 规则、状态标记、索引/详情模板
- [方案格式规范](skills/pma-vkb/docs/plan-format.md) — 方案 ID 规则、状态流转、索引/详情模板
- [子任务 Prompt 模板](skills/pma-vkb/docs/subtask-prompt-template.md) — VKB 子工作空间的指令渲染模板

## 兼容性

- **必须**：Claude Code（或其他支持 Claude Code 插件体系的 Agent 环境）
- **可选**：[Vibe Kanban](https://vibekanban.com/) MCP —— 启用后解锁多 Agent 并行编排能力

## 许可证

MIT
