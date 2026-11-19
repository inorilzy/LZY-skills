---
name: auto-dag
description: "需求讨论完成后全自动执行：用 workflow 串联生成本地 spec 与 DAG tickets 并自检（无中途人工确认），再交由 dsh-ticket-dag 右侧审批并完成当前系统优先的视觉截图验收。用户说 /auto-dag 或要求自动生成 spec、拆票并跑 agent teams 时使用。"
---

# Auto-DAG：讨论后全自动 Spec → Tickets → Agent Teams → 视觉验收

将已讨论清楚的需求，无中途打断地自动转化为本地 `.scratch/<slug>/` 规格与工单 DAG，导入 `dsh-ticket-dag` 交由用户在右侧边栏做唯一一次审批（调整并发与模型），随后并行实现、在当前操作系统优先完成 UI/功能真机截图验收并交付。

> 所有模板、格式协议与质检清单见同目录 [`TEMPLATES.md`](TEMPLATES.md)；启动 workflow 前先读取该文件，将对应模板内联传给子 Agent。

## 前置边界

- **前置输入**：默认当前对话已经完成需求讨论。只有在存在会改变接口/行为的关键歧义时才先追问；一旦需求清楚，全程不再打断用户确认 Spec 或拆票方案。
- **仅写本地**：Spec 与 Tickets 一律落盘到仓库内的 `.scratch/<slug>/`，不创建或修改 GitHub Issue。
- **唯一人工节点**：`dag_run` 返回 `mode: staged` 后，在对话中通知用户到右侧 `「工单 · <runId>」` 调整并发数（`maxWorkers`）和模型并点击审批。

---

## 执行步骤

### 步骤 1：整理需求上下文并启动 `workflow` 自动流水线

读取 [`TEMPLATES.md`](TEMPLATES.md)，从当前对话提炼一份精简的**需求事实摘要**（包含目标、关键决策、明确非目标 Out of Scope、界面形态 Web/桌面客户端、目标平台清单），然后调用一次 `workflow` 工具，串联以下 3 个阶段：

1. **阶段 A · 探索与生成 Spec（Architect 子 Agent）**
   - 探索仓库代码结构、测试框架、构建脚本与现有模式（遵循项目术语与既有设计）。
   - 按 `TEMPLATES.md` 的 Spec 模板生成并写入 `.scratch/<slug>/spec.md`。
   - 必须写清最高层测试接缝（Testing Seams）、`Out of Scope`，以及 `Acceptance & Platform Scope`（标明 UI 类型：Web/浏览器、桌面客户端或无 UI；标明目标平台及当前宿主 OS 优先验收策略）。
   - 禁止在 Spec 中写死易腐化的实现代码片段（原型敲定的状态机/Schema/接口定义除外）。

2. **阶段 B · 拆分为 DAG Tickets（Planner 子 Agent）**
   - 读取 `.scratch/<slug>/spec.md` 与仓库目录结构，按 `TEMPLATES.md` 的工单模板逐一写入 `.scratch/<slug>/issues/NN-<slug>.md`（从 `01` 编号，上游前置票排前）。
   - **垂直切片优先**：每张功能票切一条端到端可独立验证的窄切片，大小适配单个上下文窗口；宽重构改用 **Expand → Migrate → Contract**。
   - **并行防冲突规则**：
     - 多张并行票共用新接口/数据契约/骨架时，第 `01` 票设为架构基础票（后续分配 `architect`）。
     - 多张并行票都要往同一共享文件登记（路由表、导出索引、配置清单）时，把登记动作抽成独立收口票依赖它们（后续分配 `scribe`）。
   - **验收票自动封尾**：只要 `spec.md` 涉及 UI、交互或可运行产物，**必须在末尾追加一张独立验收票**（依赖所有功能与收口票，后续分配 `verifier`），写明当前宿主 OS 的真机运行与截图验收清单，以及非当前 OS 的降级检查要求。
   - **严格遵守 `dag_import` 解析格式**：依赖边用 `**Blocked by:** 01, 02`（引用数字编号）或 `**Blocked by:** None`，验收项全部写在 `## Acceptance criteria` 下的 `- [ ]` 列表。

3. **阶段 C · 对抗式质检与自动返工（Critic 子 Agent）**
   - 独立对照「需求事实摘要 + `spec.md` + 全部 `issues/NN-*.md`」执行 `TEMPLATES.md` 中的 5 项质检：需求覆盖与无私货、DAG 无环与最小真实依赖、并发共享文件碰撞、单窗口粒度、`dag_import` 格式与末尾验收票完整性。
   - 质检返回 `pass: false` 时，将缺陷列表交回阶段 B 自动修订一次，不再向用户提问确认。

---

### 步骤 2：导入 `dsh-ticket-dag` 并提交审批

`workflow` 完成后，队长立即执行：

1. 调用 `dag_import` 传入 `.scratch/<slug>`，确认 `dependencyMode` 为 `declared` 且无格式告警。
   - **严禁缩写或改写导入工单**：若 `dag_import` 返回内容过长被工具截断，先用 `read` 读取截断输出文件或逐一读取 `.scratch/<slug>/issues/NN-*.md`，绝不可把工单 `prompt` 缩成一句话或丢弃 `acceptance`、`source`。
2. 结合仓库实际结构与 CI/脚本配置，补全 manifest，并**优先使用 `import_from: ".scratch/<slug>"`（或 manifest 文件路径）提交给 `dag_run`**（只需按 `id` 补充 `agent`、`verify`、`files`，由插件自动无损继承每张票的完整 `title`、`prompt`、`acceptance`、`depends_on` 与 `source`）：
   - **角色预设（`agent`）**：基础契约/骨架 → `architect`；核心业务与测试 → `implementer`；UI/样式/前端测试 → `frontend`；文档/配置/共享文件登记 → `scribe`；末尾验收票 → `verifier`。
   - **变更范围（`files`）**：为每张票填写准确的仓库相对路径或 glob；若出现可避免的并发 `fileOverlaps`，优先通过拆分/依赖化解，无法避免时在汇报里说明原因。
   - **验证命令（`checks` 与 `check_meta`）**：从仓库真实配置（`package.json`、测试框架、lint/typecheck）派生快速精准的校验命令，为每个 check 标注 `files`（宁宽勿窄）与 `fast: true|false`。末尾 `verifier` 票可不绑额外 shell check 或仅绑轻量检查，由 `verifier` 亲自执行真实宿主验收。
   - **并发度（`max_workers`）**：显式设置合理值（考虑仓库规模与文件重叠），告知用户可在侧边栏自由调整。
3. 调用 `dag_run`。当返回 `mode: staged` 时：
   - 向用户简洁列出：生成的 Spec 路径、工单清单与依赖概览、标记为 `fast` 的检查项、默认 `max_workers`、以及验收票将覆盖的当前 OS 范围。
   - 提示用户前往右侧边栏 `「工单 · <runId>」` 核对计划、按需调整并发数与模型并点击「确认并启动」，随后**结束当前回合**等待通知。

---

### 步骤 3：验收闭环与缺陷修复

当 `dsh-ticket-dag` 通知运行进展或完成时：

1. **视觉与当前 OS 优先验收准则**（已由末尾 `verifier` 票执行，队长负责把关）：
   - **当前宿主 OS 优先实测**：
     - **Web / 浏览器 / DSH Web 插件**：刷新或拉起真实运行态（DSH 客户端插件注意检查构建产物并刷新当前 Web GUI），使用浏览器工具（如 `cloak-browser-auth` 或仓库自带 Playwright）走完核心交互、检查控制台无报错并截图。
     - **桌面客户端**：在当前 OS 启动应用，使用 `agent-desktop`（`desktop_snapshot`、`desktop_find`、`desktop_click`、`desktop_screenshot` 等）校验界面元素与真实窗口并截图。
     - **进程清理**：验收拉起的临时服务或应用进程在截完图后须清理干净。
   - **截图存放与 AI 亲眼自检**：非提交类二进制截图统一写入**主仓库根目录**下的 `<mainRepo>/.scratch/<slug>/acceptance/`（遵循 coordinator 注入的 `Non-committed acceptance artifacts` 路径，避免留在临时 worktree 中阻断自动 cleanup 或导致主仓库找不到图）。`verifier` 必须对每张截图调用 `read_image` 亲自检查布局、对齐、间距、文字截断与数据展示。
   - **多系统软件（Windows / Linux / macOS）**：当前宿主 OS 项必须 `VERIFIED`；其余未在场 OS 跑平台无关单测与跨平台静态排查（路径分隔符、换行符、OS 专属 API），在验收记录中标记 `NOT RUN (deferred: <OS>)` 并附可复用的后续检查清单。
2. **发现缺陷自动补票**：若 `verifier` 报告任何 `PARTIAL` 或 `MISSING` 项，队长通过 `dag_add` 追加最小修复票（`debugger` 或 `frontend`）+ 一张依赖它的复验票（`verifier`），直至当前 OS 验收项全部 `VERIFIED`。

---

### 步骤 4：合回分支、队长复核与对话内展示交付

全部工单合并且验收通过后：

1. **确认并处理集成分支**：`dsh-ticket-dag` 默认把结果落在 `integration/<runId>`，不会自动改动主工作分支。若当前工作树干净且可快进合回（或经用户约定直接合回当前分支），执行 `git merge --ff-only integration/<runId>`（如已分叉则明确告知用户集成分支名并询问合并方式），避免出现「显示已完成但工作区看不到新代码」。
2. **队长亲眼复核截图与报告**：读取主仓库 `.scratch/<slug>/acceptance/` 下的验收报告（以及提交进仓库的 `docs/acceptance/<slug>.md`），并对最终截图亲自调用 `read_image` 复核视觉效果无异常。
3. **可靠交付截图（避免「图片无法预览」）**：
   - 调用 `present` 工具，以**绝对路径**呈递最关键的验收截图文件卡片（单次最多 4 张），确保用户在 Web GUI 下一定可直接预览与原生打开；
   - 如需在正文内联展示，一律使用 `<绝对路径>`（严禁使用在切换 cwd 后易失效的相对路径）；
4. **在最终回复中给出**：
   - **交付摘要**：集成分支及合并状态、已合并工单结果、验证命令通过情况；
   - **验收结论与截图说明**：对应下方呈递的截图卡片与验收报告链接；
   - **跨平台待办清单**（仅多平台项目）：清晰列出 `NOT RUN (deferred: Windows/Linux)` 项及后续到目标系统后的复验步骤。
