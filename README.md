# analyzer

一个用来**分析 agent 代码库**的工具。给它一个 agent 仓库路径，它从入口开始沿数据流把程序逻辑捋成一棵树，产出结构化底座 + 可读文档。

> 分析 agent ≠ 读函数。agent 的控制流一半在代码、一半在 prompt，analyzer 把两者并重地读，结论都挂证据指针，可溯源回源码、不编造。

## 怎么用

安装：把本仓库的 `.claude/` 复制到 `~/.claude/`（全局可用）或你的项目 `.claude/`（局部可用）。装好后得到 **2 个 skill + 4 个 subagent**。

### 1. 分析一个 agent 仓库

```
/analyzer /path/to/some-agent-repo
```

主 agent 会：全仓扫描找入口 → 从入口沿数据流一节点一节点推进（每节点读+写+事实核查）→ 终审 → 把底座改写成一份可读总览。跑完，该仓库的 `.analyzer/` 下就有一整套文档。

### 2. 读已分析的文档 / 向 agent 提问

仓库分析过（`.analyzer/` 已存在）后，想问问题（"它怎么实现 X""X 在哪""X 变量什么意思"）：

**先 `/analyzer-docs` 加载这个 skill，再用自然语言向 agent 提问。**

它会**先文档后源码**：从 OVERVIEW → MAP → detail 逐层定位、用文档回答；只在确认某个具体细节时，才读文档指针指的那一段源码——不会一头扎进源码起手。回答会标注证据来自哪层（OVERVIEW / detail 某段 / 源码 path:line）。

## 产出

在被分析仓库的 `.analyzer/` 下：

| 文件 | 是什么 |
|---|---|
| `OVERVIEW.md` | 给人看的可读总览：大白话串数据流 + 关键机制 + 一个端到端例子。**人先读这个**。 |
| `map.json` | 整棵树，机器可读底座。每节点挂输入 / 处理（代码+prompt）/ 输出 + 证据指针。 |
| `MAP.md` | 树的渲染：每节点一句话意义 + 指针（索引）。 |
| `detail/*.md` | 分层节点的机制细节（变量/函数带人话 gloss）。 |
| `inventory.md` | 代码概览 + 入口定位（也是上下文压缩后的恢复点）。 |

每条结论挂证据指针（代码 `code_ref: path:line`、LLM 逻辑 `prompt_ref`），可溯源回源码，没找到填 `null` + `_notes`，绝不编造。

## 组成

- **skills**：`analyzer`（编排分析、产出文档）、`analyzer-docs`（读文档答问题、先文档后源码）。
- **subagents**：`analyzer-scout`（一次性全仓粗读找入口）、`analyzer-reader`（读+写树节点）、`analyzer-reviewer`（事实核查）、`analyzer-narrator`（把底座改写成可读 OVERVIEW）。

职责严格分离：遍历决策归主 agent，找入口归 scout，读写归 reader，核查归 reviewer，可读化归 narrator。reviewer 只查事实错误、不挑遗漏（否则 review 循环永不收敛），最多 2 轮。
