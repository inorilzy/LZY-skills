---
name: auto-dag
description: "需求讨论完成后全自动执行：队长直接写本地 spec 与 DAG tickets，交由 dsh-ticket-dag 右侧审批，并行实现后完成当前系统优先的视觉截图验收。小改动直接实现不走 DAG。用户说 /auto-dag 或要求自动生成 spec、拆票并跑 agent teams 时使用。"
---

# Auto-DAG：讨论后 Spec → Tickets → Agent Teams → 视觉验收

把已讨论清楚的需求转成本地 `.scratch/<slug>/` 规格与工单，导入 `dsh-ticket-dag`，用户在右侧边栏做唯一一次审批，随后并行实现、当前 OS 优先截图验收并交付。

模板与导入格式见同目录 [`TEMPLATES.md`](TEMPLATES.md)。

## 前置边界

- 默认需求已讨论完。只有存在会改变接口/行为的关键歧义时才追问；之后不再让用户确认 Spec 或拆票。
- Spec 与 Tickets 只写本地 `.scratch/<slug>/`，不创建或修改 GitHub Issue。
- 唯一人工节点：`dag_run` 返回 `mode: staged` 后的侧边栏审批。

## 步骤 0：规模门槛

先估算拆票数量。预计 ≤ 3 张票、或只改动单个模块时，**不走 DAG**：队长直接实现，或用 `dag_preset_subagent` 委派一个 `implementer`/`frontend`，完成后按 AGENTS.md 自行测试与截图自检，然后交付。告诉用户已因规模小跳过 DAG。

只有剩下的情况才继续下面的步骤。

## 步骤 1：队长直接写 Spec 与 Tickets

队长持有完整讨论上下文，由队长亲自写，不再用 workflow 或子 Agent 串联生成。

1. 了解仓库结构、测试框架和构建脚本。仓库较大时可用 `dag_preset_subagent` 委派一个 `explorer` 只读取事实（路径、脚本、既有模式），队长据此落笔。
2. 按 `TEMPLATES.md` 写 `.scratch/<slug>/spec.md`，写清 Testing Seams、`Out of Scope`、`Acceptance & Platform Scope`。不写易腐化的实现代码。
3. 按 `TEMPLATES.md` 写 `.scratch/<slug>/issues/NN-<slug>.md`：
   - **垂直切片优先**，每张票可独立验证、适配单个上下文窗口；宽重构用 Expand → Migrate → Contract。
   - **只写真实依赖**，相互独立的切片保持并行。
   - **架构票**：仅当 ≥ 2 张并行票确实依赖同一新契约时才建（`architect`）；否则不建。
   - **收口票**：仅当 ≥ 2 张并行票确实要改同一个注册/索引文件时才建（`scribe`）；否则不建。
   - **验收票**：涉及 UI、交互或可运行产物时，末尾建一张 `verifier` 票，依赖所有功能票。
4. 写完后队长自查一遍：讨论中的每条关键诉求都落到至少一张票的验收项，且没有引入 Out of Scope 内容。格式、环和文件冲突交给步骤 2 的工具结果检查，不另开质检 Agent。

## 步骤 2：导入并提交审批

1. 调用 `dag_import` 传入 `.scratch/<slug>`。必须是 `dependencyMode: declared` 且无告警；有问题就改票文件后重新导入。不要缩写导入的工单 prompt。
2. 用 `import_from: ".scratch/<slug>"` 调用 `dag_run`，只按 `id` 补充：
   - `agent`：契约/骨架 → `architect`；业务与测试 → `implementer`；UI → `frontend`；文档/配置/登记 → `scribe`；验收 → `verifier`。
   - `files`：仓库相对路径或 glob。结果里出现可避免的 `fileOverlaps` 时，调整依赖或拆分后重新提交；无法避免的在汇报里说明。
   - `checks` / `check_meta`：从仓库真实配置派生快速命令，`files` 宁宽勿窄，标注 `fast`。验收票不绑 check 或只绑轻量 check。
   - `max_workers`：显式设置。
3. 返回 `mode: staged` 后，向用户简要列出：Spec 路径、工单与依赖概览、fast 检查、`max_workers`、验收票覆盖的 OS；提示到右侧 `「工单 · <runId>」` 审批，然后结束回合。

## 步骤 3：验收与缺陷修复

验收由末尾 `verifier` 票完成：

- **当前宿主 OS 实测**：Web/DSH 插件用浏览器工具（`cloak-browser-auth` 或仓库自带 Playwright）走核心交互、看控制台并截图；DSH 客户端插件先确认构建产物再刷新当前 Web GUI。桌面端用 `agent-desktop` 校验界面并截图。截完图清理临时进程。
- **截图与报告**：写入**主仓库根目录**下的 `.scratch/<slug>/acceptance/`（截图 + `report.md`），不留在临时 worktree。verifier 对每张截图调用 `read_image` 亲自检查。报告每条验收项标 `VERIFIED` / `PARTIAL` / `MISSING` / `NOT RUN`。
- **多平台项目**：当前 OS 项必须 `VERIFIED`；其他 OS 跑平台无关单测和跨平台静态排查，标 `NOT RUN (deferred: <OS>)` 并附后续检查清单。

verifier 报告 `PARTIAL` / `MISSING` 时，用 `dag_add` 追加最小修复票（`debugger` 或 `frontend`）。只有修复改动到界面或运行态行为时，才再追加一张依赖它的 `verifier` 复验票；纯逻辑修复靠 check 通过即可。

## 步骤 4：合回分支与交付

1. 结果在 `integration/<runId>`。工作树干净且可快进时执行 `git merge --ff-only integration/<runId>`；已分叉就告诉用户分支名并询问合并方式。
2. 读取 `.scratch/<slug>/acceptance/report.md`。截图已由 verifier 逐张检查，队长只抽看最终关键的 1–2 张。
3. 用 `present` 以绝对路径呈递关键截图（最多 4 张）；正文内联图片也用 `<绝对路径>`。
4. 最终回复给出：集成分支与合并状态、已合并工单、验证命令结果、验收结论与截图说明；多平台项目另列 `NOT RUN (deferred: ...)` 待办及复验步骤。
