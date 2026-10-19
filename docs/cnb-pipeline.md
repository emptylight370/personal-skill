# cnb-pipeline 设计文档

> 目标：从官方 `cnb/skills` 仓库的 `cnb-pipeline` skill 中提取「编写 `.cnb.yml` 流水线配置」部分，剥离所有平台操作耦合（诊断、CLI、凭证、NPC 运行时约束），以**纯联网查官方文档**为语法事实来源，落地为本仓库一个**通用本地编写型 skill**（不在 CNB 平台运行、不面向 WorkBuddy，随本仓库按 `.agents/skills/` 方式分发）。
>
> 状态：**已落地**。最终形态见 `cnb-pipeline/` 目录。

## 1. 来源与背景

官方仓库 `cnb.cool/cnb/skills` 的 `skills/cnb-pipeline` 是「配置」部分的本体（description：`编写、修改、审查 .cnb.yml 流水线配置，诊断构建失败原因并优化性能`），由 `SKILL.md` + 8 个 `references/` 分册构成。

其**关键设计——官方刻意「去知识化」**：

> 「平台知识一律现查官方文档，不要背。字段名、内置任务参数、事件适用性、变量清单……都由 `cnb-docs` skill 负责查，本 skill 不另抄一份文档目录。」「本文件只写文档里没有的易错点，不维护字段手册。」

即官方 skill 只自带：结构骨架（分支 → 事件 → Pipeline → Stage → Job）、三链语义（stages / failStages / endStages）、易错点。字段全表、事件全表、内置变量、内置任务参数、节点规格、插件用法全部依赖兄弟 skill `cnb-docs` 在线查（其路由机制：取 `docs.cnb.cool/zh/llms.txt` 索引定位文档页）。官方 skill 也不带校验脚本。

因此提取的核心工作是：**用一个 `doc-index.md` 路由表替代 `cnb-docs` 的查证角色** + **剥离平台操作**。

## 2. 设计原则

- **纯联网、零维护**：不内联字段/事件/变量快照，所有取值类事实在运行时按 `references/doc-index.md` 的路由表 WebFetch `docs.cnb.cool` 现查。路由表只存「要查什么事实 → 哪一篇」，不抄文档内容；索引页 `/zh/llms.txt` 作为路径失配时的兜底。
- **骨架与易错点离线自带**：官方分册中不依赖文档的部分（结构骨架、三链语义、易错点、变量替换规则、定时同步规则）原样保留，离线也能写简单流水线。
- **只写不诊断**：删除「诊断工作流程」与 `diagnose-guide.md`（依赖 `cnb-api` + `cnb` CLI + `CNB_TOKEN`），删除所有 `cnb pulls`/`cnb build` 命令引用，删除「修后不轮询」「NPC 自动唤起」等 NPC 运行时专属约束。skill 从「要凭证的操作型」退化为「纯本地知识型」。
- **遵循本仓库 skill 约定**：frontmatter 仅 `name` + `description`（description 单行，英文为主、尾部附中文触发词）；正文英文祈使式指令，分发到非中文项目无需改动；YAML 示例保留官方原样。
- **平台中立**：正文与分册不含 WorkBuddy/CodeBuddy/NPC 运行时专属措辞与安装说明；插件市场等 URL 写死为 SaaS 公网地址（原 `${CNB_WEB_HOST}` 环境变量写法是 NPC 运行时产物）。
- **`self-hosted-runner.md` 瘦身**：只留纯配置约束（平台/架构支持、双配必需、`docker.image` 在 Mac/Win 不生效、按平台写脚本），删掉节点接入、日志排查、`runner-config.json` 安全配置等平台操作内容。
- **`crontab.md` 只留配置相关**：保留心智模型（`.cnb.yml` 与定时任务列表两份数据靠同步对齐）、配置要点、同步规则速查、删除清理；删掉依赖 `cnb` CLI 的排查流程，同步操作保留一句 Web UI 入口指引。
- **`npc:go` 保留**：它是写在 `.cnb.yml` 里的内置任务，属「写配置」范畴；但删去「修后不轮询」等运行时约束。

## 3. 文件清单与角色

```
cnb-pipeline/
├── SKILL.md                  # 工作流（7 步）+ 模式判定 + 对照核对 + 自托管速览 + 约束速查
└── references/
    ├── doc-index.md          # 新增：查证路由表，替代 cnb-docs（URL 已逐条核实）
    ├── syntax-reference.md   # 结构骨架 + 三链语义 + 字段易错点
    ├── builtin-tasks.md      # 内置任务用法骨架 + 判断点 + npc:go 要点
    ├── env-variables.md      # 变量声明/导入/导出/替换机制 + 易错点
    ├── plugin-search.md      # 插件市场搜索 → 读 README → 生成 settings 的流程
    ├── best-practices.md     # 锚点/!reference/.ci 拆分 + Monorepo 编排 + 常见模式
    ├── crontab.md            # 定时任务配置与同步规则
    └── self-hosted-runner.md # 自托管 Runner 纯配置约束（瘦身版）
```

未搬入：`diagnose-guide.md`（平台操作型，整体不适用）。

`doc-index.md` 的路由表以 `docs.cnb.cool/zh/llms.txt` 索引为兜底，主要条目（均已在 llms.txt 中核实存在）：

| 事实 | 路径 |
|---|---|
| 语法手册 / 配置文件 / 触发规则 / 构建环境 | `/zh/build/grammar.md`、`configuration.md`、`trigger-rule.md`、`build-env.md` |
| 简化配置 / 文件引用 / 脚本与插件任务 | `/zh/build/simplify-configuration.md`、`file-reference.md`、`script-vs-plugin.md` |
| 构建节点 / 环境变量 / 默认环境变量 / 缓存 | `/zh/build/build-node.md`、`env.md`、`build-in-env.md`、`pipeline-cache.md` |
| 定时任务 / 手动触发 / 登录调试 / 部署 | `/zh/build/crontab.md`、`web-trigger.md`、`login-debug.md`、`deploy.md` |
| 内置任务索引 + 各任务子页 | `/zh/build/internal-steps.md` + `/zh/build/internal-steps/<vendor>/<task>.md` |
| 插件市场 / 制作 / 贡献 | `/zh/plugins.md`、`/zh/build/create-plugin.md`、`/zh/build/contribute-plugin.md` |

> 初稿中的 `.html` 猜测路径已全部按 llms.txt 实际收录路径替换为 `.md`（raw markdown），并写明 `.html` 为渲染页等价物。

## 4. 运行时工作流

1. 问需求（触发分支/事件、构建语言/环境、特殊需求）；
2. 读现状（修改场景先读 `.cnb.yml` 与 `.ci/`）；
3. 按 `doc-index.md` WebFetch 对应官方文档页，确认字段/事件/变量取值；
4. 插件任务按 `plugin-search.md` 搜市场 → 读插件 README → 生成 `image` + `settings`；
5. 按 `syntax-reference.md` 骨架生成 `.cnb.yml`；
6. 逐项对照官方文档核对（结构 / 事件 / 变量 / 插件 / 骨架五张表，每条要有文档出处）；
7. 汇报并解释要点；WebFetch 失败时降级为「骨架生成 + 显式标注未核对字段」。

## 5. 提取时所做的适配（与官方原文的差异）

- **SKILL.md**：新增「Source of truth」节替代 `cnb-docs` 依赖；模式判定表删去「诊断」行、加「查证路由」行；删「诊断工作流程」整节；自托管一节缩为两句纯配置要点；约束速查删「修后不轮询」红线。
- **全部分册**：所有「让 `cnb-docs` 现查」改为「按 `doc-index.md` 查官方文档」；`plugin-search.md` 中 `${CNB_WEB_PROTOCOL:-https}://${CNB_WEB_HOST:-cnb.cool}` 环境变量 URL 写死为 `https://cnb.cool/...`；删去面向 NPC 的「顺手完善项目 skill 并随 PR 提交」类运行时指令，改为「发现分册过时就改分册」。
- **plugin-search.md**：删「插件 README 编写建议」一节（属插件贡献方，非写配置所需）。
- **self-hosted-runner.md**：删「排查前先识别 Runner 类型」「常见问题排查」「安全建议」三节与接入指引。
- **正文语言**：英文祈使式（本仓库约定）；官方示例 YAML 原样保留。

## 6. 遗留事项与已知限制

1. **纯联网的代价**：每次生成要多个 WebFetch 步骤，必须联网；离线时只能靠各分册骨架 + 易错点写简单流水线，拿不准的字段值无法核对。
2. **无校验脚本**：官方即无，靠「对照文档核对」流程兜底；语法正确性最终以官方文档为准。
3. **官方文档路径可能漂移**：`doc-index.md` 已内置 `/zh/llms.txt` 兜底规则（路径失配时重查索引），但路由表本身仍需在文档大改版时人工过一遍。
4. **`runner.tags` 官方节点取值**：官方节点池的 tags 清单（如 `cnb:arch:amd64`）只在示例中出现，属「取值查文档」类，由运行时按 `build-node.md` 核对，未在本 skill 固化。
5. **未验证分发**：尚未实际复制到某个目标项目 `.agents/skills/` 验证触发效果；首次使用时建议做一次端到端验证（如「帮我的 Go 项目写个 .cnb.yml」）。

## 7. 落地与维护

- 落地位置：本仓库 `cnb-pipeline/`；`README.md` Skill 列表已同步新增条目。
- 维护方式：与官方 `cnb-docs` 去知识化设计一致——**不维护事实快照**，只维护路由表与易错点；平台演进导致分册表述过时时，随使用顺手修订分册。
- 官方仓库更新跟踪：官方 `cnb/skills` 的 `cnb-pipeline` 若新增分册或易错点，可浅克隆对照增量吸收（`git clone --depth 1 https://cnb.cool/cnb/skills.git`）。
