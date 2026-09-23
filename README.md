# Personal Skill

个人 Skill 仓库，用于存放我在日常工作中沉淀下来的各类 Skill。每个 Skill 独立存放在一个单独的文件夹中，文件夹名称即为该 Skill 的名称，内部通常包含 `SKILL.md`（Skill 定义与说明）以及 `references/`、`scripts/` 等辅助目录。

各 Skill 的设计文档统一存放在 [docs/](./docs) 目录（文件名用对应 skill 名），不随 skill 本身分发。

## 获取 Skill

### 稀疏检出（sparse-checkout）

无需克隆整个仓库，可通过 Git 的 `sparse-checkout` 只拉取需要的 Skill 文件夹。以获取 `siyuan-skill` 为例：

```bash
# 1. 克隆仓库到本地（--sparse 启用 cone 模式稀疏检出；--depth 1 只取最新提交）
git clone --filter=blob:none --sparse --depth 1 git@github.com:emptylight370/personal-skill.git
cd personal-skill

# 2. 指定需要检出的 Skill 文件夹（可一次指定多个，文件内容在此步按需下载）
git sparse-checkout set siyuan-skill

# 3. 如需追加其他 Skill：set 为覆盖式，需连同已有文件夹一并列出
git sparse-checkout set siyuan-skill another-skill
# 也可用 add 追加（Git 2.37+），不影响已检出的文件夹
git sparse-checkout add another-skill
```

更新时进入仓库目录拉取远端最新内容即可，`sparse-checkout` 的配置会被保留：

```bash
cd personal-skill
git pull
```

若后续需要调整已检出的 Skill 文件夹，重新执行 `git sparse-checkout set <文件夹...>` 即可（该命令为覆盖式，需列出全部想保留的文件夹）；若只想追加而不影响已检出的文件夹，可用 `git sparse-checkout add <文件夹>`（Git 2.37+）。

### 复制到项目 skill 目录

如果项目已经自带 skill 目录（多半位于 `<project>/.agents/skills/`，Claude Code 中位于 `<project>/.claude/skills/`），希望在其中直接放一个普通 Skill 文件夹（不含 `.git`、不嵌套仓库），可先克隆部分或整个仓库，再把对应的 Skill 文件夹复制过去。

```bash
# 1. 克隆仓库（部分：仅稀疏检出所需 Skill；也可去掉 --sparse 克隆整个仓库）
git clone --filter=blob:none --sparse --depth 1 git@github.com:emptylight370/personal-skill.git /tmp/personal-skill

# 2. 指定需要的 Skill（克隆整个仓库时可跳过）
git -C /tmp/personal-skill sparse-checkout set siyuan-skill

# 3. 复制到项目的 skill 目录（Claude Code 为 <project>/.claude/skills）
mkdir -p <project>/.agents/skills
cp -r /tmp/personal-skill/siyuan-skill <project>/.agents/skills/
```

复制得到的是纯内容文件夹（不含 `.git`），可直接被项目使用。Windows 下可用 `$env:TEMP\personal-skill` 作为克隆目录，并将 `cp -r` 替换为 `Copy-Item -Recurse -Force`。

后续更新时重复上述操作即可：先 `git -C /tmp/personal-skill pull` 更新临时仓库，再覆盖复制到项目的 skill 目录。

## Skill 列表

### [issue-triage](./issue-triage)

对 GitHub 仓库的 issue 进行分诊与处置：拉取列表、生成摘要与初步分类建议、呈现决策卡片，由用户开放性决定处置方式（自行修改或指导 agent 修改）后执行（改码 / 回复 / 打标 / 关闭），并支持跨会话进度跟踪。所有对外可见的写操作（评论、标签、里程碑、关闭、转移、PR）一律先经用户审阅；项目级配置（标签列表、提交策略、收尾策略等）通过 agent 检索生成、用户审阅后的上下文卡（`project-context.md`）注入。

### [requirement-clarify](./requirement-clarify)

在动手实现之前先把用户需求确认清楚：agent 从用户表述中推导出其真实期望的需求并复述供确认，同时列出需求中不明确、需要用户确认的问题（优先通过内置提问工具发起），经若干轮对话直至用户明确确认；问题按"已解答 / 显式搁置 / agent 自定默认值待确认"三态跟踪。确认后开始落实前统一追问仍未解答的问题，并依据用户要求或记忆中的用户偏好确定落实方式（如分模块渐进式或一次性完成）；实施中发现证据与已确认需求矛盾时带证据上报再确认；用户后续追加需求时，与现有逻辑一致的部分直接融入并做回归验证，不明确或有冲突的部分再次确认。

### [siyuan-skill](./siyuan-skill)

通过命令行工具（`siyuan`）操作思源笔记，支持管理笔记本、文档、块、SQL 查询、搜索、导入导出、资源、属性、书签、标签、历史、引用、仓库快照、同步等。
