---
name: yistc-linear-issue
description: Workflow for handling a Linear issue end-to-end. Uupdate Linear, create a git worktree from dev, implement the change, verify it, and open a GitHub PR linked to the issue.
---

# Workflow for working on a Linear issue
## Scope
Use this workflow only when you are explicitly assigned a Linear issue, with an issue ID like `L-114`.

## Linear MCP References
- Get Issue - get_issue (MCP)(id: "L-115")
- List Comments - list_comments (MCP)(issueId: "L-115")

## Description
You will be assigned an issue with issue ID such as 'L-114'.
You need to strictly follow the steps listed below. You may consider each step as a task.

## Goal
Complete the issue by:
1. updating the Linear issue
2. creating a dedicated git worktree and branch
3. planning before implementation
4. making the code changes
5. verifying the result
6. opening a GitHub PR linked to the Linear issue

## Steps
### Step 1: Read and understand the issue
Before making changes:
- Read the issue title, description, comments, and any linked context.
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

`~/Developer/worktrees/<repo>/<branch>`

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

### Step 6: Verify the change
Before opening a PR:
- Review the diff for correctness and unnecessary changes.
- Run the most relevant validation commands available in the project, such as:
  - tests
  - lint
  - build (only if necessary)
- Do NOT run build if the user explicitly specifies that build should not be executed.
- Do NOT run typecheck by default.
  - Only run typecheck if it is necessary to validate the correctness of your changes
    (e.g. type-related changes, schema changes, or TypeScript errors are likely).
- If full validation is not performed:
  - explicitly state what was skipped,
  - explain why it was skipped.
- Confirm that the change actually addresses the issue’s acceptance criteria.

### Step 7: Commit and open a PR
Create a commit and open a PR with GitHub CLI.

PR requirements:
- Push the branch to origin.
- PR title should clearly describe the change.
- The first line of the PR description must be:

`closes <issue_id>`

Example:

`closes L-114`

After the first line:
- explain the problem,
- describe the implementation,
- summarize validation performed,
- mention any known limitations or follow-up work.

### Step 8: Update Linear with the PR
After opening the PR, add a comment to the Linear issue containing:
  - a short summary of what was changed,
  - the validation performed
  - any remaining caveat

## Failure or blocker handling
If you are blocked, add a Linear comment explaining:
  - what blocked you
  - what you tried
  - what is needed to proceed

  If you cannot complete the issue safely, do not pretend it is done.

## Cleanup
Only perform cleanup when explicitly requested by the user

Cleanup may include:
- removing the git worktree
- deleting the local branch
- posting a final Linear comment summarizing:
  - the problem
  - the plan
  - the result
  - the PR status or merge status
