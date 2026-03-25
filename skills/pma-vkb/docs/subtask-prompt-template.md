# Sub-Task Prompt Template

This template is used by the orchestrator to render the `description` field of VKB child issues created in Step A5. The `start_workspace` tool uses the issue description as the agent's prompt when no explicit prompt is provided.

There are two templates:
- **Leaf Executor Template**: For sub-tasks that should be implemented directly.
- **Orchestrator Node Template**: For sub-tasks that are complex enough to require their own fission cycle.

The orchestrator selects the template based on the sub-task's **type** classification (Section 3.2.2 of SKILL.md).

---

## Leaf Executor Template

```markdown
## 执行模式: 子任务实现

你是一个独立执行的子任务 agent。严格按以下要求完成工作，不做任何额外的事情。

### 任务

{title}

### 上下文

{context}

### 要求

{requirements}

### 验收标准

{acceptance_criteria}

### 文件范围（严格限制）

只允许修改以下文件：
{file_list}

新建文件：
{new_files}

**禁止修改范围外的任何文件。**

### 约束

- 不修改范围外文件
- 不添加未请求的重构或功能
- 不创建 docs/task/、docs/plan/ 或 docs/changelog.md 文件
- Commit message 使用 conventional commits 格式（中文描述），例如 `feat: 新增用户认证中间件`
- 每个逻辑完整的改动做一次 commit
- 完成所有改动后运行验证命令确认通过
- **完成后必须 push 到 origin**：`git push origin HEAD`

### 环境准备

如有依赖安装命令，先执行：
\`\`\`bash
{setup_command}
\`\`\`
如显示 `（无）` 则跳过此步。

### 验证命令

\`\`\`bash
{test_command}
{lint_command}
{build_command}
\`\`\`

### 错误处理

- 如果验证命令失败，修复问题后重新运行，不要跳过
- 如果遇到范围外的依赖问题（缺少类型、接口未定义等），在 commit message 中记录，继续完成范围内的工作
- 如果完全无法继续，commit 已完成的工作并 push，在最后一个 commit message 中说明阻塞原因

### 参考代码

- 现有模式参考: {reference_paths}
```

---

## Orchestrator Node Template

```markdown
## 执行模式: 编排节点

你是一个编排节点 agent。你的职责是对当前任务进行调研、裂变判断和编排。你运行的是 pma-vkb skill 的完整流程。

### 任务

{title}

### 上下文

{context}

### 父级方案摘要

{plan_excerpt}

### 要求

{requirements}

### 模块范围

本编排节点负责以下模块/目录：
{module_scope}

### 编排指令

1. 对当前任务执行 investigation（读代码、理解范围）
2. 进行裂变判断（参照 SKILL.md Section 3.2.1）：
   - 如果可以直接实现（DIRECT: ≤ 5 文件、单一模块）→ 使用 Path B 直接实现
   - 如果需要裂变（FISSION）→ 创建子 issues、dispatch workspaces、监控、review、merge
3. 裂变时遵循 pma-vkb skill 的 Path A 流程（A2~A10）
4. 当前 issue 作为 PARENT_ISSUE_ID，子 issues 作为其 children
5. 完成所有工作后将当前 issue 状态设为 Done

### 递归深度

当前深度: {current_depth}
最大深度: 3
根 Issue ID: {root_issue_id}

如果 current_depth ≥ 3，**禁止继续裂变**，必须使用 Path B 直接实现。

### 约束

- 自动执行，不等待人工确认（proceed 已在根 session 授权）
- Commit message 使用 conventional commits 格式（中文描述）
- 完成所有改动后运行验证命令确认通过
- **完成 merge 后必须 push 到 origin**：`git push origin HEAD`
- 不创建 docs/changelog.md 文件（仅根编排器管理 changelog）
- 可以在自己的 workspace 内创建 docs/task/ 中的子任务文件（如果运行 Path A 流程），但这些文件仅作为本 workspace 的内部追踪，根编排器不依赖它们

### 环境准备

\`\`\`bash
{setup_command}
\`\`\`

### 验证命令

\`\`\`bash
{test_command}
{lint_command}
{build_command}
\`\`\`

### 参考代码

- 现有模式参考: {reference_paths}
```

---

## Variable Reference

### Shared Variables (both templates)

| Variable | Description | Source |
|----------|-------------|--------|
| `{title}` | Sub-task title | Split table "标题" column |
| `{context}` | Current state and relevant background | `PLAN-NNN.md` "现状" section |
| `{requirements}` | Specific changes to implement | `PLAN-NNN.md` "方案" section, filtered to this sub-task |
| `{setup_command}` | Dependency install command | `bun install`, `npm ci`, etc. Use `（无）` if pre-installed |
| `{test_command}` | Project test command | `package.json` scripts, `Makefile`, or project convention |
| `{lint_command}` | Project lint command | Same source as test_command |
| `{build_command}` | Project build command | Same source as test_command |
| `{reference_paths}` | Paths to similar existing code | From proposal investigation phase |

### Leaf Executor Variables

| Variable | Description | Source |
|----------|-------------|--------|
| `{acceptance_criteria}` | Numbered list of pass/fail criteria | Split table "验收标准" column |
| `{file_list}` | Existing files this sub-task may modify | Split table "文件范围" column, one per line with `- ` prefix |
| `{new_files}` | New files to create | From proposal, one per line with `- ` prefix. Use `（无）` if none |

### Orchestrator Node Variables

| Variable | Description | Source |
|----------|-------------|--------|
| `{plan_excerpt}` | Relevant sections from the parent plan (现状, 方案, 风险) | Extracted from `PLAN-NNN.md`, filtered to this sub-task's scope |
| `{module_scope}` | Directories/modules this orchestrator is responsible for | Split table "文件范围" column (directory-level) |
| `{current_depth}` | Current recursion depth (0 = root) | Parent's depth + 1 |
| `{root_issue_id}` | The root issue ID of the entire task tree | Passed unchanged from parent; root session sets it to PARENT_ISSUE_ID |

## Rendering Rules

1. Replace each `{variable}` with its value. If a value is empty, use `（无）`.
2. `{acceptance_criteria}` must be a numbered list (1. 2. 3. ...).
3. `{file_list}`, `{new_files}`, and `{module_scope}` must be bullet lists with `- ` prefix.
4. The rendered description is passed as-is to `create_issue(description=...)`.
5. **Template selection**: Use **Leaf Executor** when the sub-task type is `叶子`. Use **Orchestrator Node** when the sub-task type is `编排`.
