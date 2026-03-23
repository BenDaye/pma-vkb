# Path A: VKB Orchestration — Full Procedure

This document contains the detailed A1–A10 steps for VKB-orchestrated parallel implementation. Referenced from SKILL.md Step 4, Path A.

---

## A1. Claim Task in Docs

1. Update `docs/task/index.md`: change the task marker from `[ ]` to `[-]`.
2. Update the task detail file: set `status: in_progress`, set `owner: pma-vkb-orchestrator`.
3. If a plan exists, update `docs/plan/index.md`: change `[ ]` to `[-]`. Update plan detail: set `status: implementing`.

## A2. Create Sub-Task Docs

For each sub-task in the split table:
1. Create `docs/task/SUB-NNN.md` using the Chinese template. Set `status: pending`, `priority` from the table, `blocked by` from the DAG.
2. Append to `docs/task/index.md`:
   ```
   - [ ] [**SUB-NNN 标题**](SUB-NNN.md) `P1`
   ```

## A3. Create VKB Parent Issue

Call:
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

## A4. Link Main Workspace

If `MAIN_WORKSPACE_ID` is not null:
```
mcp__vibe_kanban__link_workspace_issue(
  workspace_id = MAIN_WORKSPACE_ID,
  issue_id     = PARENT_ISSUE_ID
)
```

This makes the current workspace the **main workspace** and the current session the **main session**. The parent issue status will auto-change to "In progress".

## A5. Create VKB Child Issues

For each sub-task, call:
```
mcp__vibe_kanban__create_issue(
  project_id      = PROJECT_ID,
  title           = "{SUB-NNN} {sub-task title}",
  description     = <rendered from subtask-prompt-template.md — see docs/subtask-prompt-template.md>,
  priority        = <from split table>,
  parent_issue_id = PARENT_ISSUE_ID
)
```
→ Store each returned `issue_id` in a map: `SUB_ISSUES[SUB-NNN] = {issue_id, branch: null, workspace_id: null}`.

## A6. Create Blocking Relationships

For each dependency pair in the DAG (e.g. SUB-001 blocks SUB-002):
```
mcp__vibe_kanban__create_issue_relationship(
  issue_id         = SUB_ISSUES["SUB-001"].issue_id,
  related_issue_id = SUB_ISSUES["SUB-002"].issue_id,
  relationship_type = "blocking"
)
```

## A7. Dispatch Sub Workspaces

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

## A8. Monitoring Loop

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

**Step 8f — Output progress report:**

After each poll cycle, output a **concise** status line per sub-task (do NOT repeat full tables every iteration to conserve context window):

```
[轮询 {N}/20] SUB-001: Done ✓ | SUB-002: In progress (3 commits) | SUB-003: blocked
```

Every **5th iteration**, output the full table and append to `docs/task/{TASK_ID}.md` under Notes:

```
## 执行进度 — {TASK_ID} (轮询 {N}/20)

| 子任务 | VKB ID | 状态 | branch | 最新 commit |
|--------|--------|------|--------|------------|
| SUB-001 | BEN-XX | Done | feature/xxxx-... | abc1234 |
| SUB-002 | BEN-YY | In progress | feature/yyyy-... | def5678 |
| SUB-003 | BEN-ZZ | To do (blocked by SUB-001) | — | — |
```

**Step 8g — Iteration limit reached:**

If 20 iterations are exhausted:
```
监控已达上限（20 轮）。当前状态：
{full status table}
请检查未完成的子任务后回复 `继续监控` 或 `跳到合并`。
```

## A9. Merge Phase

When **all sub-tasks** are Done (or failed sub-tasks have been acknowledged):

1. Get each sub-task's branch from the cached `SUB_ISSUES` map.

2. Fetch all remote branches:
   ```bash
   git fetch origin
   ```

3. Merge in **topological order** of the dependency DAG (dependencies first):
   ```bash
   git merge origin/{sub-branch-1} --no-ff --no-edit
   ```
   - If the merge succeeds → run the verification command from the merge strategy (e.g. `bun run lint && bun run test`).
   - If the merge has conflicts → abort and **STOP**:
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
   - If verification fails after a successful merge → output the failure details and ask user:
     ```
     合并 {sub-branch} 后验证失败：
     {failure details}
     请选择：
     1. `reset` — 回退到合并前状态（git reset --hard HEAD~1）
     2. `继续` — 保留合并，手动修复后回复
     ```
     If user chooses `reset`, run `git reset --hard HEAD~1` and skip this sub-branch.

4. After all branches are merged, run full verification:
   ```bash
   {project test/lint/build commands}
   ```

## A10. Cleanup and Completion

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

4. **Push or confirm:**
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
