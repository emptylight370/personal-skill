# Plugin jobs: search and config generation

A CNB plugin is "a packaged Docker image + a `settings` parameter interface" and covers most CI/CD scenarios. This file expands step 4 of the configuration workflow ("verify plugins"): **search the market first, then generate `image` + `settings` from the plugin README**.

## When a plugin search is mandatory

In these scenarios **search the plugin market first**, don't hand-write scripts:

- **Notifications**: push build status/results to IM group chats or webhooks
- **Code quality**: lint, formatting, AI review, commit/PR convention checks
- **Build and packaging**: build/merge Docker images, or use market build-tool plugins (Maven, Gradle, Kaniko, etc.; plain code build/test prefers script jobs — see "Plugin job vs script job" below)
- **Release and versioning**: auto tag, changelog, semantic release
- **Deploy and ops**: deploy to container platforms/Serverless/edge, run SSH/kubectl commands
- **Security and scanning**: dependency vulnerability scans, security audits
- **Git collaboration**: automated PR creation or Git operations
- **Files and storage**: COS upload/download, environment variable substitution
- **WeChat mini-program CI**: mini-program build and upload

> **Core rules**:
> - **Shape is fixed**: plugin jobs use `image` + `settings`.
> - **Scripts as fallback**: only when no matching plugin exists or the need is trivial, fall back to a handwritten `script`.
> - **Verify parameters**: a plugin's usage (`settings` names, types, requiredness) follows the plugin README, **never generate from memory** — same-named plugins of different versions can differ in parameters.

## Plugin job vs script job

Pipeline `Job`s come in three kinds: script, plugin, built-in (built-in: see `builtin-tasks.md`). Plugin vs script follows the official "script vs plugin" doc (see `doc-index.md`):

| Scenario | Prefer | Why |
|------|------|------|
| Notifications (WeCom/Feishu/DingTalk etc.) | Plugin job | Cohesive function, simple config |
| Generic CI/CD operations (release, deploy) | Plugin job | Mature market solutions |
| Code quality checks (lint, scanning) | Plugin job | Reusable |
| Reusable, shareable generic functions | Plugin job | Publishable to the market |
| Code build and test | Script job | Flexible control, adapts to any build tool |
| Custom business logic | Script job | Full control of execution flow |
| Strongly business-specific | Script job | Not worth reusing |

**Quick rule**: a ready-made plugin fits → plugin job; need flexible custom commands → script job; function is reusable/shareable → plugin job.

**Core differences**:
- **Execution**: script jobs use `script`/`commands`; plugin jobs run the image's built-in `ENTRYPOINT`
- **Parameter channel**: script jobs use `env`/`imports`; plugin jobs use `settings` (auto-converted to `PLUGIN_*` uppercase prefix into the container)
- **Explicit variable reference**: plugin jobs do **not** inherit `env`/`imports` variables; reference them with `$VAR` inside `settings`/`args` (see "Generate config" below)
- **Reusability**: script jobs are business-specific; plugin jobs can be shared to the market

## Mode selection

- **Search mode** (user described a need but named no plugin) → follow "Search workflow" below
- **Direct mode** (user named the plugin) → skip search, go straight to "Fetch plugin docs → Generate config"

## Search workflow

### 1. Intent analysis

Extract key information from the user's description:

- **Operation type**: build, lint, test, deploy, release, notify, approve, scan, tag, report
- **Language/framework**: Node.js, Go, Python, Java/Maven/Gradle, C++, Android, WeChat mini-program, Helm
- **Target platform/service**: WeCom, Feishu, DingTalk, TKE, EdgeOne Pages, SCF, COS, TSF, CDN, Cloudflare
- **Special needs**: AI review, security scanning, commit convention checks, multi-arch images, knowledge base

### 2. Search the plugin market

Fetch the latest plugin catalog:

**Default data source** (CNB SaaS plugin market):
```
URL: https://cnb.cool/cnb/plugins/market/-/git/raw/main/plugins/plugins.json
```

From the plugin list, find the entries most relevant to the user's need keywords; return `id`, `name`, `description`, `tags`, `readme`, `images`, `source`; sort by relevance; list the top 5.

> **Private / custom data sources**: if the SaaS source is unavailable (private deployment or self-built market),
> ask the user for the `plugins.json` content (same format as SaaS) or its URL.

Each plugin entry (example `wecom-message`):
```json
{
  "id": "tencentcom/wecom-message",
  "name": "wecom-message",
  "description": "企业微信群里发消息的插件，通过机器人，可以向固定的群中推送消息。",
  "tags": ["企业微信", "WeCom", "消息"],
  "mark": "",
  "images": "https://hub.docker.com/r/tencentcom/wecom-message/tags",
  "source": "https://cnb.cool/cnb/plugins/tencentcom/wecom-message",
  "bugs": "https://cnb.cool/cnb/plugins/tencentcom/wecom-message/-/issues",
  "logo": "logo.png",
  "readme": "https://cnb.cool/cnb/plugins/market/-/git/raw/main/plugins/tencentcom/wecom-message/README.md"
}
```

Entry fields:

- `id`: unique identifier, format `org/plugin-name`
- `name`: plugin name, English only, hyphenated
- `description`: plugin description
- `tags`: tag array used for search categorization
- `mark`: badge: `official` / `private` (specific org) / empty (community)
- `images`: Docker image address (source for the `image` config value)
- `source`: source repo address
- `bugs`: issue tracker address
- `logo`: logo relative path, optional
- `readme`: raw README address, used directly for fetching the doc

**Matching strategy** (by priority):
1. **Tag hit** — user keywords hit `tags`
2. **Description hit** — keywords hit `name` or `description`
3. **Semantic proximity** — operation type matches (e.g. "send notification" matches tags: notification/message/webhook)

### 3. Recommend and confirm

Show the top 3–5 matches and let the user pick:

```
Found matching plugins:

1. wecom-message — send messages to WeCom groups
   tags: 企业微信, WeCom, 消息, message
   image: tencentcom/wecom-message

2. feishu-message — send messages to Feishu groups
   ...

3. webhook — send build status notifications via Webhook
   ...

Recommend #1. Use it, or pick another number?
```

### 4. Fetch plugin docs

After the user confirms, fetch the plugin's README:

```
URL: <plugin readme URL>
```

Focus on the usage instructions: the `settings` parameter list, each parameter's type and meaning, and `.cnb.yml` snippets in the usage examples.

> Each `plugins.json` entry already contains a `readme` field pointing at the cached README raw address in the market repo.
>
> **README missing or insufficient**: fall back in order: `source` repo README → Docker Hub image page (`images` field) → official plugin docs per `doc-index.md`. If parameters still can't be confirmed, say so honestly and give a placeholder config marked as unverified.

### 5. Generate config

From the doc's `settings` parameters and usage examples, generate the correct YAML snippet.

**Generation rules**:

- `image` — extract the Docker image name from the `images` field (no tag, defaults to `latest`). Common forms:
  - `https://hub.docker.com/r/<org>/<name>/tags` → `<org>/<name>`
  - `https://hub.docker.com/repository/docker/<org>/<name>/tags` → `<org>/<name>`
  - `docker.cnb.cool/.../<name>` → full path
  - Other forms: use whatever image identifier is recognizable in the `images` value; when unsure, check the plugin README
  - To pin a version, append `:<version>` (e.g. `tencentcom/npm:1.2.3`) to avoid upgrade-compatibility risk
- `settings` — fill from the doc's parameter table, **parameter names stay lowercase** (CI auto-converts to `PLUGIN_` + uppercase into the plugin; no need to care when writing)
- `args` — when the plugin supports CLI arguments, pass them as an `args:` array (appended to ENTRYPOINT, like `docker run <image> <args...>`).
  - Most CNB plugins prefer `settings`
- **Variable reference mechanism (core)**: variables declared in `env` are **not passed into** plugins; reference them with `$VAR` in `settings` or `args`.
  - `settings` is the parameter whitelist
  - System built-in variables (e.g. `CNB_REPO_SLUG`) are always passed in
- **Environment variable declaration**: declare secrets inline in `env`; the `VAR` in `$VAR` must be a real variable name declared in that `env`

**Config template**:

```yaml
# In .cnb.yml (declare sensitive variables inline in env)
main:
  push:
    - env:
        MY_TOKEN: <value>      # inline secret variable
      stages:
        - name: <task-name>
          image: <image-name>    # no tag = latest
          settings:
            <paramB>: <value>
            <paramC>: $MY_TOKEN   # MY_TOKEN must be declared in the env above
```

### 6. Integrate and cross-check

After generating:

- **Merge if present**: the project already has `.cnb.yml` → locate the target Stage position and merge
- **Snippet if not**: for a standalone query → give a complete Stage/Job YAML snippet
- **Always cross-check**: after producing a full `.cnb.yml`, verify item by item per `SKILL.md` "Cross-check against docs"; fix any mismatch
- Remind the user: declare secrets inline in `env`; images default to `latest`

> **Notification plugin placement**: failure notifications go in `failStages`, success notifications in the last task of `stages`; never put "success-only notifications" in `endStages` (see `SKILL.md` "Stage execution semantics").

## Finding plugins by scenario

**No market snapshot is kept** (a snapshot always goes stale). For each scenario, run the search workflow above against `plugins.json`:

- **Notifications**: WeCom / Feishu / DingTalk group bots, Webhook push
- **Code quality**: AI code review, commit and PR title conventions, PR change-size limits, lint
- **Build and release**: npm / Maven / Gradle publishing, changelog, semantic release, multi-arch image merge, mini-program CI
- **Deploy**: container platforms, Serverless, static sites, kubectl / SSH / Terraform / Kaniko
- **Git collaboration**: auto tag, create PR, delete branch
- **Files and tools**: COS upload/download, env variable substitution

> When the scenario mentions a platform name (`TKE`, `EdgeOne Pages`, `SCF`, `COS`, `TSF`, `CDN`), use it directly as a search keyword.
> Platform-level overview (official vs community plugins, what plugins can do): fetch the official plugin docs per `doc-index.md`.

## Constraint quick list

### Config
- **Version optional**: images default to no tag (`latest`); to avoid upgrade risk append `:<version>`
- **Whitelist parameters**: `settings` is the parameter whitelist; variables declared in `env` must be referenced with `$VAR` inside `settings` to reach the plugin
- **Notifications in place**: failure notifications in `failStages`, success notifications last in `stages`

### When nothing matches
- Say so honestly. A plugin is just a Docker image, so guide the user to:
  - find a fitting image on Docker Hub or other registries and use it directly as a plugin job
  - write their own plugin (parameter design, Dockerfile, exporting variables, publishing): fetch the official plugin-authoring and contribution docs per `doc-index.md`
  - market entry: `https://cnb.cool/cnb/plugins/market`

### Who decides
- **Platform-level plugin knowledge** (market overview, plugin-vs-script choice, authoring and contribution rules) → official docs per `doc-index.md`
- **A single plugin's `settings` parameters** → that plugin's README, read directly by this workflow
