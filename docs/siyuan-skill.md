# siyuan-skill 设计文档

> 目标：skill 是思源笔记 CLI（`siyuan`）的**辅助文档**，内容从本地安装的 siyuan 命令行帮助输出中读取并生成；skill 的更新**由用户手动触发**，agent 负责读取本地 siyuan 命令的输出，与 skill 现有内容比对后修订需要更新的部分。

## 1. 设计原则

- **以本地 CLI 输出为唯一事实来源**：skill 中记录的所有命令、flag、默认值、描述，均以本地实际安装的 `siyuan` 版本的 `--help` 输出为准，不凭记忆或网络资料臆写。
- **手动触发更新**：skill 不会自动检测 CLI 升级；只有用户明确要求"更新 siyuan-skill / 检查 siyuan 命令是否有变化"时，agent 才执行更新流程。
- **只修订有差异的部分**：更新时逐命令比对，仅改写与新版本输出不一致的条目（版本号、flag 增删、描述变更、默认值调整），未变化的命令保持原文不动，避免无谓 churn。
- **文档忠实于输出**：`references/commands.md` 声明为"Auto-generated from `siyuan --help` output"，因此正文应忠实转写 CLI 实际文案；即使上游文案疑似笔误，也如实记录并可附注说明，不擅自"修正"。

## 2. skill 的内容构成

| 文件 | 来源 | 作用 |
|------|------|------|
| `SKILL.md` | 从 CLI 输出提炼 | 工作流（先发现工作区、再带 `-w` 执行）、领域概念（块类型、标题层级、hPath 等）、全局 flag、子命令索引表 |
| `references/commands.md` | 直接转写 CLI 输出 | 全部命令组的完整 flag 与描述参考，按需加载 |

SKILL.md 与 references 分工：SKILL.md 控制"怎么用"（工作流与易错点），references 承载"有什么"（完整参数表），后者篇幅大、按需加载。

## 3. 用户手动触发的更新流程

用户触发更新后，agent 按以下步骤执行：

1. **确认本地版本**：`siyuan --version`，与 skill 中记录的版本号对比；相同则询问用户是否仍需全量核对（默认跳过，只报"无变化"）。
2. **导出全部帮助输出**：`siyuan --help` + 对全部子命令组（`workspace`、`notebook`、`document`、`block`、`sql`、`search`、`export`、`import`、`asset`、`attr`、`bookmark`、`tag`、`history`、`ref`、`repo`、`sync`、`database`、`dailynote`、`file`、`inbox`、`outline`、`template`、`system`、`serve`、`completion`）逐一执行 `<cmd> --help`，并对其下的叶子子命令同样导出；结果先写入临时文件再读取，避免直接输出过长。导出文件用完即删，不留在仓库。
3. **逐项比对**，识别需要修订的差异：
   - 版本号标注（SKILL.md 的 CLI Version 行、commands.md 头部说明）；
   - 子命令的新增 / 移除 / 更名；
   - flag 的新增 / 移除 / 更名，以及描述文案、默认值的变化；
   - 类型枚举等约定值清单的变化（如 database key 类型）。
4. **修订 skill**：
   - commands.md：按差异更新对应命令组的表格与说明；
   - SKILL.md：若差异涉及工作流或领域概念（如新 flag 改变某个操作的正确用法），同步更新相应小节；纯参数表变化不动 SKILL.md 正文；
   - SKILL.md 的子命令索引表与实际命令集保持一致。
5. **汇报变更清单**：按"实质性变更 / 文案类变更 / 无变化"分组汇报本轮修订内容，供用户复核。
6. **校验**：改动完成后运行 `quick_validate.py` 确认 frontmatter 仍合法（description 保持单行）。

## 4. 更新边界

- **不主动升级 CLI 本身**：agent 只读取本地已安装版本的输出，不负责安装或升级 siyuan。
- **不联网拉取文档**：以本地 `--help` 输出为准；如需了解某个 release 的变更背景，可在汇报中附注，但不据网络资料改写参数表。
- **临时导出文件不入库**：帮助输出转存文件仅作比对中间产物，更新完成后删除。
- **版本号先行**：即使本轮只有版本号变化（命令集无差异），也要更新两处版本标注并在汇报中说明，使 skill 始终声明其依据的 CLI 版本。

## 5. 落地形态

```
personal-skill/
├── siyuan-skill/
│   ├── SKILL.md                    # 工作流 + 领域概念 + 全局 flag + 子命令索引（英文祈使式）
│   └── references/
│       └── commands.md             # 完整命令参考（自 CLI --help 输出转写，按需加载）
└── docs/
    └── siyuan-skill.md             # 本文档（中文，不随 skill 复制分发）
```

- **SKILL.md** 与 **references/commands.md** 一起随 skill 分发；分发到目标项目后，若目标项目安装的 siyuan 版本不同，由用户自行处理。
- **设计文档**（本文档）只存于本仓库 `docs/`，不进入 skill 目录。
