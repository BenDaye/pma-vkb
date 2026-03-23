# Sub-Task Prompt Template

This template is used by the orchestrator to render the `description` field of VKB child issues created in Step A5. The `start_workspace` tool uses the issue description as the agent's prompt when no explicit prompt is provided.

## Template

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

## Variable Reference

| Variable | Description | Source |
|----------|-------------|--------|
| `{title}` | Sub-task title | Split table "标题" column |
| `{context}` | Current state and relevant background | `PLAN-NNN.md` "现状" section |
| `{requirements}` | Specific changes to implement | `PLAN-NNN.md` "方案" section, filtered to this sub-task |
| `{acceptance_criteria}` | Numbered list of pass/fail criteria | Split table "验收标准" column |
| `{file_list}` | Existing files this sub-task may modify | Split table "文件范围" column, one per line with `- ` prefix |
| `{new_files}` | New files to create | From proposal, one per line with `- ` prefix. Use `（无）` if none |
| `{setup_command}` | Dependency install command | `bun install`, `npm ci`, `pip install -r requirements.txt`, etc. Use `（无）` if pre-installed |
| `{test_command}` | Project test command | `package.json` scripts, `Makefile`, or project convention |
| `{lint_command}` | Project lint command | Same source as test_command |
| `{build_command}` | Project build command | Same source as test_command |
| `{reference_paths}` | Paths to similar existing code | From proposal investigation phase |

## Rendering Rules

1. Replace each `{variable}` with its value. If a value is empty, use `（无）`.
2. `{acceptance_criteria}` must be a numbered list (1. 2. 3. ...).
3. `{file_list}` and `{new_files}` must be bullet lists with `- ` prefix.
4. The rendered description is passed as-is to `create_issue(description=...)`.
