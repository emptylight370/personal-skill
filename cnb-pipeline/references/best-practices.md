# CNB pipeline best practices

> An on-demand reference of `cnb-pipeline`: **config skeletons and selection patterns**. Read when about to write one of these patterns.
> Platform facts (field names, variable lists, plugin parameters) still follow the live docs per `doc-index.md`; this file only holds ready-to-use structures and judgments.

## Sections

- [Reuse: anchors, `!reference`, and `.ci/` split](#reuse-anchors-reference-and-the-ci-split)
- [Multi-module orchestration and Monorepo](#multi-module-orchestration-and-monorepo)
- [Common config patterns](#common-config-patterns)

---

## Reuse: anchors, `!reference`, and the `.ci/` split

### Same-file reuse: YAML anchors

Keys starting with `.` are not recognized as branch names — good for reusable fragments.

```yaml
.node-env: &node-env
  docker:
    image: node:20
    volumes:
      - node_modules

.install: &install
  name: install
  script: npm ci

main:
  push:
    - <<: *node-env
      stages:
        - *install
        - name: build
          script: npm run build
  pull_request:
    - <<: *node-env
      stages:
        - *install
        - name: test
          script: npm test
```

### Cross-file reuse: `.ci/` directory + `!reference`

When config exceeds ~150 lines, split it by "function + trigger event". One file does one thing; the file name is the responsibility.

```
.cnb.yml                    # include list only
.ci/
├── .shared-config.yml      # shared config (Docker env, Go env, Node install, etc.)
├── .variables.yml          # global variables (webhook robot ID, etc.)
├── .docker-build.yml       # Docker image build
├── .pr.yml                 # PR checks
├── .push-deploy-dev.yml    # push → dev deploy
├── .deploy.yml             # production/staging deploy
├── .tag-push.yml           # tag release
└── .e2e-test.yml           # E2E tests
```

Shared config is defined with `.`-prefixed keys; business files reference it with `!reference`; anchors still work within a file and the two mix:

```yaml
# .ci/.shared-config.yml — referenced cross-file via !reference
.docker-config:
  build:
    dockerfile: .ide/Dockerfile
  volumes:
    - node_modules/.cache

.go-env: &go-env
  GOPROXY: https://goproxy.cn,direct

# .ci/.variables.yml — centralizes global constants; supports multi-level path references
.wecom:
  warrior_robot: 2dfe45f7-xxxx-xxxx

# .ci/.push-deploy-dev.yml — references the shared config
.deploy-dev: &deploy-dev
  name: deploy-dev
  docker: !reference [.docker-config]           # reference the whole object
  env: !reference [.go-env]
  lock: { key: deploy-dev, wait: true, cancel-in-wait: true }
  stages:
    - name: build
      script: npm run build
  failStages:
    - name: notify failure
      image: tencentcom/wecom-message
      settings:
        robot: !reference [.wecom, warrior_robot]  # multi-level path to a scalar
        content: "部署失败: $CNB_BUILD_ID"

main:
  push:
    deploy-dev: *deploy-dev        # same-file reference via YAML anchor
```

For template inheritance: within one file use `&anchor` + `<<: *anchor`; across files use `!reference`:

```yaml
.docker-build-base: &docker-build-base
  services:
    - docker
  docker: !reference [.docker-config]       # cross-file reference
  runner:
    cpus: 16

.docker-build-stages: &docker-build-stages
  - name: docker build
    script: make $MODULE_NAME

.frontend-build: &frontend-build
  <<: *docker-build-base                    # inherit the base template
  stages:
    - name: build frontend
      script: npm run build
    - *docker-build-stages                  # anchor reference of shared steps
```

**Merge rules**: arrays append, same-named object keys are overwritten by the later one, local wins.
**The referenced source key starts with `.`** (e.g. `.docker-config`), otherwise it is treated as a branch name; `!reference` key names must be globally unique —
prefix them when shared across files.

| Mechanism | Scope | Purpose | Syntax |
|------|---------|------|------|
| YAML anchors `&`/`*` | Within one file | Define + reference templates, supports `<<:` merge expansion | `&name` define, `*name` reference, `<<: *name` merge |
| `!reference` | Across files (via include) | Reference values (scalar/array/object), no merge expansion | `!reference [.key]` or `!reference [.key, subkey]` |
| `.`-prefixed keys | Global | Not treated as branch names, used only as reference sources | `.docker-config:`, `.wecom:` |

> `include` supports cross-repo remote references (`https://cnb.cool/<org>/<repo>/-/blob/main/x.yml`);
> business repos use `!reference` on the template's `.`-prefixed keys — **anchors do not work across files**. Merge details: official simplified-config doc (see `doc-index.md`).

---

## Multi-module orchestration and Monorepo

With multiple modules/services in one repo, combine `ifModify` + `CNB_PIPELINE_KEY` +
`cnb:resolve`/`cnb:await` for **build-on-change + orchestrated deploy**.

- `CNB_PIPELINE_KEY`: the pipeline key in object form (e.g. `frontend`), auto index in array form (e.g. `pipeline-0`)
- `ifModify`: at Pipeline or Stage level; triggers only when matching paths changed
- `cnb:resolve` / `cnb:await`: a module signals completion; the deploy pipeline waits until all modules are ready

**Template + batch mapping**: define a build template with `${CNB_PIPELINE_KEY}` referencing the module name dynamically, and expand per module by key at the trigger site:

```yaml
.build-module: &build-module
  services:
    - docker
  ifModify:
    - packages/${CNB_PIPELINE_KEY}/**    # triggers only when this module directory changed
  stages:
    - name: build
      script: cd packages/${CNB_PIPELINE_KEY} && docker build -t $IMAGE_TAG .
    - name: resolve
      type: cnb:resolve
      options:
        key: build-${CNB_PIPELINE_KEY}

main:
  push:
    user-service: *build-module          # CNB_PIPELINE_KEY = "user-service"
    order-service: *build-module
    gateway: *build-module
```

> Adding a module is one line: `new-module: *build-module`. When steps differ, use "base template + derived template" inheritance
> (`<<: *build-base`, see "Reuse" above).

**Deploy orchestration**: use `cnb:await` to wait until all modules are built, then deploy:

```yaml
.await-module: &await-module
  type: cnb:await
  options:
    key: build-${CNB_BUILD_JOB_NAME}         # CNB_BUILD_JOB_NAME = the job's key name

main:
  push:
    deploy:
      lock: { key: deploy-dev, wait: true, cancel-in-wait: true }
      stages:
        - name: await all modules
          jobs:
            frontend: *await-module          # wait for the frontend build
            backend: *await-module
        - name: deploy
          script: ./deploy.sh
```

**PR checks with on-demand waiting**: each module checks and `resolve`s independently; the summary pipeline only `await`s modules with changes —
`cnb:await`'s `ifModify` anchors the module's path rules:

```yaml
"**":
  pull_request:
    golang-lint:
      ifModify: &go-files
        - "**/*.go"
      stages:
        - name: lint
          script: golangci-lint run ./...
        - name: resolve
          type: cnb:resolve
          options: { key: golang-check }

    summary:
      stages:
        - name: await golang
          type: cnb:await
          ifModify: *go-files               # wait only when Go files changed
          options: { key: golang-check }
        - name: notify
          image: tencentcom/wecom-message
          settings: { msgType: markdown, content: "PR 检查通过" }
```

**Cross-repo shared templates**: when many business repos share the same build/deploy flow, encapsulate the generic flow in a template repo;
business repos only declare variables. Template/business separation, centralized secrets, `$CNB_PIPELINE_KEY` dynamically loading per-pipeline variable files —
one repo can run many pipelines:

```yaml
# Template repo build-deploy.yml — business differences injected via variables
.build-and-deploy:
  services:
    - docker
  imports:
    - https://cnb.cool/<org>/<secret-repo>/-/blob/main/secret.yml   # credentials, kubeconfig
    - .cnb/workload_env.$CNB_PIPELINE_KEY.yml                      # matched dynamically by pipeline key
  stages:
    - name: build image
      script: |
        docker build -t ${IMAGE}:${CNB_COMMIT_SHORT} .
        docker push ${IMAGE}:${CNB_COMMIT_SHORT}
    - name: deploy
      image: bitnami/kubectl
      script: kubectl set image ${WORKLOAD_TYPE}/${NAME} ${CONTAINER}=${IMAGE}:${CNB_COMMIT_SHORT} -n ${NS}

# Business repo .cnb.yml — cross-file must use !reference, not anchors
include:
  - https://cnb.cool/<org>/template/build-deploy/-/blob/main/build-deploy.yml
main:
  push:
    service-a: !reference [.build-and-deploy]   # CNB_PIPELINE_KEY = "service-a"
```

---

## Common config patterns

### Multi-pipeline orchestration with `cnb:await` / `cnb:resolve`

```yaml
main:
  push:
    build-frontend:
      stages:
        - name: build
          script: npm run build
        - name: resolve
          type: cnb:resolve
          options: { key: frontend-ready }
    deploy:
      stages:
        - name: await frontend
          type: cnb:await
          options: { key: frontend-ready }
        - name: deploy
          script: ./deploy.sh
```

### Stage-level `imports` for minimal secret exposure

Pipeline-level `imports` applies to all stages, but not every step needs every secret.
Push the secret reference down to the Stage that needs it (least privilege):

```yaml
- name: build-and-push
  stages:
    - name: build
      script: npm run build
    - name: docker login
      imports:
        - https://cnb.cool/<org>/<secret-repo>/-/blob/main/registry-credentials.yml
      script: docker login -u "$REGISTRY_USER" -p "$REGISTRY_TOKEN" mirrors.example.com
    - name: docker push
      script: docker push mirrors.example.com/my-app:$CNB_COMMIT_SHORT
```

### Deploy anti-pileup: `lock` + `cancel-in-wait`

```yaml
main:
  push:
    - name: deploy-dev
      lock:
        key: deploy-dev
        wait: true              # queue instead of failing (without it, concurrent runs fail)
        cancel-in-wait: true    # cancel queued same-key pipelines, keep only the newest
      stages:
        - name: deploy
          script: ./deploy.sh
```

> `cancel-in-progress` cancels running ones; `cancel-in-wait` cancels queued ones. Deploy scenarios usually set both.

### Pre-install system dependencies via Dockerfile

Avoid `apt install`/`yum install` in scripts — it repeats on every pipeline run.
Installing system packages in scripts reinstalls them on every run. Point `docker.build` at a Dockerfile:
the image is auto-cached and rebuilt only when the Dockerfile or dependency files change (see the official build-env doc).

```yaml
# ✅ preferred: pre-install via image/Dockerfile
main:
  push:
    - docker:
        build: image/Dockerfile
      stages:
        - name: build
          script: python3 build.py
```

### Give Pipelines / Stages meaningful names

Names show up in the build UI. Use "verb + noun": `install-deps`, `run-unit-tests`, `deploy-to-staging`.
Avoid uninformative names like `step1` or `build2`.

### Multi-arch images (amd64 + arm64)

Use `runner.tags` to build on different architecture nodes concurrently, orchestrate with `cnb:resolve`/`cnb:await`,
then merge with a manifest plugin:

```yaml
.build-by-arch: &build-by-arch
  - name: docker build & push
    script:
      - docker build -t $IMAGE:$VERSION-linux-$BUILD_ARCH .
      - docker push $IMAGE:$VERSION-linux-$BUILD_ARCH
  - name: resolve
    type: cnb:resolve
    options: { key: build-$BUILD_ARCH }

main:
  push:
    - name: build-amd64
      runner: { tags: [cnb:arch:amd64] }
      services: [docker]
      env: { BUILD_ARCH: amd64 }
      stages: [*build-by-arch]
    - name: build-arm64
      runner: { tags: [cnb:arch:arm64:v8] }
      services: [docker]
      env: { BUILD_ARCH: arm64 }
      stages: [*build-by-arch]
    - name: combine-arch
      services: [docker]
      stages:
        - name: await amd64
          type: cnb:await
          options: { key: build-amd64 }
        - name: await arm64
          type: cnb:await
          options: { key: build-arm64 }
        - name: manifest
          image: cnbcool/manifest
          settings:
            target: $IMAGE:$VERSION
            template: $IMAGE:$VERSION-OS-ARCH
            platforms: [linux/amd64, linux/arm64]
```

### End-to-end example: Node.js CI/CD

Covers push test-and-build, PR checks, and tag release:

```yaml
.node-env: &node-env
  docker:
    image: node:20
    volumes: [node_modules]

main:
  push:
    - name: test-and-build
      <<: *node-env
      ifModify: [src/**, package.json]
      stages:
        - name: install
          script: npm ci
        - name: test
          script: npm test
      endStages:
        - name: notify
          script: echo "Build $CNB_BUILD_ID completed"

"**":
  pull_request:
    - name: pr-check
      <<: *node-env
      stages:
        - name: test-with-coverage
          script: npm test -- --coverage
        - name: coverage-report         # incremental coverage only under PR events
          type: testing:coverage
          options: { pattern: coverage/lcov.info, lines: 80, diffLines: 90 }

"$":
  tag_push:
    - name: release
      docker: { build: ./Dockerfile }
      services: [docker]
      stages:
        - name: build-image
          script: |
            docker build -t $CNB_DOCKER_REGISTRY/$CNB_REPO_SLUG:$CNB_BRANCH .
            docker push $CNB_DOCKER_REGISTRY/$CNB_REPO_SLUG:$CNB_BRANCH
        - name: create-release
          type: git:release
          options: { descriptionFromFile: CHANGELOG.md }
```
