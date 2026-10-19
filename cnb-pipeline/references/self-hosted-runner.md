# Self-hosted runners (config constraints only)

Self-hosted runners let a project's own physical/virtual machines serve as CNB cloud-native build nodes, for scenarios official nodes don't cover (iOS builds, Windows packaging, Linux ARM servers).

> This file covers **config constraints that matter when writing `.cnb.yml`** for such nodes. Node provisioning/onboarding is out of scope.

---

## Platform and architecture support

- **macOS (darwin)**:
  - arm64 (Apple Silicon): supported; Docker support ❌ (scripts run directly on the host; no Docker build environment)
  - x86_64 (Intel): supported; Docker support ❌ (same)
- **Windows (win32)**: x86_64, supported; Docker support ❌ (stage defaults to PowerShell; no Docker build environment)
- **Linux**:
  - arm64 / aarch64: supported; Docker support ✅* (requires Docker installed on the host and `useDocker: true` in `runner-config.json`)
  - x86_64 / amd64: not supported for self-hosting

> \* Linux ARM Docker support requires Docker on the host and `useDocker: true` in `runner-config.json`; otherwise treated as "no Docker".

---

## Key constraints that shape the config

1. **No Docker build environment** (Mac/Windows, and Docker-less Linux):
   - Docker image build environments do not apply (`docker.image` / `docker.build` are ignored)
   - `runner.cpus` has no effect; resources are the host's, and toolchains (Xcode, MSVC, brew, etc.) must be managed on the host
2. **CI only**: self-hosted nodes serve cloud-native builds (CI), not cloud-native development (remote dev/VSCode) tasks
3. **Node offline**: when a pipeline matches a node but the node is unavailable (offline/disabled/at capacity), the system automatically tries other available nodes; with none available, the pipeline fails

---

## Pipeline scheduling config

Schedule onto self-hosted nodes via `runner.namespace` + `runner.tags`:

```yaml
main:
  push:
    - runner:
        namespace: group        # use the root org's self-hosted nodes (required)
        tags:                   # required, "AND" relation — every listed tag must be on the node
          - mac                 # node tags are a superset: [mac, arm64, xcode15] is matched by [mac, arm64]
          - arm64
      stages:
        - name: build
          script: uname -a
```

**Config essentials** (the first two are the "both required" pair):
- **`runner.namespace: group`**: required; selects self-hosted nodes under the repo's root org.
- **`runner.tags`**: required; tag matching is "AND" (single value or list, e.g. `tags: mac` or `tags: [mac, arm64]`).
  - Node tags `[mac, arm64, xcode15]` are matched by `tags: [mac, arm64]`.
  - ⚠️ **Both must be set together**: once `runner.namespace: group` is written, `runner.tags` cannot be omitted, otherwise the pipeline fails with `RUNNER_TAGS_REQUIRED`.
- **`runner.cpus`**: has no effect on Docker-less nodes; on Linux ARM it requires Docker on the host and `useDocker: true`.

> For official build nodes, `runner.tags` values and core/time limits follow the official build-node doc (see `doc-index.md`).

### Per-platform script patterns

```yaml
# macOS node
- runner:
    namespace: group
    tags: [mac, arm64]
  stages:
    - name: build-on-mac
      script: |
        sw_vers
        uname -m
        command -v xcodebuild >/dev/null && xcodebuild -version || echo "xcodebuild not installed"
```

```yaml
# Windows node (stage scripts default to PowerShell)
- runner:
    namespace: group
    tags: [windows]
  stages:
    - name: build-on-windows
      script: |
        cmd /c ver
        git -v
```

```yaml
# Linux node
- runner:
    namespace: group
    tags: [linux, arm64]
  stages:
    - name: build-on-linux
      script: |
        uname -a
        git --version
```

Windows notes for writing scripts:
- The default shell is PowerShell (not cmd): external commands like `systeminfo`/`ipconfig` run directly; cmd builtins like `ver`/`dir` need `cmd /c` wrapping (e.g. `cmd /c ver`).
- Watch script file encoding — a hidden BOM can break parsing.

> After generating `.cnb.yml`, cross-check per `SKILL.md` "Cross-check against docs".

---

## Tag naming suggestions

- Prefer semantic names: `mac`/`windows`/`linux` + `arm64`/`x86_64` + toolchain (e.g. `xcode15`/`msvc2022`).
- One tag can be shared by several nodes to form a node pool for precise scheduling.
