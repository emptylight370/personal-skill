# CNB built-in tasks reference

**Task lists, parameters, and applicable events always follow the official docs** (they change with the platform); fetch them live per `doc-index.md`. This file keeps **only the usage skeleton and judgment points** — it does not restate parameter tables or copy the doc directory (a copy is just stale data waiting to happen).

## Usage skeleton

Built-in tasks are declared with `type` (dotted name `<category>:<task>`), take `options`, and export results as environment variables via `exports`:

```yaml
- name: my-task
  type: <type>
  options:
    key: value
  # optionsFrom: options.yml     # load options from a file
  exports:
    output_key: ENV_VAR_NAME
```

Judgment points:

- **Dotted names are not commands**: `xxx:yyy` are all built-in task types (`npc:go`, `cnb:apply`, `vscode:go`…).
  Their meaning follows the official docs — **never guess from the literal name**.
- **Check applicable events first**: some tasks only take effect under specific events; the wrong event fails the build or does nothing.
  Before writing `type`, fetch that task's doc page and confirm the current event is listed.
- **Prefer `exports` over parsing logs**: get task results via `exports`, don't grep logs.

## Common tasks: fetch before writing

For the common built-in tasks (trigger other repos `cnb:trigger`, trigger sub-pipelines `cnb:apply`, multi-pipeline orchestration
`cnb:await`/`cnb:resolve`, cache docker builds `docker:cache`, auto-merge `git:auto-merge`, release `git:release`,
reviewers `git:reviewer`, Issue/PR updates, coverage reporting, TAPD, `vscode:go`, `npc:go`…),
**the purpose can be guessed from the name; the parameters and applicable events cannot** —
fetch the task's doc page per `doc-index.md` before writing `type`.

Judgment order:

1. First confirm the task is valid under the current event (the task page's "applicable events" section is the answer).
   For prefixed events like `api_trigger_*` / `web_trigger_*` / `tag_deploy.*`, judge by the docs' description of
   `api_trigger` / `web_trigger` / `tag_deploy`.
2. Then confirm `options` parameter names and requiredness.
3. If unsure, fetch first — never probe by writing.

> This skill ships no validation script; **event availability always follows the official task page's "applicable events"**.
> Never judge from memory or old notes; when the docs contradict this file, fix this file.

## npc:go essentials

The entry point for running AI tasks inside a pipeline; supports all events. **Parameters and usage follow the official `npc:go` and NPC docs** (model, prompt, image options are still evolving — fetch the page per `doc-index.md`, don't copy old parameters). Only the easy-to-get-wrong points:

- **Prompt source depends on the trigger**: comment triggers (`issue.comment@npc` / `pull_request.comment@npc`)
  get the prompt injected by the platform from the role definition in `.cnb/settings.yml`; prompts configured in `options` are ignored.
  Other triggers (`api_trigger`, `push`, `crontab`, manual) require a prompt in `options`,
  and **user-input parameters must be provided** — a role name alone is rejected.
- **Role names match exactly**: comment-trigger role names must exactly equal `npc.roles[].name`, no glob.
- **Permissions follow the user**: the NPC's `CNB_TOKEN` ceiling is the triggering user's repo role; low-privilege roles don't get work mode
  (work mode = the NPC may autonomously write code, push branches, open PRs; enabled via the "work for me" switch in the comment area).
- **Runs in a specified image**: default `cnbcool/default-npc:latest`; swap in your own CLI/Skills image via `docker.image`.

Failure-notification scenarios usually pair with `failStages`; the placement rules for success/failure notifications are in `SKILL.md` "Stage execution semantics".
