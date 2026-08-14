---
name: analyzer-docs
description: 回答一个已经由 analyzer 分析过的 agent 代码库问题。先读目标仓库 .analyzer/ 的 OVERVIEW、MAP/map.json、detail，再按证据指针读取少量源码确认。用户问“它怎么实现 X”“X 在哪”“为什么这样设计”或变量含义时使用。
---

# analyzer-docs — 分析文档阅读向导

只读，不修改 `.analyzer` 或源码。若目标没有 `.analyzer/`，说明需要先运行 `$analyzer`。

## 顺序

1. 从用户给出的路径确定目标；未给路径时，优先从当前仓库 `.analyzer/map.json._meta.target_repo` 判断，仍不明确再询问。
2. 完整读取 `OVERVIEW.md`，建立整体数据流并定位主题。
3. 读取 `MAP.md` 对应节点；需要机器字段时读取 `map.json` 的该节点。
4. 读取 MAP 指向的 `detail/*.md`，取得变量 gloss、分支条件和证据指针。
5. 只有文档不足以确认具体阈值、返回值或条件时，才读取 `code_ref` 指向的源码小段；不要发散搜索全仓。
6. 用大白话回答，并标注证据层：OVERVIEW、MAP/detail 节点，必要时源码 path:line。

## 边界

- 文档未覆盖就明说，不假装已有结论。
- 用户要求更新分析产物时切换到 `$analyzer`，本 skill 不写文件。
- 优先使用 `rg`/`rg --files`，但源码检索必须从文档指针开始。
