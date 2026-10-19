---
name: cnb-pipeline
description: Write, modify, and review .cnb.yml pipeline configuration for the CNB cloud-native build platform, covering branch/event triggers, pipeline/stage/job structure, built-in tasks, plugin jobs, environment variables, caching, crontab schedules, and self-hosted runner scheduling. Trigger words: CNB pipeline, write .cnb.yml, .cnb.yml config, cloud-native build, pipeline syntax, pipeline YAML, 流水线配置, 云原生构建.
---

# CNB Pipeline Configuration

> When the project has its own conventions in `.cnb.yml`, `CONTRIBUTING`, or other explicit docs, those take precedence.

## Source of truth (important)

**Never write platform facts from memory.** Field names, built-in task parameters, event applicability, variable lists, runner specs, and plugin usage all change with the platform. Fetch the official documentation live: find the right page in `references/doc-index.md` and fetch it from `docs.cnb.cool`.

This file only keeps judgments nothing else can replace (criteria live here; patterns, syntax details, plugin workflow, and per-topic facts live in `references/` and are read on demand — the "Mode routing" table says when to read each):

- **Look before you write**: fetch the relevant docs page before writing any field or event name.
- **Where errors come from**: when config and docs disagree, re-check the official doc first, then re-read the matching reference file.
- **Docs win**: when config and docs conflict, trust the docs; fix outdated wording in these files while you are at it.
- **Example values are placeholders**: image names, architecture tags, plugin names, and built-in task parameters in examples are placeholders, not conventions. Verify against the docs before writing them down.

## Mode routing

Locate the workflow and the reference file to read; the trunk keeps only criteria, operation details live in the reference files:

| Trigger | Where to go |
|---|---|
| Write/modify a pipeline | "Configuration workflow" (trunk) |
| Crontab schedule config and sync behavior | `references/crontab.md` |
| Self-hosted runner scheduling config | "Self-hosted runners" (trunk) → `references/self-hosted-runner.md` |
| Write a `type` built-in task | `references/builtin-tasks.md` |
| Write `env`/`imports`/`exports`/variable substitution | `references/env-variables.md` |
| Field pitfalls, structure skeleton, three stage chains | `references/syntax-reference.md` |
| Search the plugin market, generate plugin job config | `references/plugin-search.md` |
| Reuse, multi-module orchestration, ready-made patterns | `references/best-practices.md` |
| Which official docs page to fetch for a fact | `references/doc-index.md` |

Reference files are read on demand: only the writing step needs them; a quick simple pipeline does not.

## Configuration workflow

Run in order (steps 1–4 are "look before writing", step 5 writes, step 6 cross-checks):

1. **Ask for requirements** — trigger branches, events, build language/environment, build steps, special needs. Generate directly when the information is sufficient.
2. **Read the current state** — in modify scenarios, read `.cnb.yml` and the `.ci/` directory first.
3. **Fetch docs** — event names, fields, variables, and runner specs are platform facts: fetch the matching page per `references/doc-index.md`. When unsure, fetch; never write from memory.
4. **Verify plugins** — for plugin jobs (`image` + `settings`), follow `references/plugin-search.md`: search the plugin market → read the plugin README → generate the snippet. `settings` parameters follow the plugin README; **never generate them from memory**.
5. **Generate from the skeleton** — produce a complete runnable config per `references/syntax-reference.md` "Structure skeleton".
6. **Cross-check against docs** — after every generation/modification, verify item by item (see below).
7. **Explain the key points** — briefly explain the important parts.

### Cross-check against docs

This skill ships **no validation script**; cross-checking always goes through the official docs, never a local tool.

After generating/modifying, verify each row; every item must have a docs source:

| Check | How |
|---|---|
| Structure | Field names, nesting, indentation match the official grammar manual (`.cnb.yml` uses spaces for indentation) |
| Events | Event names exist and are applicable to the built-in task (check the task page's "applicable events" section) |
| Variables | Built-in variable spelling and in which event/phase they hold values (check the env docs) |
| Plugins | `settings` parameter names and requiredness match the plugin README |
| Skeleton | Compare with `references/syntax-reference.md` "Structure skeleton"; don't drop structural fields like `version`/`main` |

When a check fails, trace back to the failing layer:

1. **Syntax** → YAML indentation uses spaces, quoting, special-character escaping
2. **Semantics** → field/variable/event semantics: re-check the official doc, then re-read the matching reference file
3. **Schema** → field spelling, types, required fields, against the official grammar manual

> Rule tables inside the reference files (built-in variable lists, built-in task applicable events) are **writing aids, not authority**; they inevitably lag behind the platform. **When they disagree with the docs, the docs win** — and fix the outdated wording in the reference file while you are at it.

## Self-hosted runners

Self-hosted runners add your own machines as build nodes for scenarios official nodes don't cover (iOS builds, Windows packaging, Linux ARM). Config-only essentials:

- Scheduling requires **both** `runner.namespace: group` **and** `runner.tags` (missing either raises `RUNNER_TAGS_REQUIRED`); tags match in "AND" relation.
- On Mac/Windows nodes the platform-level Docker build environment is off (`docker.image` does not apply); on Docker-less Linux nodes `runner.cpus` is ignored.

Full config constraints and per-platform script patterns: `references/self-hosted-runner.md`.

## Constraint quick list

Cross-file red lines. Field-level pitfalls live in `references/syntax-reference.md`; variables and reuse details live in their own files.

**Syntax**

- **Indent with spaces**: YAML indentation uses spaces, not tabs; branch names with special characters need quoting, e.g. `"feature/*"`

**Semantics**

- **Ordering**: Stages run in order within a Pipeline; the `jobs` array within a Stage is serial
- **Secrets scoped nearby**: `imports` at Pipeline level applies to all stages; at Stage level only to that Stage — reference secrets at Stage level when only one Stage needs them
- **Repo-level events live under `$`**: `issue.*`/`tag_push`/`tag_deploy.*`/`vscode`/`auto_tag` are unrelated to branch names and only go under the `$` fallback branch; `crontab` must sit under an explicit branch name; `issue.comment@npc`/`pull_request.comment@npc` go under `$` or an exact role name (role names must match `npc.roles[].name` exactly, no glob)
- **Built-in task events**: some built-in tasks only work under specific events; check the task's "applicable events" section before writing `type`

**Security**

- **Sensitive operations go on push**: `CNB_TOKEN` has restricted permissions on PR events; put sensitive operations on `push`/`pull_request.target`/`tag_push`
