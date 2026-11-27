---
name: auto-dag
description: "需求讨论完成（或已有 PRD/spec）后全自动执行：队长写一页本地 spec，把工单直接提交给 dsh-ticket-dag 右侧审批，并行实现、合回后由队长做当前系统截图验收。小改动直接实现不走 DAG。用户说 /auto-dag 或要求自动拆票并跑 agent teams 时使用。"
---

# Auto-DAG：讨论 → 一页 Spec → Agent Teams → 截图验收

把已讨论清楚的需求拆成工单交给 `dsh-ticket-dag` 并行实现，用户只在右侧边栏审批一次，合回后队长亲自验收并交付。

## 前置边界

- 需求来源：当前对话的讨论结论，或用户给出的 PRD / 已有 `.scratch/<slug>/spec.md`。只有存在会改变接口/行为的关键歧义时才追问，之后不再打断。
- 本地文件沿用 Matt Pocock 技能的 tracker 约定：`.scratch/<slug>/`。不创建或修改 GitHub Issue。
- 唯一人工节点：`dag_run` 返回 `mode: staged` 后的侧边栏审批。

## 步骤 0：规模门槛

预计 ≤ 3 张票、或只改动单个模块时，**不走 DAG**：队长直接实现，或用 `dag_preset_subagent` 委派一个 `implementer`/`frontend`，完成后按 AGENTS.md 测试与截图自检再交付。告诉用户已因规模小跳过 DAG。

## 步骤 1：准备一页 Spec

所有工单共享的上下文只写一次，放在 `.scratch/<slug>/spec.md`（文件名固定为 `spec.md`，让 `to-tickets`、`code-review` 等技能能识别）：

- 已有 `.scratch/<slug>/spec.md`（如 `/to-spec` 产出）：直接复用，不重写。
- 有 PRD：把 PRD 路径写进 spec 顶部，spec 只补 PRD 没写的已定技术决策；PRD 已足够时 spec 只写一行指向它。
- 只有对话：把讨论结论写成一页，使用下面的格式。

```markdown
# <标题>

## 目标
一两句话说明要解决什么、做成什么样。

## 已定决策
- 讨论中确认的行为、接口、数据结构、技术选型

## 验收要点
- 可观察的外部行为；涉及 UI 时写清当前 OS 要截图的页面/交互

## Out of Scope
- 明确不做的内容
```

不写易腐化的实现代码。仓库较大时可先用 `dag_preset_subagent` 派一个 `explorer` 只读查路径、脚本和既有模式。

## 步骤 2：拆票并提交审批

队长读仓库后直接把工单写进 `dag_run` 的 manifest，不写 `issues/` 文件，不调 `dag_import`。

- `spec_id`：用同一个 `<slug>`（协调器的验收目录 `.scratch/<spec_id>/acceptance` 会落在同一文件夹）。
- 每张票：`id`、`title`、`prompt`、`acceptance`、`depends_on`、`agent`、`files`、`verify`。
  - `prompt` 自包含本票要做什么，并写明「共享上下文见 `.scratch/<slug>/spec.md`」（及 PRD 路径）；不在多张票里重复抄 spec。
  - **垂直切片优先**，每张票可独立验证、适配单个上下文窗口；宽重构用 Expand → Migrate → Contract。
  - `depends_on` 只写真实依赖，相互独立的切片保持并行。
  - `agent`：契约/骨架 → `architect`；业务与测试 → `implementer`；UI → `frontend`；文档/配置/登记 → `scribe`。
  - **架构票**仅当 ≥ 2 张并行票确实依赖同一新契约时才建；**收口票**（`scribe`）仅当 ≥ 2 张并行票确实要改同一注册/索引文件时才建。
- `checks` / `check_meta`：从仓库真实配置派生快速命令，`files` 宁宽勿窄，标注 `fast`。
- `max_workers`：显式设置。
- 结果里出现可避免的 `fileOverlaps` 时调整依赖或拆分后重新提交；无法避免的在汇报里说明。

返回 `mode: staged` 后，向用户简要列出：spec 路径、工单与依赖概览、fast 检查、`max_workers`；提示到右侧 `「工单 · <runId>」` 审批，然后结束回合。

## 步骤 3：合回与队长验收

全部工单集成后：

1. 结果在 `integration/<runId>`。工作树干净且可快进时执行 `git merge --ff-only integration/<runId>`；已分叉就告诉用户分支名，询问合并方式后再继续。
2. 涉及 UI 或可运行产物时，队长在主仓库实测当前 OS：
   - Web / DSH 插件：用浏览器工具（`cloak-browser-auth` 或仓库自带 Playwright）走核心交互、看控制台并截图；DSH 客户端插件先确认构建产物再刷新当前 Web GUI。
   - 桌面端：用 `agent-desktop` 校验界面并截图。
   - 截图存到 `.scratch/<slug>/acceptance/`，逐张用 `read_image` 检查；截完图清理临时进程。
   - 多平台项目：其他 OS 跑平台无关测试和跨平台静态排查，标 `NOT RUN (deferred: <OS>)` 并给出后续检查清单。
3. 发现问题：小问题队长直接修；较大问题用 `dag_add` 追加最小修复票（`debugger` 或 `frontend`），集成后再合回并复测相关画面。

## 步骤 4：交付

- 用 `present` 以绝对路径呈递关键截图（最多 4 张）；正文内联图片也用 `<绝对路径>`。
- 最终回复给出：spec 路径、集成分支与合并状态、已合并工单、验证命令结果、验收结论；多平台项目另列 `NOT RUN (deferred: ...)` 待办。
