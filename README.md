# analyzer

一个用来**分析 agent 代码库**的 agent。给它一个 agent 仓库路径，它从入口开始沿数据流把程序逻辑捋成一棵树，产出结构化底座 + 可读地图文档。

> 分析 agent ≠ 读函数。agent 的控制流一半在代码（确定性脚手架），一半在 prompt / tool description（LLM 运行时才决定走哪条路）。analyzer 把两者并重地读，结论必须挂证据指针。

## 怎么用

在 Claude Code 里（需将本仓库的 `.claude/` 放到你的项目，或复制到 `~/.claude/` 全局可用）：

```
/analyzer 分析 /path/to/some-agent-repo
```

或者clone该项目后，让你的coding agent安装此项目的1个skill和3个subagent。

主 agent 会：
1. **阶段一**：分批派 scout 全仓扫描，产出 `inventory.md` + 定位入口，scout 退场。
2. **阶段二**：从入口节点开始，沿数据流一个节点一个节点推进——每节点派 reader 读+写、派 reviewer 事实核查，通过后进下一节点，长成一棵树。
3. **终审**：`json.load` 验证、树连通性检查、MAP.md 长度控制，给一句话总结。

## 核心理念

**从入口追数据流，长成一棵树**——不套预设的"分层"框架。

每个树节点 = 一个模块 / 一个处理步骤：

```
节点
├─ 输入：什么数据进来，从哪个上游
├─ 处理：
│  ├─ 程序逻辑（确定性，code_ref）
│  └─ prompt 逻辑（LLM 运行时决定的控制流，prompt_ref 原文；无则空）
├─ 输出：产出什么 → 传给哪个下游节点
└─ attached：横切项（guardrails / model_config / evals / 死代码 等）挂这里
```

- 一个模块只出现一次，代码逻辑和 prompt 逻辑并列挂在自身（不会像分层框架那样把 prompt 拆到"入口"和"资产"两处、把循环拆到"主循环"和"数据流"两处）。
- 横切项挂到触发它的节点：`maxiter` 挂主循环节点、`cost/model_config/retry` 挂 LLM 调用节点、`evals` 挂评测入口、死代码挂对应节点。重复出现写"同 <节点id>"。
- **"模型何时调某工具/走哪条路"这条逻辑只在 prompt 里**——这是分析时最容易漏的点，结论必须去 prompt 找原文证据，不能靠猜调用图。

## 四层流水线

职责严格分离，避免越界：

| 角色 | 职责 | 写什么 |
|---|---|---|
| **主 agent**（SKILL） | 编排：统计仓库→派 scout→从入口沿数据流决定下一个节点→收敛判定→终审 | 只写 `_notes` 残留项 |
| **scout** | 一次性初始步骤：分批全仓粗读，产代码概览 + 找入口 | `.analyzer/inventory.md` |
| **reader** | 只做 read+写+报告客观发现。读指定文件，写一个树节点 | `map.json`（唯一写入方）+ `MAP.md` / `detail/*.md` |
| **reviewer** | 只读事实核查：指针造假/摘录失真/编造/越界/JSON 破坏/树断链，**不挑遗漏** | 无（只读） |

**关键约束**：
- reader 不决定下一个节点、不开药方——遍历决策归主 agent。找入口归 scout，聚焦决策归主 agent，reader 只管 read。
- reviewer 禁止挑遗漏——遗漏永远挑得出来，允许挑遗漏会让 review-fix 循环永不收敛。最多 2 轮，仍错就记 `_notes` 推进。
- reviewer 用 `json.load` 实检 map.json 合法性，不只肉眼看（堵住漏逗号）。
- scout 单批 ≤10 个文件；依赖/日志/产物大目录读 1-2 个样本就整目录标跳过，不逐个读。

## 文件结构

```
.claude/
├── skills/analyzer/SKILL.md          # 主 agent 编排方法论
└── agents/
    ├── analyzer-scout.md             # 一次性粗读执行者
    ├── analyzer-reader.md            # 读取与记录执行者（持有 map.json schema）
    └── analyzer-reviewer.md          # 事实核查执行者
```

## 产出

在被分析仓库的 `.analyzer/` 目录下：

```
.analyzer/
├── map.json          # 整棵树（完整底座，机器可读，不拆）
├── MAP.md            # 树的渲染（≤300 行 / ≤10000 字符，超了则分层）
├── inventory.md      # scout 代码概览 + 入口定位（也是上下文压缩后的恢复点）
└── detail/*.md       # 分层时的细节文件（按源文件同名）
```

- `map.json` 是 `nodes` 数组，每节点有 `id / parent / inputs / process.code / process.prompt / outputs / attached`。
- 每条结论挂证据指针：代码结论 `code_ref: "path:line"`，LLM 决定的逻辑 `prompt_ref: "摘自…原文…"`。没找到填 `null` + `_notes`，绝不编造。

## 设计动机

读一个复杂 agent 仓库时，传统"分层"分析（边界/入口/主循环/数据流/控制边界/资产）会重合——主循环既是路由又是循环、turn_steps 本就是数据流、prompt 散落在各层却被归到"资产"。数据流树消除这些重合：模块只出现一次，程序逻辑和 prompt 逻辑并列。

而分析过程本身也是个 agent 任务，所以用 scout/reader/reviewer 三层分离 + 事实核查专职化来控制质量——reader 视野窄不该决定方向、遗漏挑不完所以 reviewer 被禁止挑遗漏、JSON 语法错误靠实检不靠肉眼。
