---
name: yaak-cli
description: Use when working with the Yaak CLI and a shared Yaak GUI workspace to inspect workspaces, folders, requests, environments, and safely send API requests without exposing inherited auth tokens.
---

# Yaak CLI

Use this skill when the user mentions Yaak or `yaak`, or when you need to inspect or send requests stored in the shared Yaak workspace.

## Safety Rules

- Treat Yaak GUI and CLI as sharing the same saved workspace data.
- Do not inspect, print, or expand authentication tokens from requests, folders, or inherited parent folders unless the user explicitly asks.
- Do not use `yaak -v` or `--verbose`. Verbose mode prints request headers.
- Assume inherited auth on parent folders is correct unless the user explicitly asks you to debug auth.
- Never run raw `yaak request show <request_id>` or raw `yaak folder show <folder_id>` in user-facing work when auth may be present. Always sanitize those outputs with `jq`.
- If you need to inspect a request definition, sanitize auth fields:

```bash
yaak request show <request_id> | jq 'del(.authentication, .authenticationType)'
```

If you need to inspect a folder definition, sanitize auth fields:

```bash
yaak folder show <folder_id> | jq 'del(.authentication, .authenticationType)'
```

Prohibited by default:

- `yaak -v send <request_id>`
- `yaak request show <request_id>` without `jq`
- `yaak folder show <folder_id>` without `jq`
- any workflow that tries to reveal inherited bearer tokens just to confirm they exist

## Discovery Flow

Start with:

```bash
yaak workspace list
yaak folder list <workspace_id>
yaak request list <workspace_id>
yaak environment list <workspace_id>
```

Useful detail commands:

```bash
yaak workspace show <workspace_id>
yaak folder show <folder_id> | jq 'del(.authentication, .authenticationType)'
yaak request show <request_id> | jq 'del(.authentication, .authenticationType)'
yaak environment show <environment_id>
```

## Safe Send Workflow

Prefer non-verbose sends and explicitly pick the environment when variables are involved.

```bash
yaak -e <environment_id> send <request_id>
```

Observed behavior:

- non-verbose `yaak send` prints the response body only
- `yaak -v send` prints request headers and must be avoided around real tokens

When validating CLI control, prefer this order:

1. read-only unauthenticated request
2. read-only authenticated request with `-e <environment_id>`
3. negative-case request expected to return `400` or `404`

Use read-only requests first when validating CLI control, such as `GET` list or `GET` fetch requests. Avoid sending folder or workspace IDs unless the user explicitly wants batch execution.

## Common Commands

List and inspect:

```bash
yaak workspace list
yaak request list <workspace_id>
yaak folder list <workspace_id>
yaak environment list <workspace_id>
```

Create resources:

```bash
yaak workspace create --name "My Workspace"
yaak request create <workspace_id> --name "Ping" --method GET --url "https://example.com"
yaak environment create <workspace_id> --name "local"
```

Schemas for automation:

```bash
yaak workspace schema
yaak environment schema
yaak request schema http
```

## Notes

- Template variable syntax is `${[ my_var ]}`, not `{{ my_var }}`
- Requests can inherit data from parent folders, including auth and variables
- `yaak send <id>` accepts request IDs, folder IDs, and workspace IDs
- For manual API coverage reviews, compare route definitions in code with `yaak request list <workspace_id>` and identify missing endpoints or missing negative-case requests
