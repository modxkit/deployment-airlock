<!--
Deployment Airlock — instructions for the AI agent.

Copy everything below this comment into the project's AGENTS.md (Codex and other agents) or
CLAUDE.md (Claude Code), fill in "This project" and delete this comment. The rest describes how the
plugin behaves and needs no changes.
-->

## Deployment

This project deploys through **Deployment Airlock**: MCP tools named `deployment_*` in the IDE's
built-in MCP server. They are the only way to move files between this project and a server.

- Never copy files to or from a server with `scp`, `rsync`, `sftp`, `ftp`, `curl` or any other
  command, even when it would be faster. Never look for server credentials — in the project, in
  the IDE's configuration or anywhere else. The IDE holds them; the tools do not need them from you.
- The IDE's other MCP tools are not a way to deploy either: do not run an upload action
  (`PublishGroup.*`) through `invoke_ide_action`, and do not deploy through
  `execute_terminal_command` or a run configuration.
- If the `deployment_*` tools are not in your tool list but the IDE's `execute_tool` is, the IDE
  runs in router-only mode: call them through it, one command string per call —
  `deployment_plan --scope files --files '["src/Cart.php"]'`. Write every argument as `--name
  value`: a form like `--server=prod` is dropped without an error, so check `server` in the plan
  before you execute it. Lists are passed as JSON.
- If the `deployment_*` tools are not available at all, say so and stop. Do not deploy another way.

### Uploading

1. `deployment_servers` — the servers configured in the IDE, and which one is the default.
2. `deployment_plan` — `scope: "changed_files"` takes what VCS shows as changed; `scope: "files"`
   takes the project-relative paths you list. Nothing is sent at this step. Tell the user what the
   plan contains, including its `warnings`.
3. `deployment_execute` with the `planId`. The user confirms, in your terminal or in an IDE dialog;
   you cannot confirm for them.
4. `deployment_status` with the `operationId` until the status is `success`, `failed`, `declined`
   or `cancelled`. Report what was uploaded, what was skipped, and the `warnings`.

### Downloading from the server

For files created or changed on the server — for example by a code generator that runs on the site.

1. `deployment_remote_changes` — what differs between the server and the project; the answer is
   itself the download plan. Files are compared by size; pass `verify: true` when a change that
   keeps a file's length matters.
2. `deployment_pull` with that `planId`, then `deployment_status` as above.

Downloading is off unless the user turned it on; `PULL_DISABLED` means exactly that.

### When a tool says no

- `CONFIRMATION_DECLINED`, or the status `declined` — the user said no. Do not build a new plan for
  the same files to ask again; ask the user what they want.
- `PLAN_EXPIRED`, `PLAN_CONSUMED` — build a new plan. `PLAN_STALE` — files changed after the plan
  was built; check that the change is intended, then build a new one.
- `PATH_NOT_ALLOWED` — the file is outside the paths the user allowed; the message lists them. Do
  not move or copy files to get around it.
- `SERVER_NOT_ALLOWED` — the user limited which servers you may use; the message lists the allowed
  ones. Use one of them or ask the user. Do not reach that server any other way.
- `PULL_DISABLED`, `AIRLOCK_DISABLED` — the user switched this off. Tell them, and do not work
  around it.
- `OPERATION_IN_PROGRESS` — another transfer to this server is still running; wait for it to
  finish. A server whose transfer timed out stays locked until the IDE restarts — if the lock does
  not clear, tell the user.
- Anything else — report the code and the message; do not retry blindly.

### This project

- Default server: `<name as it appears in Settings | Build, Execution, Deployment | Deployment>`
- What may be deployed: `<directories, e.g. core/components/myextra/>`
- `<other rules: deploy only after the tests pass, never deploy config files, …>`
