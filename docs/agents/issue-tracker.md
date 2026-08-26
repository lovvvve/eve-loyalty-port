# 问题跟踪器：本地 Markdown

本仓库的问题和规格以 Markdown 文件形式保存在 `.scratch/` 中。

## 约定

- 每项功能使用一个目录：`.scratch/<feature-slug>/`
- 规格文件为 `.scratch/<feature-slug>/spec.md`
- 实施工单分别保存为 `.scratch/<feature-slug>/issues/<NN>-<slug>.md`，从 `01` 开始编号；不使用单个合并工单文件
- 分诊状态记录在每个问题文件顶部附近的 `Status:` 行中（角色字符串见 `triage-labels.md`）
- 评论和对话历史追加到文件底部的 `## Comments` 标题下

## 当技能要求“发布到问题跟踪器”时

在 `.scratch/<feature-slug>/` 下创建新文件，并按需创建目录。

## 当技能要求“获取相关工单”时

读取所引用路径的文件。用户通常会直接提供文件路径或问题编号。

## Wayfinding 操作

供 `/wayfinder` 使用。**地图（map）**为每个工单配有一个**子文件（child）**。

- **地图（Map）**：`.scratch/<effort>/map.md`，正文包含 `Notes`、`Decisions-so-far` 和 `Fog`
- **子工单（Child ticket）**：`.scratch/<effort>/issues/NN-<slug>.md`，从 `01` 开始编号，正文记录问题；`Type:` 行记录工单类型（`research`/`prototype`/`grilling`/`task`），`Status:` 行记录 `claimed`/`resolved`
- **阻塞（Blocking）**：文件顶部附近使用 `Blocked by: NN, NN`；列出的所有文件均为 `resolved` 后，该工单解除阻塞
- **前沿（Frontier）**：扫描 `.scratch/<effort>/issues/`，查找处于打开、未阻塞且未认领状态的文件；编号最小者优先
- **认领（Claim）**：先将 `Status:` 设置为 `claimed` 并保存，再开始工作
- **解决（Resolve）**：将答案追加到 `## Answer` 标题下，把 `Status:` 设置为 `resolved`，然后在 `map.md` 的 `Decisions-so-far` 中追加上下文指针（摘要和链接）
