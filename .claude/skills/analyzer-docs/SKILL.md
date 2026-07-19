---
name: analyzer-docs
description: 当用户对一个**已经被 /analyzer 分析过**的 agent 代码库提问（"它怎么实现 X""X 在哪""为什么这么设计""X 变量什么意思"）时用本 skill。不直接 grep/读源码起手——先读 `<target>/.analyzer/`：OVERVIEW.md（人话总览）→ MAP.md/map.json（树结构+指针）→ detail/*.md（机制细节）逐层定位，用文档回答；需确认某具体行为细节时，再用文档给的 `code_ref`/`@行号` 去源码对应那一小段读一眼确认。产出给人看的回答 + 标注证据来自哪层（OVERVIEW / detail 节点X / 源码 path:line）。需要理解一个**已分析**的 agent 怎么工作时使用；若该仓库还没分析过，先用 /analyzer。
---

# analyzer-docs — analyzer 文档阅读向导

你加载本 skill 后，是 analyzer 产出文档的**阅读向导**。前置：目标仓库的 `<target>/.analyzer/` 已存在（由 `/analyzer` 产出：OVERVIEW.md + map.json + MAP.md + inventory.md + detail/*.md）。**若不存在 → 告诉用户先跑 `/analyzer` 分析该仓库，本 skill 没底可读。**

## 核心原则：先文档后源码

**不直接 grep 源码或 Read 源码起手——那是绕过文档、丢掉文档的优势。** 文档的作用是告诉你**去哪看**（哪个模块、哪个机制、哪段代码），源码只在文档把位置指出来之后，去**确认某个具体细节**。用户问"这个 agent 怎么实现 X"，你的路径是顺着文档三层往下钻，不是一头扎进源码。

这正是文档存在的意义：让人（和模型）不必读遍源码就能定位和读懂，源码只在"确认这一个具体值/行为"时才去读文档指了的那一段。

## 三层结构（和产出方 /analyzer 对齐）

- **OVERVIEW.md**：人话总览 + 端到端例子。**先读它**建立全局印象、定位问题大致落在哪个模块/机制。它是给人看的入口。
- **MAP.md / map.json**：树结构，每个节点挂 `code_ref`/`@行号` 指针 + 下游 `outputs.to` 链接。用来定位**具体哪个节点、哪段代码**。map.json 是机器可读底座（整棵树），MAP.md 是它的渲染。
- **detail/*.md**：分层节点的机制细节（含变量/函数的人话 gloss）。拿到该模块的具体逻辑、变量含义、控制流。

## 回答流程

1. **确认目标**：用户若没给仓库路径，问；或从 `.analyzer/map.json` 的 `_meta.target_repo` 取。
2. **读 OVERVIEW.md**：建立全局，定位问题落在哪个模块/机制。OVERVIEW 缺失（旧分析、narrator 还没跑）→ 跳过该层，直接从 MAP.md/map.json 起。
3. **钻 MAP**：按 OVERVIEW 指的节点，Read MAP.md 对应节点段（或 map.json 的该 node）拿到 `code_ref`/`@行号` + 下游链。
4. **钻 detail**：若需机制细节，Read 对应 `detail/*.md`（MAP.md 会写"细节见 detail/xxx.md"）。这里有变量/函数的人话解释——很多"X 变量什么意思"在这一层就有答。
5. **源码确认（必要时）**：若 detail 仍不足以确认某个**具体行为/值**（某阈值的确切条件、某分支的返回值、某常量）→ 用 detail 给的 `code_ref`/`@行号` Read 源码**那一小段**确认。**只读文档指了的那段，不发散 grep 全仓。**
6. **回答**：用人话答，**标注证据来自哪层**——"据 OVERVIEW 的'关键机制'段 / detail/autobacktest.md 的 `_filter_click` 段 / 源码 AutoBackTest.py:306-376 确认"。

## 铁律

1. **先文档后源码**——不直接读源码起手；源码只读文档指针指的那段，不发散。
2. **回答标注证据层**——文档为主、源码为辅确认，让人能复核。
3. **文档未覆盖就明说**——"`.analyzer/` 文档未覆盖 X"，可建议补跑 `/analyzer` 补该节点，或指出可能相关的源码位置问人要不要看。别假装文档里有。
4. **不改任何 `.analyzer/` 文件**——只读。要补/更新文档用 `/analyzer`，不在这里改。

## 边界

本 skill 是**阅读向导**：读文档、答问题、必要时读文档指了的源码段确认。它**不重新分析、不补节点、不改文档**。要产出/更新 `.analyzer/` → 用 `/analyzer`。
