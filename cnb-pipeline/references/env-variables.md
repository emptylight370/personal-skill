# CNB environment variables

> **The full variable list always follows the official docs** — fetch it live per `doc-index.md` (built-in read-only variables, and which page covers declaration/import/export). This file only records **mechanisms and syntax**; it maintains no variable table and no docs path table — the variable list evolves with the platform, and a copied list only misleads.

## Declaration and import

```yaml
- env:                           # Pipeline/Job level
    MY_VAR: value
  imports:                       # import secret repos (independent secret repo referenced via absolute https path)
    - https://cnb.cool/<org>/<secret-repo>/-/blob/main/secrets.yml
    - $NEXT_FILE                 # variables from earlier files are visible to later ones
```

When `env` and `imports` keys collide, `env` wins; cross-Stage passing uses `exports` (below).

> **Secret repo references**: sensitive values (tokens, passwords, webhook urls) belong in an independent secret repo referenced via absolute https path: `https://cnb.cool/<org>/<secret-repo>/-/blob/main/<file>.yml`. On permission-check problems, fetch the official file-reference doc per `doc-index.md`.

---

## Export and passing (cross-Stage)

### Method 1: parse from script output

```yaml
- name: set-version
  script: echo "##[set-output VERSION=1.0.0]"
  exports:
    VERSION: MY_VERSION          # from-key: to-key
```

### Method 2: export from execution result

```yaml
- name: run-script
  script: echo -n "hello"
  exports:
    info: SCRIPT_OUTPUT          # info/code/stdout/stderr/skip
```

### Method 3: export from a built-in task

```yaml
- name: release
  type: git:release
  options: { tag: v1.0.0 }
  exports:
    version: RELEASE_VERSION
```

---

## set-output encoding

`##[set-output key=value]` supports these encoding prefixes (for values with newlines or other special characters):

- `##[set-output key=base64,<base64-encoded-value>]`
- `##[set-output key=escape<escaped-value>]`

The length limit for variable values follows the official env docs.

---

## Variable substitution

The following fields support `$VAR` substitution:

- `env`, `imports`
- `options` / `optionsFrom`
- `settings` / `settingsFrom`
- `docker` (image / build / volumes)
- `stage.image`
- `ifModify`, `name`
- `lock.key`, `allowFailure`
- `runner.tags`

Use `\$` to prevent substitution.

---

## The most misused ones

The full table goes to the docs; these are the ones easiest to trip on when reading config:

- `CNB_PIPELINE_KEY`: for object-form pipelines it is the pipeline key (e.g. `frontend`), for array form the auto index (e.g. `pipeline-0`);
  prefer it when associating config by key. `CNB_PIPELINE_NAME` is empty when no `name` is declared.
- `CNB_PIPELINE_STATUS` / `CNB_BUILD_FAILED_MSG`: only hold values in `endStages` / `failStages` respectively.
- `CNB_TOKEN`: build-time temporary token; permissions are restricted on PR events — put sensitive operations on `push`/`pull_request.target`/`tag_push`.
- The `CNB_` prefix is reserved by the system; never define a custom variable with it. Referencing a non-builtin `CNB_` variable is usually a typo.

## How to look up built-in variables

1. **Need the full table** → fetch the official built-in-variables doc per `doc-index.md`: categorized by base / commit / repo / build / merge / cloud-native dev / Issue / comment / NPC, each marked with its available events.
2. **Need to judge one variable** → read its entry on that page; anything about **in which phase it holds values** (e.g. final/failure stages) must not be guessed.
3. **Need to confirm whether some `CNB_*` is built-in** → fetch the docs first; if it's not there, it's basically a typo.

Two commonly miswritten categories:

- **Phase-related**: variables that only hold values in `endStages` / `failStages` read as empty inside `stages`.
- **Event-related**: PR-type, Issue-type, comment-type, and NPC-type variables are only injected under their matching events.

> This skill ships no validation script; **the built-in variable list always follows the official env docs**; when the docs have moved on, fix this file's outdated wording.

---

## Naming rules

- Only letters, digits, underscores
- Cannot start with a digit
- Case-sensitive
- **Never use the `CNB_` prefix**: all `CNB_`-prefixed variables are system built-ins and read-only; custom variables use another prefix
