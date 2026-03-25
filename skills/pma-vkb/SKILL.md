---
name: pma-vkb
version: 2.0.0
description: >
  Project development lifecycle management with a strict three-phase
  workflow (investigate → proposal → implement), file-based task tracking
  in docs/task/, plan tracking in docs/plan/, and Vibe Kanban MCP
  orchestration for parallel agent dispatch via isolated workspaces.
  Supports recursive fission: complex tasks are automatically decomposed
  into sub-tasks, each dispatched to its own workspace where the agent
  independently decides whether to implement directly or fission further.
  Only one user `proceed` is needed at the root session — all subsequent
  orchestration (splitting, dispatching, reviewing, merging, reporting,
  pushing) is fully automated.
  Use when handling feature development, bug fixes, refactors, planning,
  progress tracking, multi-agent parallel execution, or any code change
  in an existing codebase. Trigger this skill whenever the user asks to
  fix a bug, add a feature, refactor code, or make any code change —
  even small ones. Also trigger when they mention task tracking, project
  planning, work coordination, or want to dispatch multiple agents in
  parallel. Automatically invoked for all implementation tasks — the
  agent should not write code without first completing the investigation
  and proposal phases.
compatibility:
  tools:
    - mcp__vibe_kanban__* (optional — gracefully degrades to PMA-only mode)
---

# PMA-VKB — Project Management + Vibe Kanban Orchestration

Run delivery work with clear gates, minimal diffs, explicit task/plan tracking, and optional parallel agent dispatch via Vibe Kanban.

---

## Hard Rules

1. All conversation output, generated documents, task files, plan files, commit messages, and PR content are in Chinese. Skill definitions and config files stay in English.
2. Use English filenames only (e.g. `architecture.md`, `changelog.md`).
3. Read before write: inspect call chains, related config/tests, and recent changelog context before editing logic.
4. Make only the minimal requested changes; do not add unrequested refactors or features.
5. Never use plan mode (`EnterPlanMode`, `mode: "plan"`). Manage plans in `docs/plan/` files only.
6. Do not implement before explicit confirmation (`proceed` / `开始实现`) — **unless `AUTO_PROCEED = true`** (see Step 1.0).
7. `docs/task/` and `docs/plan/` files are the **sole source of truth**. VKB state syncs FROM docs, never the reverse.
8. VKB integration is pluggable. When VKB MCP is unavailable, degrade to PMA-only mode transparently.

## Recursive Fission Limits

When using recursive fission (sub-workspaces dispatching their own sub-workspaces), these hard limits apply to prevent runaway orchestration:

| Limit | Value | Enforcement |
|-------|-------|-------------|
| **Max fission depth** | 3 levels | Level 0 = user-created root issue. Level 1 = first fission. Level 2 = second fission. Level 3 = must be leaf executor, no further fission allowed. |
| **Max total issues per root task** | 30 | Before creating a child issue, count direct children via `list_issues(parent_issue_id=PARENT_ISSUE_ID)`. If the current parent already has children approaching the limit, stop fissioning. Since VKB only supports single-level parent queries, use a conservative estimate: `current_children_count × average_grandchildren ≤ 30`. |
| **Max children per parent issue** | 8 | A single issue may have at most 8 direct child issues. |

**Depth tracking**: Each orchestrator node template includes `{current_depth}` and `{root_issue_id}` variables. The root session sets depth = 0 and `root_issue_id` to `PARENT_ISSUE_ID`. Each fission increments depth by 1 and passes `root_issue_id` unchanged. When depth ≥ 3, the agent MUST use Path B (direct implementation) regardless of complexity.

---

## Step 0: Project Initialization

On first invocation, check and create these files if missing:

1. Read `docs/task/index.md`. If it does not exist, create it from the template in [docs/task-format.md](docs/task-format.md) using the Chinese template.
2. Read `docs/plan/index.md`. If it does not exist, create it from the template in [docs/plan-format.md](docs/plan-format.md) using the Chinese template.
3. Read `docs/changelog.md`. If it does not exist, create it with:
   ```markdown
   # Changelog
   ```

Then proceed to Step 1.

---

## Step 1: VKB Context Discovery

Execute these sub-steps **in order**. Cache every result for use in later steps.

### 1.0 Detect Entry Mode

Determine how this session was initiated.

**Initialize defaults:**
```
ENTRY_MODE        = "workspace_driven"
AUTO_PROCEED      = false
IS_SUB_ORCHESTRATOR = false
MAIN_WORKSPACE_ID = null
LINKED_ISSUE_ID   = null
CURRENT_DEPTH     = 0
```

Run `git branch --show-current` → store as `CURRENT_BRANCH`.

Call `mcp__vibe_kanban__list_workspaces()`.

- Iterate through workspaces. If any workspace `branch` equals `CURRENT_BRANCH`:
  - Set `MAIN_WORKSPACE_ID` to its `id`.
  - If the workspace has an associated `issue_id`:
    - Call `mcp__vibe_kanban__get_issue(issue_id)` → read the issue's `title`, `description`, `project_id`, `parent_issue_id`.
    - Set `ENTRY_MODE = "issue_driven"`.
    - Set `LINKED_ISSUE_ID` to that `issue_id`.
    - Set `PROJECT_ID` to the issue's `project_id`.
    - **Auto-proceed detection**: If the issue `description` starts with `## 执行模式:`, this is a skill-generated issue. Set `AUTO_PROCEED = true`. The agent will skip the proceed gate (Step 3.4) and enter Phase 3 automatically after completing the proposal.
    - If `parent_issue_id` is not null → set `IS_SUB_ORCHESTRATOR = true` (this workspace is a child of another orchestrator).
    - Skip to Step 1.1 (VKB availability is already confirmed since the workspace call succeeded).
  - If the workspace has no associated `issue_id`:
    - Set `ENTRY_MODE = "workspace_driven"`.
    - Continue to Step 1.1.
- If no workspace matches `CURRENT_BRANCH`:
  - Set `ENTRY_MODE = "workspace_driven"`.
  - Set `MAIN_WORKSPACE_ID = null`.
  - Continue to Step 1.1.

### 1.1 Detect VKB Availability

Call `mcp__vibe_kanban__list_organizations()`. This call is always needed — even in `issue_driven` mode — to obtain `ORG_ID` and `org_name` for the context output in Step 1.5.

- **If the call fails or the tool does not exist**: set `VKB_MODE = false`. Output:
  ```
  VKB MCP 不可用，使用 PMA-only 模式。
  ```
  Skip to Step 2 (Phase 1).

- **If it succeeds**: extract the organizations list.
  - If exactly one org → set `ORG_ID` to its `id`.
  - If multiple orgs → list them and ask the user to choose. Set `ORG_ID` to the chosen `id`.

### 1.2 Resolve Project

If `PROJECT_ID` was already set in Step 1.0 (issue_driven mode):
- Call `mcp__vibe_kanban__list_projects(organization_id = ORG_ID)` → find the project matching `PROJECT_ID` to cache `project_name`.
- Skip the rest of this step.

Otherwise, call `mcp__vibe_kanban__list_projects(organization_id = ORG_ID)`.

- If the project list is empty → output: "VKB 中没有 Project，请先在 Vibe Kanban UI 中创建（MCP 无 create_project 工具）。" Set `VKB_MODE = false`. Skip to Step 2.

- Get the current directory name: `basename` of `$PWD`.
- Iterate through returned projects. Try to match by:
  1. Exact match (case-insensitive): project `name` == directory name.
  2. Substring match: project `name` contained in directory name, or vice versa.

- **If exactly one project matches** → set `PROJECT_ID` to its `id`.
- **If multiple projects match, or no project matches** → ask the user. Output:
  ```
  VKB 中有以下 Project，请选择用于本次任务的 Project：
  1. {project_name_1} ({project_id_1})
  2. {project_name_2} ({project_id_2})
  ...
  当前目录: {directory_name}
  请输入 Project 编号或名称:
  ```
  Wait for user response. Set `PROJECT_ID` to the chosen project's `id`.
  Do not guess or auto-select when ambiguous — always ask.

### 1.3 Resolve Repository

Call `mcp__vibe_kanban__list_repos()`.

- Iterate through repos. If any repo `name` matches the current directory name → set `REPO_ID` to its `id`.
- If no match → list all repo names and ask the user to confirm which one.

### 1.4 Identify Current Workspace

If `MAIN_WORKSPACE_ID` was already set in Step 1.0, skip this step.

Otherwise:
- Run `git branch --show-current` → store as `CURRENT_BRANCH` (if not already set).
- Call `mcp__vibe_kanban__list_workspaces()`.
- Iterate through workspaces. If any workspace `branch` equals `CURRENT_BRANCH` → set `MAIN_WORKSPACE_ID` to its `id`.
- If no match → set `MAIN_WORKSPACE_ID = null`. VKB orchestration is still possible; the workspace will be linked later.

### 1.5 Confirm Context

Set `VKB_MODE = true`. Output:

```
VKB 上下文已建立:
  Entry Mode:   {ENTRY_MODE} {AUTO_PROCEED ? "(auto-proceed)" : ""}
  Organization: {org_name} ({ORG_ID})
  Project:      {project_name} ({PROJECT_ID})
  Repository:   {repo_name} ({REPO_ID})
  Workspace:    {ws_name} ({MAIN_WORKSPACE_ID}) @ {CURRENT_BRANCH}
  Linked Issue: {LINKED_ISSUE_ID ? simple_id + " " + title : "(无)"}
```

---

## Step 2: Phase 1 — Investigation

### 2.1 Understand the Request

Read and trace:
- Upstream/downstream call chains and symbol references related to the request.
- Related config, tests, migrations, and type definitions.
- The tail of `docs/changelog.md` (last 20 lines) for recent context.

### 2.2 Find or Create Task

Read `docs/task/index.md`.

- Search for an existing task matching the request topic.
  - If found and `[ ]` (pending) → this is the task. Read its detail file.
  - If found and `[-]` (in progress) → read its detail file, check `owner`. If owned by another agent, inform user and stop. Otherwise, continue.
  - If found and `[x]` or `[~]` → the task is already done or closed. Inform user.

- If no matching task exists → create one:
  1. Determine the next available ID for the appropriate prefix (e.g. `FEAT-001`, `BUG-001`).
  2. Create `docs/task/PREFIX-NNN.md` using the Chinese template from [docs/task-format.md](docs/task-format.md).
  3. Append one line to `docs/task/index.md`:
     ```
     - [ ] [**PREFIX-NNN 简短标题**](PREFIX-NNN.md) `P1`
     ```

### 2.3 VKB Cross-Reference (if VKB_MODE = true)

Call `mcp__vibe_kanban__list_issues(project_id = PROJECT_ID, search = "关键词")` to check if a related VKB issue already exists.

- If found → note the `issue_id` and `simple_id` in the task detail file under Notes.
- If not found → no action needed; the VKB issue will be created in Phase 3.

### 2.4 Evaluate Complexity

If the change touches **≥ 3 files** or **crosses module boundaries**:
1. Determine the next PLAN ID (e.g. `PLAN-001`).
2. Create `docs/plan/PLAN-NNN.md` using the Chinese template from [docs/plan-format.md](docs/plan-format.md). Fill in the "现状" section with investigation findings.
3. Append one line to `docs/plan/index.md`:
   ```
   - [ ] [**PLAN-NNN 简短标题**](PLAN-NNN.md) `YYYY-MM-DD`
   ```
4. Set `relatedTask` in the plan detail to the task ID.

---

## Step 3: Phase 2 — Proposal

### 3.1 Output Proposal

Output the following sections **in Chinese**:

1. **现状**: What exists now (files, modules, current behavior).
2. **方案**: What to change and how.
3. **风险**: Side effects, migration needs, potential bugs.
4. **工作量**: Estimated scope (files, modules affected).
5. **备选方案**: Alternative approaches (if multiple exist).

### 3.2 VKB Orchestration Plan (if VKB_MODE = true)

#### 3.2.1 Fission Judgment

Based on investigation results, the agent evaluates the task's **fission level**:

| Level | Name | Criteria | Action |
|-------|------|----------|--------|
| **DIRECT** | 直接实现 | ≤ 5 files AND single module/bounded context | Use Path B (direct implementation) |
| **FISSION** | 裂变 | Can be split into ≥ 2 sub-tasks with disjoint file sets/modules | Split into sub-tasks via Path A. Each sub-task is classified as leaf or orchestrator (see 3.2.2). |

The agent chooses DIRECT when possible. FISSION is used whenever the task is too large for direct implementation.

**Key principle**: Each sub-task dispatched to a workspace runs the **same SKILL.md**. A sub-task classified as "orchestrator node" will run its own fission judgment — the recursive model handles multi-level decomposition naturally. The orchestrator only plans **one level down**.

#### 3.2.2 Sub-Task Classification

For each sub-task in the split, classify it:

- **Leaf executor (叶子执行)**: The sub-task is small enough to implement directly. Use the leaf executor prompt template.
- **Orchestrator node (编排节点)**: The sub-task is complex enough that it will likely need its own fission. Use the orchestrator node prompt template. The dispatched agent will run the full SKILL.md cycle (investigate → fission judgment → implement or sub-dispatch).

Heuristics for classification:
- Leaf: ≤ 5 files, single concern, clear acceptance criteria
- Orchestrator: crosses module boundaries, >5 files, or the agent cannot yet enumerate all files (needs further investigation in context)

#### 3.2.3 Output Format

If fission level is FISSION, output these additional sections:

6. **子任务拆分表**:

   | 子任务 ID | 标题 | 类型 | 文件范围 | 依赖 | 优先级 | 验收标准 |
   |-----------|------|------|---------|------|--------|---------|
   | SUB-001 | ... | 叶子 | src/hooks/... | 无 | high | ... |
   | SUB-002 | ... | 编排 | src/modules/auth/ | SUB-001 | high | (子 workspace 自行调研) |

   - **类型** column: `叶子` (leaf executor) or `编排` (orchestrator node).
   - For orchestrator nodes, **文件范围** can be a directory or module path (not exhaustive file list).
   - For leaf executors, **文件范围** must list specific files.

7. **文件归属矩阵**: Confirm each file/directory appears in **exactly one** sub-task. If any file appears in multiple sub-tasks, merge those sub-tasks or re-split.

8. **依赖 DAG**: List blocking relationships. Example: `SUB-001 → blocks → SUB-002`.

9. **Merge 策略**: The order in which sub-task branches will be merged (topological order of the dependency DAG), and which verification commands to run after each merge.

10. **Executor 选择**: Default `CLAUDE_CODE`. Specify alternatives only if a sub-task requires a different executor.

### 3.3 Write to Plan File

If a plan file exists (`PLAN-NNN.md`), fill in all remaining sections (方案, 风险, 工作量, 备选方案, and the orchestration sections if applicable).

### 3.4 Proceed Gate

**If `AUTO_PROCEED = true`** (skill-generated issue, detected in Step 1.0):
- Do NOT stop. Do NOT wait for user confirmation.
- Output: "自动执行模式：跳过确认，直接进入实现阶段。"
- Immediately proceed to Step 4.

**Otherwise** (user-created issue or workspace-driven entry):
- Output: "方案已输出。请确认后回复 `proceed` 开始实现。"
- **Do not proceed until the user explicitly confirms.**

---

## Step 4: Phase 3 — Implementation

On receiving `proceed` (or `开始实现` or equivalent confirmation), or when `AUTO_PROCEED = true`:

First, determine which implementation path to take:

- **Path A (VKB Orchestration)**: `VKB_MODE = true` AND fission level is FISSION (Section 3.2.1).
- **Path B (PMA-only)**: `VKB_MODE = false` OR fission level is DIRECT (no sub-task table).

### Why Path A and Path B are mutually exclusive

When VKB orchestration is active and sub-tasks have been defined, the main workspace acts purely as an **orchestrator** — it creates VKB issues, dispatches sub-workspaces, monitors progress, and merges branches. It does not write application code itself, because sub-workspace agents own the implementation and their branches would conflict with direct edits on the main branch.

The only code the main workspace may write directly:
- `docs/task/` and `docs/plan/` files
- Project scaffolding that must exist before sub-tasks can run (e.g. `create-next-app` initialization, `package.json`, base config files)
- Merge commits when combining sub-task branches

---

### Path A: VKB Orchestration

#### A1. Claim Task in Docs

1. Update `docs/task/index.md`: change the task marker from `[ ]` to `[-]`.
2. Update the task detail file: set `status: in_progress`, set `owner: pma-vkb-orchestrator`.
3. If a plan exists, update `docs/plan/index.md`: change `[ ]` to `[-]`. Update plan detail: set `status: implementing`.

#### A2. Create Sub-Task Docs

For each sub-task in the split table:
1. Create `docs/task/SUB-NNN.md` using the Chinese template. Set `status: pending`, `priority` from the table, `blocked by` from the DAG.
2. Append to `docs/task/index.md`:
   ```
   - [ ] [**SUB-NNN 标题**](SUB-NNN.md) `P1`
   ```

#### A3. Resolve or Create VKB Parent Issue

**If `ENTRY_MODE = "issue_driven"`** (workspace was started from an existing issue):
- `PARENT_ISSUE_ID` is already set to `LINKED_ISSUE_ID` from Step 1.0.
- Do NOT create a new issue. Skip to A4.

**If `ENTRY_MODE = "workspace_driven"`** (no pre-existing issue):
- Call:
  ```
  mcp__vibe_kanban__create_issue(
    project_id = PROJECT_ID,
    title      = "{TASK_ID} {task title}",
    description = "编排任务（main workspace 执行）。\n详见 docs/plan/PLAN-NNN.md",
    priority   = "high"
  )
  ```
  → Store the returned `issue_id` as `PARENT_ISSUE_ID`.

Record `PARENT_ISSUE_ID` in the task detail file under Notes.

#### A4. Link Main Workspace

**If `ENTRY_MODE = "issue_driven"`**: skip (workspace is already linked to the issue).

**Otherwise**, if `MAIN_WORKSPACE_ID` is not null:
```
mcp__vibe_kanban__link_workspace_issue(
  workspace_id = MAIN_WORKSPACE_ID,
  issue_id     = PARENT_ISSUE_ID
)
```

This makes the current workspace the **main workspace** and the current session the **main session**. The parent issue status will auto-change to "In progress".

#### A5. Create VKB Child Issues

For each sub-task in the split table, render the issue description based on its **type** (from Section 3.2.2):

- **Leaf executor (叶子)**: render from the **leaf executor template** in [docs/subtask-prompt-template.md](docs/subtask-prompt-template.md).
- **Orchestrator node (编排)**: render from the **orchestrator node template** in [docs/subtask-prompt-template.md](docs/subtask-prompt-template.md). Include `{plan_excerpt}` with the relevant sections from the current plan (现状, 方案, 风险) so the sub-orchestrator has full context.

Then call:
```
mcp__vibe_kanban__create_issue(
  project_id      = PROJECT_ID,
  title           = "{SUB-NNN} {sub-task title}",
  description     = <rendered description>,
  priority        = <from split table>,
  parent_issue_id = PARENT_ISSUE_ID
)
```
→ Store each returned `issue_id` in a map: `SUB_ISSUES[SUB-NNN] = {issue_id, type: "leaf"|"orchestrator", branch: null, workspace_id: null}`.

#### A6. Create Blocking Relationships

For each dependency pair in the DAG (e.g. SUB-001 blocks SUB-002):
```
mcp__vibe_kanban__create_issue_relationship(
  issue_id         = SUB_ISSUES["SUB-001"].issue_id,
  related_issue_id = SUB_ISSUES["SUB-002"].issue_id,
  relationship_type = "blocking"
)
```

#### A7. Dispatch Sub Workspaces

Identify sub-tasks with **no unresolved dependencies** (nothing blocks them, or all blockers are already Done).

For each such sub-task:
```
mcp__vibe_kanban__start_workspace(
  name         = "{SUB-NNN}-{short-description}",
  executor     = "CLAUDE_CODE",
  repositories = [{"repo_id": REPO_ID, "branch": CURRENT_BRANCH}],
  issue_id     = SUB_ISSUES["SUB-NNN"].issue_id
)
```
→ Store returned `workspace_id` in `SUB_ISSUES["SUB-NNN"].workspace_id`.

Then update docs:
1. `docs/task/SUB-NNN.md`: set `status: in_progress`, set `owner: vkb-workspace`.
2. `docs/task/index.md`: change `[ ]` to `[-]` for this sub-task.

#### A8. Monitoring Loop

**Hard limits**: Maximum **20 poll iterations**. Each iteration starts with a `sleep` of at least **60 seconds** (increase if sub-tasks are large). If all 20 iterations are exhausted without completion, STOP and inform the user.

Repeat the following until **all sub-tasks are in a terminal state** (Done or failed-and-handled) or the iteration limit is reached:

**Step 8a — Wait, then query VKB statuses:**
```bash
sleep 60
```
```
mcp__vibe_kanban__list_issues(
  project_id      = PROJECT_ID,
  parent_issue_id = PARENT_ISSUE_ID
)
```
Record each issue's `status`.

**Step 8b — Resolve branches (first iteration + after new dispatches):**

On the first iteration, and whenever new sub-tasks were dispatched in Step 8d of a previous iteration, get branch names for sub workspaces:
```
mcp__vibe_kanban__list_workspaces()
```
Match by `workspace_id` → extract `branch`. Store in `SUB_ISSUES["SUB-NNN"].branch`.

**Step 8c — Determine completion:**

A sub-task is considered **complete** when its VKB issue status is **"Done"**.

As a secondary signal, for "In progress" sub-tasks, check git activity:
```bash
git fetch origin
git log origin/{branch} --oneline -1
```

When a sub-task's VKB status is Done:
1. Update `docs/task/SUB-NNN.md`: set `status: completed`.
2. Update `docs/task/index.md`: change `[-]` to `[x]`.

**Step 8d — Dispatch newly unblocked sub-tasks:**

For each sub-task still in "pending" state:
- Check if ALL its blocking dependencies are now Done.
- If yes → dispatch it (same as Step A7).

**Step 8e — Handle stalls:**

If a sub-task has been "In progress" for **3+ consecutive polls** with no new commits on its branch:
1. Attempt recovery: call `mcp__vibe_kanban__create_session(workspace_id = SUB_ISSUES["SUB-NNN"].workspace_id)` then `mcp__vibe_kanban__run_session_prompt(session_id = <new_session_id>, prompt = "继续完成上一个 session 未完成的任务。")`.
2. If still no progress after **3 more polls** → mark as failed:
   - Update `docs/task/SUB-NNN.md`: append failure details to Notes.
   - Inform user: "子任务 {SUB-NNN} 执行失败，请检查。"
   - Do NOT block other sub-tasks.

**Step 8f — Recursive progress drill-down (for orchestrator-type sub-tasks):**

Execute this step **every 3rd iteration** (iterations 3, 6, 9, ...) and on the **final iteration** to limit API call volume.

For each sub-task with `type = "orchestrator"` that is "In progress":
```
mcp__vibe_kanban__list_issues(
  project_id      = PROJECT_ID,
  parent_issue_id = SUB_ISSUES["SUB-NNN"].issue_id
)
```
If grandchild issues exist, include them in the progress report as nested entries. This provides visibility into the recursive fission without the main orchestrator needing to manage those grandchild tasks directly.

**Step 8g — Output progress report:**

After each poll cycle, output a **concise** status line per sub-task (do NOT repeat full tables every iteration to conserve context window):

```
[轮询 {N}/20] SUB-001: Done ✓ | SUB-002: In progress (3 commits) | SUB-003(编排): In progress (2/4 子任务完成)
```

Every **5th iteration**, output the full table and append to `docs/task/{TASK_ID}.md` under Notes:

```
## 执行进度 — {TASK_ID} (轮询 {N}/20)

| 子任务 | VKB ID | 类型 | 状态 | branch | 最新 commit |
|--------|--------|------|------|--------|------------|
| SUB-001 | BEN-XX | 叶子 | Done | feature/xxxx-... | abc1234 |
| SUB-002 | BEN-YY | 叶子 | In progress | feature/yyyy-... | def5678 |
| SUB-003 | BEN-ZZ | 编排 | In progress | feature/zzzz-... | — |
|   ↳ SUB-003/SUB-001 | BEN-AA | — | Done | feature/aaaa-... | ghi9012 |
|   ↳ SUB-003/SUB-002 | BEN-BB | — | In progress | feature/bbbb-... | — |
```

**Step 8h — Iteration limit reached:**

If 20 iterations are exhausted:

- **If `AUTO_PROCEED = true`** (sub-orchestrator): output the status table to the issue notes (via a commit message) and mark the issue as failed. The parent orchestrator will detect this.
- **Otherwise** (root session with user present):
  ```
  监控已达上限（20 轮）。当前状态：
  {full status table}
  请检查未完成的子任务后回复 `继续监控` 或 `跳到合并`。
  ```

#### A9. Review & Merge Phase

When **all sub-tasks** are Done (or failed sub-tasks have been acknowledged):

1. Get each sub-task's branch from the cached `SUB_ISSUES` map.

2. Fetch all remote branches:
   ```bash
   git fetch origin
   ```

3. **AI Code Review** — For each sub-task branch (in topological order), before merging:
   ```bash
   git diff {CURRENT_BRANCH}...origin/{sub-branch}
   ```
   Review the diff for:
   - **Correctness**: Does the code match the sub-task's acceptance criteria?
   - **Scope violation**: Does the diff touch files outside the sub-task's declared file scope?
   - **Security**: Hardcoded secrets, injection vulnerabilities, unsafe patterns.
   - **Style**: Consistent with project conventions.
   - **Conflicts with other sub-tasks**: Changes that would cause semantic conflicts even if git merge succeeds.

   If issues are found:
   - For **minor issues** (style, naming) → note them but proceed with merge. Fix post-merge if needed.
   - For **significant issues** (scope violation, bugs, security) → attempt automated fix:
     1. Call `mcp__vibe_kanban__create_session(workspace_id = SUB_ISSUES["SUB-NNN"].workspace_id)`.
     2. Call `mcp__vibe_kanban__run_session_prompt(session_id = <new>, prompt = "Code review 发现以下问题，请修复后 push:\n{issue_list}")`.
     3. Wait for the fix (re-enter monitoring for this sub-task, max 5 poll iterations).
     4. Re-fetch and re-review the branch.
   - If **AUTO_PROCEED = false** (root session) and the issue is critical → inform the user and ask for guidance.
   - If **AUTO_PROCEED = true** (sub-orchestrator) and fix fails after retry → proceed with merge but log the issue in the task notes.

4. **Merge** in **topological order** of the dependency DAG (dependencies first):
   ```bash
   git merge origin/{sub-branch-1} --no-ff --no-edit
   ```
   - If the merge succeeds → run the verification command from the merge strategy (e.g. `bun run lint && bun run test`).
   - If the merge has conflicts:
     - **If `AUTO_PROCEED = true`** (sub-orchestrator): attempt auto-resolution. If unable, abort and mark as failed.
     - **If `AUTO_PROCEED = false`** (root session): abort and **STOP**:
       ```bash
       git merge --abort
       ```
       Output:
       ```
       Merge 冲突：{sub-branch} → {CURRENT_BRANCH}
       冲突文件：
       - {file1}
       - {file2}
       已执行 git merge --abort 回滚。请选择：
       1. 手动解决冲突后回复 `继续`
       2. 跳过此子任务回复 `跳过 {SUB-NNN}`
       ```
       Wait for user to resolve and confirm before continuing.
   - If verification fails after a successful merge:
     - **If `AUTO_PROCEED = true`** (sub-orchestrator): attempt to fix the issue directly. If unable after one attempt, log the failure and continue with remaining merges.
     - **If `AUTO_PROCEED = false`** (root session): output the failure details and ask user:
       ```
       合并 {sub-branch} 后验证失败：
       {failure details}
       请选择：
       1. `reset` — 回退到合并前状态（git reset --hard HEAD~1）
       2. `继续` — 保留合并，手动修复后回复
       ```
       If user chooses `reset`, run `git reset --hard HEAD~1` and skip this sub-branch.

5. After all branches are merged, run full verification:
   ```bash
   {project test/lint/build commands}
   ```

#### A10. Cleanup and Completion

1. **Archive all sub workspaces:**
   For each sub-task with a workspace_id:
   ```
   mcp__vibe_kanban__update_workspace(
     workspace_id = SUB_ISSUES["SUB-NNN"].workspace_id,
     archived     = true
   )
   ```

2. **Update VKB parent issue:**
   ```
   mcp__vibe_kanban__update_issue(
     issue_id = PARENT_ISSUE_ID,
     status   = "Done"
   )
   ```

3. **(Optional) Clean up remote sub-branches:**
   ```bash
   git push origin --delete {sub-branch-1} {sub-branch-2} ...
   ```
   Skip if the team prefers to keep branches for audit.

4. **Push:**

   **If `AUTO_PROCEED = true`** (sub-orchestrator): push automatically, no confirmation needed.
   ```bash
   git push origin {CURRENT_BRANCH}
   ```

   **If `AUTO_PROCEED = false`** (root session with user present):
   Output:
   ```
   所有子任务已合并到 {CURRENT_BRANCH}。是否 push 到 origin？回复 `push` 执行，或自行操作。
   ```
   If user replies `push`:
   ```bash
   git push origin {CURRENT_BRANCH}
   ```

5. **Update PMA docs:**
   - `docs/task/{TASK_ID}.md`: set `status: completed`, append completion summary to Notes.
   - `docs/task/index.md`: change `[-]` to `[x]` for the main task.
   - If a plan exists:
     - `docs/plan/PLAN-NNN.md`: set `status: completed`.
     - `docs/plan/index.md`: change `[-]` to `[x]`.
   - `docs/changelog.md`: append entry:
     ```markdown
     ## YYYY-MM-DD HH:MM [进度]

     {TASK_ID}: {task title} — 完成。
     子任务: {list of SUB-NNN with VKB IDs}
     ```

6. **Output final report:**
   ```
   ## 完成报告 — {TASK_ID}

   所有子任务已完成并合并。
   | 子任务 | VKB ID | branch | 状态 |
   |--------|--------|--------|------|
   ...

   验证结果: {pass/fail details}
   分支合并: {merge summary}
   ```

---

### Path B: PMA-Only Implementation

#### B1. Claim Task

1. Update `docs/task/index.md`: `[ ]` → `[-]`.
2. Update task detail: `status: in_progress`, `owner: pma-agent`.
3. If plan exists: `docs/plan/index.md` `[ ]` → `[-]`, plan detail `status: implementing`.

#### B2. Implement

Follow the approved proposal step by step. After each significant change, run focused verification (compile, lint, relevant tests).

#### B3. Verify

Run full project verification (test suite, lint, build).

#### B4. Complete

1. `docs/task/{TASK_ID}.md`: set `status: completed`.
2. `docs/task/index.md`: `[-]` → `[x]`.
3. If plan exists: `docs/plan/PLAN-NNN.md` → `status: completed`, `docs/plan/index.md` → `[x]`.
4. `docs/changelog.md`: append entry.

---

## Status Sync Rules

Direction:
- **Orchestrator's own operations**: docs first → then sync to VKB.
- **Sub-agent completion**: VKB issue status is authoritative → orchestrator syncs back to docs.

| Timing | docs action | VKB action |
|--------|------------|------------|
| Task created | Create `docs/task/PREFIX-NNN.md` + append index | `create_issue(...)` |
| Task claimed | `[ ]` → `[-]`, status: in_progress | `link_workspace_issue(...)` (auto sets "In progress") |
| Sub-task dispatched | `[ ]` → `[-]` for sub-task | `start_workspace(issue_id=...)` (auto sets "In progress") |
| Sub-task done | `[-]` → `[x]`, status: completed | `update_issue(status="Done")` |
| Parent task done | `[-]` → `[x]`, status: completed | `update_issue(status="Done")` |
| Task closed | `[-]` → `[~]`, status: closed | (optional) `update_issue(...)` |

If any VKB call fails → log the error in the task detail Notes section. The docs state remains valid. Retry on next opportunity.

---

## Failure Recovery

| Scenario | Detection | Recovery |
|----------|-----------|----------|
| VKB MCP unavailable | `list_organizations()` fails | Set `VKB_MODE=false`, use Path B |
| No matching project | `list_projects()` returns no match | Ask user; if none, Path B |
| No matching repo | `list_repos()` returns no match | Ask user to confirm repo |
| `start_workspace` fails | MCP returns error | Log in task notes, inform user |
| Sub-agent no progress | No new git commits for 3+ consecutive polls | `create_session` + `run_session_prompt("继续完成任务")`. Retry once. |
| Sub-agent fails twice | No progress after 3 more polls post-retry | Mark sub-task failed in docs, inform user, continue other sub-tasks |
| Merge conflict | `git merge` returns non-zero | If `AUTO_PROCEED`: attempt auto-resolution, else stop and ask user |
| Issue status mismatch | `update_issue(status=X)` fails | Call `list_issues` to discover valid statuses, retry with correct name |
| Fission depth exceeded | `current_depth ≥ 3` | Force Path B (direct implementation), no further fission |
| Total issue limit reached | Descendant count ≥ 30 | Stop fissioning, implement remaining work directly |
| Sub-orchestrator fails | Orchestrator-type child issue marked failed | Log in parent task notes; if `AUTO_PROCEED=false`, inform user |
| Code review finds issues | AI review detects bugs/security/scope violations | Create new session with fix prompt; retry once; proceed if still failing |

---

## Format References

- Task format: [docs/task-format.md](docs/task-format.md)
- Plan format: [docs/plan-format.md](docs/plan-format.md)
- Sub-task prompt template: [docs/subtask-prompt-template.md](docs/subtask-prompt-template.md)

---

## PR Workflow

When creating a PR after completion:

1. Analyze **full** commit history from branch point using `git diff main...HEAD`.
2. Title: under 70 characters, in Chinese.
3. Body:
   ```
   ## 概要
   - {bullet points summarizing changes}
   - VKB Issues: {PARENT_ISSUE simple_id} + {list of SUB simple_ids}

   ## 子任务完成情况
   | 子任务 | 状态 | branch |
   |--------|------|--------|
   ...

   ## 测试计划
   - [ ] {verification items}
   ```
4. Push with `-u` flag if new branch.

---

## Changelog Conventions

Entry format:

```markdown
## YYYY-MM-DD HH:MM [tag]

[content in Chinese]
```

Tags: `[进度]`, `[BUG-P0]`, `[BUG-P1]`, `[踩坑]`, `[决策]`
