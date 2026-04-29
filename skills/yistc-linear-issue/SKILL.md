---
name: yistc-linear-issue
description: Workflow for handling a Linear issue end-to-end. Uupdate Linear, create a git worktree from dev, implement the change, verify it, and open a GitHub PR linked to the issue.
---

# Workflow for working on a Linear issue
## Scope
Use this workflow only when you are explicitly assigned a Linear issue, with an issue ID like `L-114`.

## Guidelines
- Language use: you should use Chinese in PR and issue comment and English in title and code.

### Behavioral rules
- Do NOT change issue status unless explicitly required by a workflow step or requested by the user.
- Do NOT run local test, lint, build, or typecheck unless explicitly requested by the user.
- Always keep Linear updated with meaningful progress, especially after fixing CI failures or when the PR becomes ready to merge.

### Linear MCP References
- Get Issue - get_issue (MCP)(id: "L-115")
- List Comments - list_comments (MCP)(issueId: "L-115")
- Create Comment: save_comment (MCP)(issueId: "L-115", body: "<comment body>")
- Update commentt: save_comment (MCP)(id: "<comment_id>", issueId: "L-115", body: "<comment body>")
- Delete comment: delete_comment (MCP)(id: "<comment_id>")

### Description
You will be assigned an issue with issue ID such as 'L-114'.
You need to strictly follow the steps listed below. You may consider each step as a task.

### Goal
Complete the issue by:
1. updating the Linear issue
2. creating a dedicated git worktree and branch
3. planning before implementation
4. making the code changes
5. opening a GitHub PR linked to the Linear issue
6. relying on GitHub Actions for default validation unless the user explicitly requests local validation

## Steps
### Step 1: Read and understand the issue
Before making changes:
- Read the issue title, description, comments, and any linked context.
- If this issue is in a project, review the project description and all project ducoments.
- Identify the expected outcome, constraints, and acceptance criteria.
- If the issue is ambiguous, state the ambiguity clearly in a Linear comment before proceeding.
- Do not start coding until the scope is understood well enough to produce a concrete plan.

### Step 2: Update the Linear issue
When starting work:
- Change the issue status to `In Progress`.
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

`linear/<issue-id>-<short-slug>`

Example:

`linear/L-114-fix-login-timeout`

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
- If you discover the issue is larger than expected, leave a Linear comment describing the new scope before continuing too far.

### Step 6: Prepare for PR
Before opening a PR:
- Review the diff for correctness and unnecessary changes.
- Do NOT run test, lint, build, or typecheck
- Prefer to rely on the repository’s GitHub Actions / CI to perform full validation after the PR is opened.

### Step 7: Commit and open a PR
Create a commit and open a PR with GitHub CLI.
For any code changes, you MUST create a RP without asking.

PR requirements:
- Push the branch to origin.
- PR title should clearly describe the change.
- PR should merge to `dev` branch.
- The first line of the PR description must be:

`closes <issue_id>`

Example:

`closes L-114`

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

### Step 9: Update Linear with PR and follow-up status
After opening a PR OR after any significant PR update (e.g. CI failure fix, CI passing, PR ready):

Add a comment to the Linear issue containing:
  - a short summary of what changed,
  - current PR status (e.g. CI failed, CI passed, ready to merge),
  - whether validation is handled or completed by GitHub Actions / CI,
  - any remaining caveats.

Linear Statue: change to 'In Review'

# Failure or blocker handling
If you are blocked, add a Linear comment explaining:
  - what blocked you
  - what you tried
  - what is needed to proceed

If you cannot complete the issue safely, do not pretend it is done
