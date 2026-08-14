---
name: analyzer
description: 编排分析一个 agent 代码库，从入口沿数据流产出可溯源的 .analyzer/map.json、MAP.md、detail 与 OVERVIEW.md。需要系统理解某个 agent 项目如何工作、追踪代码逻辑与 prompt 逻辑、或更新既有 analyzer 产物时使用。
---

# analyzer — Codex 编排器

把用户输入当数据包，从程序入口开始沿真实调用链建立数据流树。每个节点记录输入、程序逻辑、prompt 逻辑、输出和证据指针。不要套固定“六层”框架。

## Codex 子 agent 启动规则

Claude Code 的 `.claude/agents/*.md` 不会被 Codex 自动注册为 `subagent_type`。在 Codex 中使用通用 `spawn_agent`，并把本 skill 的角色文件作为角色说明。

设 `<skill_dir>` 为本 `SKILL.md` 所在目录。每次启动角色前，主 agent 必须完整读取对应文件；子 agent 的首条消息也必须要求它先读取同一文件。

| 角色 | 角色文件 | `reasoning_effort` |
|---|---|---|
| scout | `references/roles/analyzer-scout.md` | `medium` |
| reader | `references/roles/analyzer-reader.md` | `high` |
| reviewer | `references/roles/analyzer-reviewer.md` | `high` |
| narrator | `references/roles/analyzer-narrator.md` | `high` |

启动形态：

```text
spawn_agent({
  task_name: "analyzer_<role>_<scope>",
  fork_turns: "none",
  reasoning_effort: "<上表值>",
  message: "先完整读取 <absolute-skill-dir>/references/roles/<role-file>，然后执行：<目标仓库、范围、输入、写入边界、验收标准>"
})
```

- 必须使用 `fork_turns:"none"`，并在 `message` 中显式传全任务上下文。完整历史 fork 会继承父 reasoning effort，不能单独覆盖，也会污染 reviewer 的独立性。
- 默认省略 `model`，让子 agent 继承父模型。只有用户明确指定模型时才覆盖。
- 若当前模型不支持上表 effort，使用该模型支持的最高不超过目标的档位；不要退到 `none`。
- Codex 没有 Claude agent frontmatter 的 `tools:` 强制白名单。角色文件中的只读/写入边界是行为约束；主 agent必须在每轮后检查实际改动范围。
- 不让这些角色再派子 agent。编排和下一节点决策只属于主 agent。

## 数据流树

```text
节点 = 一个模块或处理步骤
├─ 输入：什么数据从哪里进入
├─ 处理
│  ├─ 程序逻辑：确定性行为 + code_ref
│  └─ prompt 逻辑：模型决策规则 + prompt_ref 原文
├─ 输出：产出什么，流向哪个节点
└─ attached：在本节点触发的权限、模型、重试、预算、评测等横切项
```

`map.json` 是完整机器底座；`MAP.md` 是紧凑树索引；复杂机制写入 `detail/*.md`；`OVERVIEW.md` 是最终人话版本。

## 阶段一：全仓 scout

1. 用 `rg --files` 统计仓库文件与目录分布，确定批次。不要预先假定某目录无用。
2. 每批最多 10 个需要粗读的文件。疑似依赖、日志、构建产物的大目录单独成批，让 scout 读 1–2 个样本后整目录判定。
3. 读取 `references/roles/analyzer-scout.md`，按上表启动 scout。消息中传目标仓库、精确批次和唯一允许写入的 `<target>/.analyzer/inventory.md`。
4. 等待结果；若有未读清单，继续下一批。最终从 inventory 确认 CLI/server/headless 等真实入口。

scout 完成后退场，不用于后续节点分析。

## 阶段二：逐节点 reader → reviewer

主 agent 读取 `inventory.md` 和已有 `map.json`，根据上个节点的 `outputs.to` 决定下一个节点与文件范围。每个节点最多两轮闭环：

1. 读取 `references/roles/analyzer-reader.md`，启动 reader。消息必须给出：目标仓库、节点 id/名称/parent、要回答的数据流问题、文件清单、写入 `MAP.md` 还是指定 `detail/*.md`。
2. reader 完成后，主 agent检查它只改了约定的 `.analyzer` 文件。
3. 读取 `references/roles/analyzer-reviewer.md`，启动一个新的 reviewer。消息只给本轮节点、改动文件和事实基准；不要透露预期结论或暗示可能的错误。
4. 若 reviewer 报错，把错误逐条交回原 reader 修复，再启动新的 reviewer 复核。
5. 第二轮仍有错时，把残留事实问题记入 `map.json._notes` 后继续，避免死循环。

reviewer 只查事实错误，不挑遗漏或文风。涉及 `map.json` 时必须实际解析 JSON；只核查 Markdown 时跳过 JSON 检查。

## 阶段三：终审与 OVERVIEW

数据流追到出口且分支收敛后，主 agent执行：

1. 验证 `map.json` 可解析，root/parent/outputs.to 连通，`MAP.md` 不超过 300 行且不超过 10000 字符。
2. 读取 `references/roles/analyzer-narrator.md`，启动 narrator。要求它只写 `<target>/.analyzer/OVERVIEW.md`，以底座为事实源。
3. narrator 完成后启动新的 reviewer，只核查 OVERVIEW 的编造、术语失真和例子矛盾，不把可读版改回指针墙。
4. 最多两轮 narrator/reviewer。通过后结束。

## 写入与安全边界

- 保留用户已有改动；开始前和每个子 agent 完成后检查工作树。
- reader 是 `map.json`、`MAP.md`、`detail/*.md` 的唯一写入角色；narrator 只写 `OVERVIEW.md`；reviewer 只读。
- 使用 `rg`/`rg --files` 搜索，使用 `apply_patch` 修改文本；不要用 shell 重定向覆盖用户文件。
- 不进入 `node_modules`、vendor、构建产物或日志做深度分析。
- 证据不足时写 `null`/存疑，不得把推测写成事实。

## 产物

```text
<target>/.analyzer/
├── inventory.md
├── map.json
├── MAP.md
├── detail/*.md
└── OVERVIEW.md
```
