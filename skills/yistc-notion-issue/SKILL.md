---
name: yistc-notion-issue
description: Workflow for handling a Notion issue end-to-end. Update Notion, create a git worktree from dev, implement the change, verify it, and open a GitHub PR linked to the issue.
---

# Workflow for working on a Notion issue
## Scope
Use this workflow only when you are explicitly assigned an issue, with an issue ID like `KIS-7`.

## Guidelines
- Language use: 在代码中使用英文代码风格+注释, 但是在 issue和文档等地方使用中文作为主语言
- You will be assigned an issue with issue ID such as 'KIS-7'. This ID is a notion database page property, not the page ID you need for mcp operations. You need to search for this page first and get the page id. 当你获取到 page id后, 你的回复中必须包含以下格式的一句话:
"记住：KIS-X 的 page_id 是 <page_id>"

- You need to strictly follow the steps listed below. You may consider each step as a task.

### Behavioral rules
- Do NOT change issue status unless explicitly required by a workflow step or requested by the user.
- Do NOT run local test, lint, build, or typecheck unless explicitly requested by the user.
- Always keep Notion updated with meaningful progress, especially after fixing CI failures or when the PR becomes ready to merge.

### Notion MCP References and Guidelines
- 在 Notion Page的Updates 区域添加新的 update toggle的时候, 遵循这个格式: "Update 2026-01-10: <update_summary>". 且新的update要放在最上面

### Goal
Complete the issue by:
1. updating the Notion issue
2. creating a dedicated git worktree and branch
3. planning before implementation
4. making the code changes
5. opening a GitHub PR linked to the Notion issue
6. relying on GitHub Actions for default validation unless the user explicitly requests local validation

## Steps
### Step 1: Read and understand the issue
Before making changes:
- Read the issue title, description, comments, and any linked context.
- If this issue is in a project, review the project description and all project ducoments.
- Identify the expected outcome, constraints, and acceptance criteria.
- If the issue is ambiguous, state the ambiguity clearly in a Notion comment before proceeding.
- Do not start coding until the scope is understood well enough to produce a concrete plan.

### Step 2: Update the Notion issue
When starting work:
- Change the issue status property to `In progress`.
- Add a comment that:
  - you have started working on it,
  - you are creating a branch/worktree,
  - you will follow up with implementation progress and a PR link.

### Step 3: Create a git worktree and branch
Create a dedicated worktree under:

`~/dev/worktrees/<repo>/<branch>`

Branch rules:
- Base the branch on `dev`.
- Branch name should follow this format:

`agent/<issue-id>-<short-slug>`

Example:

`agent/KIS-7-fix-login-timeout`

Behavior:
- If the branch does not exist, create it from `dev`.
- If the branch already exists locally, reuse it.
- Move into the worktree directory before making changes.

### Step 4: Plan before implementation
Before editing code:
- Summarize the issue in your own words.
- Write an implementation plan.
- Identify the files or modules likely to be affected.
- Consider edge cases, risks, and whether tests need to be added or updated.

### Step 5: Implement the change
- Make the smallest set of code changes necessary to solve the issue.
- Follow the repository’s existing conventions and patterns.
- Avoid unrelated refactors unless they are required to complete the issue safely.
- If you discover the issue is larger than expected, leave a Notion comment describing the new scope before continuing too far.

### Step 6: Prepare for PR
Before opening a PR:
- Review the diff for correctness and unnecessary changes.
- Do NOT run test, lint, build, or typecheck
- Prefer to rely on the repository’s GitHub Actions / CI to perform full validation after the PR is opened.

### Step 7: Commit and open a PR
Create a commit and open a PR with GitHub CLI.
For any code changes, you MUST create a PR without asking.

PR requirements:
- Push the branch to origin.
- PR title should clearly describe the change.
- PR should merge to `dev` branch.
- The first line of the PR description must be:

`closes <issue_id>`

Example:

`closes KIS-7`

After the first line:
- explain the problem,
- describe the implementation,
- summarize validation performed,
- explicitly note that full validation is expected to run in GitHub Actions / CI,
- mention any known limitations or follow-up work.

### Step 8: Check GitHub Actions results
After creating the PR, the test workflow will run automatically. Wait for it to complete (typically under 3 minutes), then review the results.

If any checks fail, make only the minimal changes required to fix the issue, update the branch, and push new commits. Summarize what was fixed.

Each new commit triggers another workflow run. Repeat this process until all tests pass.

### Step 9: Update Notion with PR and follow-up status
After opening a PR OR after any significant PR update (e.g. CI failure fix, CI passing, PR ready):

Add a comment to the Notion issue containing:
  - a short summary of what changed,
  - current PR status (e.g. CI failed, CI passed, ready to merge),
  - whether validation is handled or completed by GitHub Actions / CI,
  - any remaining caveats.

Notion Status: change to 'Reviewing'

# Failure or blocker handling
If you are blocked, add a Notion comment explaining:
  - what blocked you
  - what you tried
  - what is needed to proceed

If you cannot complete the issue safely, do not pretend it is done
