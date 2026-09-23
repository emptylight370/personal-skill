# CODEBUDDY.md This file provides guidance to CodeBuddy when working with code in this repository.

## 仓库性质

这是一个**纯 Markdown 的个人 Skill 集合仓库**，没有任何代码构建、测试或 lint 流程。所有"开发"工作都是编写和维护 skill 文档。提交信息使用中文，遵循仓库现有风格。

## 文件操作工具优先级

获取目录下文件列表、删除文件等文件操作，**优先使用 CodeBuddy 内置工具**（`list_dir`、`search_file`、`search_content`、`read_file`、`delete_file` 等），而不是通过终端执行 `ls`、`Get-ChildItem`、`Remove-Item` 之类的 shell 命令。仅当内置工具无法满足需求时（如需要批量重命名、权限修改等），才回退到终端命令。

## GitHub 访问方式

访问 GitHub（查询 repo、release、issue、PR 等）时，**优先使用本地安装的 `gh` 命令行工具**，默认断言其处于登录状态，无需事先检查认证。如果某次 `gh` 调用失败，先检查命令本身（参数、网络等）并重试；重试两次后仍失败，才回退到直接调用 GitHub API（如 `Invoke-RestMethod https://api.github.com/...`）。

## 常用命令

### 校验 skill 结构与 frontmatter

新增或修改 skill 后，用 skill-creator 的校验脚本检查（路径来自 CodeBuddy 扩展安装目录，若版本号不同请按实际路径调整）：

```bash
python "c:\Users\13789\.vscode\extensions\tencent-cloud.coding-copilot-*\out\extension\builtin\skill-creator\scripts\quick_validate.py" <skill 目录>
```

输出 "Skill is valid!" 即通过。注意：description 必须写成**单行**，不能用 YAML 折叠标量 `>`，否则脚本会把 `>` 误判为尖括号而报错。

### 初始化新 skill 骨架

```bash
python "<skill-creator 目录>\scripts\init_skill.py" <skill-name> --path <本仓库根目录>
```

生成模板后删除用不到的 `scripts/`、`assets/` 示例目录。

### 获取/更新单个 skill（稀疏检出）

见 `README.md` 中的 sparse-checkout 说明——本仓库的主要消费方式是把单个 skill 文件夹复制到目标项目的 `.agents/skills/`（Claude Code 为 `.claude/skills/`）。

## 架构与结构

### 顶层布局

```
personal-skill/
├── <skill-name>/        # 每个 skill 一个目录，目录名 = skill 名
│   ├── SKILL.md         # 必需：frontmatter（name + description）+ 正文
│   └── references/      # 可选：按需加载的参考材料
├── docs/                # 设计文档集中地（中文，文件名用 skill 名）
├── README.md            # Skill 列表索引，新增 skill 必须同步更新
├── AGENTS.md            # 本文件
└── *.md（如 chat-history.md）  # 单个 skill 设计过程中的临时文件，可能随时移除，不作为任何依据
```

### Skill 的组织约定

- **SKILL.md 是唯一入口**：frontmatter 只含 `name`（小写连字符，与目录名一致）和 `description`（英文为主、尾部附中文触发词，单行）。正文用英文祈使式指令书写；将 skill 分发到非中文项目时无需改动。
- **references/ 存放按需加载的材料**（如模板），不放在 SKILL.md 正文中，控制主体篇幅；不需要 `scripts/`、`assets/` 时直接删除示例目录。
- **skill 与设计文档分离**：skill 目录只包含会被分发的内容；设计文档统一放顶层 `docs/<skill-name>.md`，不随 skill 复制，以 `docs/` 和 skill 本体为准。仓库根目录下可能出现 `chat-history.md` 之类的**临时文件**——它们是单个 skill 设计过程中的对话沉淀，不属于仓库的正式内容，可能随时被移除，不要将其作为依据或主动维护。

### 现有 skill

- **siyuan-skill**：通过 `siyuan` CLI 操作思源笔记，命令参考在 `references/commands.md`。
- **issue-triage**：GitHub issue 分诊流程。其特有机制：项目上下文卡 `project-context.md` 运行时由 agent 检索生成后写入 SKILL.md 同级目录，**不随 skill 复制**；进度跟踪按 issue 一个文件落盘。修改此 skill 时注意保持"审阅前置、不预设操作分支"的核心设计不变。

### 新增/修改 skill 的完整流程

1. （可选前置）**生成临时设计文档**：当需求较复杂或尚未明确时，先在 `docs/` 下创建同名设计文档草稿（中文），梳理设计原则、流程与落地形态，确认后再进入下一步；
2. `init_skill.py` 生成骨架 → 编写 SKILL.md 与 references；
3. `quick_validate.py` 校验通过；
4. **同步更新 `README.md` 的 Skill 列表**（这是本仓库最容易遗漏的步骤）；
5. 在 `docs/` 创建同名设计文档（中文），说明设计原则、流程与落地形态。
