# Official docs index (fetch live)

All platform facts come from the official docs at `https://docs.cnb.cool`. Fetch the paths below with a web fetch (`.md` gives raw markdown, `.html` the rendered page — same content).

**Fallback rule**: if a path no longer resolves, fetch `https://docs.cnb.cool/zh/llms.txt` — the full official doc index with titles and paths — and relocate the page there. Never guess a path; never write a fact from memory.

## Grammar and structure

| Fact needed | Docs path |
|---|---|
| Full grammar: field reference, pipeline/stage/job fields, three stage chains | `/zh/build/grammar.md` |
| Config file overview, `main` entry, pipeline forms (array/object) | `/zh/build/configuration.md` |
| Trigger events, event list, branch glob matching | `/zh/build/trigger-rule.md` |
| Build environment: `docker.image` / `docker.build` / `devcontainer` | `/zh/build/build-env.md` |
| Timeout policy (`timeout`) | `/zh/build/timeout.md` |
| Simplified config: `include`, anchors, merge rules | `/zh/build/simplify-configuration.md` |
| File reference `!reference`, cross-file reuse | `/zh/build/file-reference.md` |
| Script job vs plugin job | `/zh/build/script-vs-plugin.md` |
| Deploy pipelines | `/zh/build/deploy.md` |
| Manual trigger (`.cnb/web_trigger.yml`) | `/zh/build/web-trigger.md` |

## Nodes, variables, caching

| Fact needed | Docs path |
|---|---|
| Runner nodes: `runner.cpus`, `runner.tags`, concurrency/time limits | `/zh/build/build-node.md` |
| Environment variables: declaration, substitution, scoping | `/zh/build/env.md` |
| Built-in read-only variables (`CNB_*`) with per-event availability | `/zh/build/build-in-env.md` |
| Volume cache / image cache (`docker.volumes`, `docker:cache`) | `/zh/build/pipeline-cache.md` |
| Crontab schedules | `/zh/build/crontab.md` |

## Built-in tasks (`type: ...`)

| Fact needed | Docs path |
|---|---|
| Built-in task index (all `type` names) | `/zh/build/internal-steps.md` |
| A specific task's parameters and applicable events | `/zh/build/internal-steps/<vendor>/<task>.md` — resolve the exact path from the index above |

Common task pages (verify paths against the index before fetching):

- `cnb:trigger` → `/zh/build/internal-steps/cnb/trigger.md`
- `cnb:apply` → `/zh/build/internal-steps/cnb/apply.md`
- `cnb:await` / `cnb:resolve` → `/zh/build/internal-steps/cnb/await-resolve.md`
- `docker:cache` → `/zh/build/internal-steps/docker/cache.md`
- `git:release` → `/zh/build/internal-steps/git/release.md`
- `git:auto-merge` → `/zh/build/internal-steps/git/auto-merge.md`
- `git:reviewer` → `/zh/build/internal-steps/git/reviewer.md`
- `testing:coverage` → `/zh/build/internal-steps/testing/coverage.md`
- `npc:go` → `/zh/build/internal-steps/npc/go.md`

## Plugins

| Fact needed | Docs path |
|---|---|
| Plugin market overview | `/zh/plugins.md` |
| Plugin authoring (`ENTRYPOINT`, exporting variables) | `/zh/build/create-plugin.md` |
| Contributing a plugin to the market | `/zh/build/contribute-plugin.md` |

Individual plugin `settings` parameters come from the plugin's own README, not these pages (see `plugin-search.md`).
