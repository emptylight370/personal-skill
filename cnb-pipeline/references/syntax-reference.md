# CNB pipeline syntax reference

> This file is a **pitfall memo**, not a field manual, and does not maintain a docs path table —
> fetch the full field/event/default-value tables live from the official docs (find the page in `doc-index.md`).
> It only records "this is easy to misconfigure, and here is what happens when you do".

## Structure skeleton

Config hierarchy: **branch → event → Pipeline → Stage → Job**. Fetch the full field table from the official grammar manual (`doc-index.md`).
The skeleton only shows the shape; event names, runner specs, plugin parameters, and other values are always fetched live from the docs:

```yaml
<branch>:                          # main / "feature/*" / "$"(fallback)
  <event>:                         # event names and applicability: fetch the trigger-rule doc
    - name: <pipeline-name>
      # runner:                    # optional; default cores and available specs: fetch the build-node doc
      #   cpus: 8
      #   namespace: group         # self-hosted runner: must be set together with tags (see self-hosted-runner.md)
      #   tags: [mac, arm64]
      docker:
        image: node:20             # or build (Dockerfile) or devcontainer
      services:
        - docker                   # Docker-in-Docker
      env:
        KEY: value
      imports:
        - https://cnb.cool/<org>/<secret-repo>/-/blob/main/secrets.yml
      stages:
        - name: step1
          script: echo hello       # script job
        - name: step2
          type: <category:task>    # built-in task, e.g. cnb:trigger
          options: { ... }
        - name: step3
          image: plugins/docker    # plugin job
          settings: { ... }
      failStages: [...]            # runs when stages fail
      endStages: [...]             # after prepare succeeds, runs on both success and failure
```

### Stage execution semantics (three chains)

- **`stages` (main chain)**: runs in order, any failure aborts and jumps to `failStages`; **affects pipeline status**. For build, test, deploy.
- **`failStages` (failure chain)**: runs only when `stages` failed; **affects pipeline status**. For failure notification, rollback, cleanup.
- **`endStages` (final chain)**: after `prepare` succeeds, runs regardless of `stages` outcome (after `stages`/`failStages`);
  **does not affect final status**. For temp-file cleanup, log archiving.

To send different messages on success and failure: failure notification goes in `failStages`, success notification in the last task of `stages` (only reached when everything before succeeded).
**Do not** put "success-only notifications" in `endStages` — it also runs on failure, and the user gets both a red and a green notification.

### Pipeline field pitfalls

- **`runner` usually needs no config**: default is 8 cores / 16G; write it only when pinning architecture or core count.
  Node tags and core ranges follow the official docs (they change; don't hardcode remembered values).
- **`runner.namespace: group` and `runner.tags` must be configured together** — see `self-hosted-runner.md`.
- **`docker` is one of three**: `image` / `build` (Dockerfile, auto-cached) / `devcontainer`;
  if omitted, the platform default image applies — the default value follows the official docs.
- **`lock`**: `wait` decides queue-vs-fail; `cancel-in-progress` and `cancel-in-wait`
  cancel "running" and "queued" same-key pipelines respectively — deploy scenarios usually want both.
- **`imports`**: secret repos use absolute https paths; Pipeline level applies to all stages, Stage level only to the current Stage.

### Stage field pitfalls

- `image` **only overrides the image**, not other build-environment forms like `build`/`devcontainer`.
- `retry`, `timeout`, `allowFailure`, `ifModify`, `ifNewBranch`, `if` can all be written on a Stage;
  `ifModify` and `allowFailure` support variable substitution.
- **`scripts` is not `script`**: script jobs use `script` (single line or multiline); put multiple commands in one `script`.

### jobs syntax

Within one Stage, `jobs` can be an **array** (each item has `name` + job body) or an **object** (the key is the job name).
Both forms are syntactically valid; their execution order follows the official grammar manual.

> **Job parallelism is not a recommended pattern**: this skill does not provide parallel configs;
> when parallelism is needed, confirm the bottleneck first, then write it per the official docs.

### include and !reference

- **Merge rules**: arrays append, same-named object keys are overwritten by the later one, local wins (details in the official simplified-config doc).
- **The referenced source key starts with `.`** (e.g. `.docker-config`), otherwise it is treated as a branch name.
- **`!reference` takes values only**, it does not support `<<:` merge expansion; merge expansion within one file uses YAML anchors `&`/`*`.
- `!reference` supports multi-level paths: `!reference [.wecom, warrior_robot]`.
- Practical patterns (`.ci/` split, template inheritance, cross-repo include) are in `best-practices.md`, sections "Reuse" and "Multi-module orchestration and Monorepo".

### Volume caching

`docker.volumes` supports copy-on-write / read-write / shared-within-pipeline / read-only modes.
**The exact mode suffixes and concurrency behavior follow the official caching doc** — a wrong volume type only shows up under concurrent pipelines,
and is easily misdiagnosed as a "flaky failure".

### Variable substitution

The list of fields supporting `$VAR` is in `env-variables.md` "Variable substitution"; use `\$` to prevent substitution.
