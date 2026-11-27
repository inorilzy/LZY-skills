# Auto-DAG 模板参考

本文件供 `auto-dag` 队长写 Spec 与 Tickets 时引用。

---

## 1. Spec 模板（`.scratch/<slug>/spec.md`）

```markdown
# <Feature Title>

## Problem Statement

从用户视角描述要解决的问题。

## Solution

从用户视角描述方案。

## User Stories

1. As an <actor>, I want <feature>, so that <benefit>.
2. ...

## Implementation Decisions

- 需要新增或修改的模块职责与边界
- 模块间接口、数据契约、状态流转
- 需求讨论中确认的技术与交互决策
- （不写死具体文件路径或大段实现代码；原型敲定的关键 Schema / 状态机可精简内联）

## Testing Decisions

- 最高层测试接缝（Testing Seams）：优先验证外部行为而非内部实现
- 需要测试的模块与既有测试参考（prior art）

## Acceptance & Platform Scope

- **UI 形态**：`web` | `desktop` | `both` | `none`
- **当前宿主 OS（优先全量验收）**：`<例如 macOS>` —— 需启动真实运行态并截图自检的关键页面/交互流
- **其他目标平台（本次延迟实测）**：`<例如 Windows, Linux，无则写 None>` —— 本次执行平台无关单测 + 静态排查，实机验收标为 `NOT RUN (deferred: <OS>)`

## Out of Scope

- 明确本次不做的内容（防止下游 Agent 范围蔓延）

## Further Notes

其他补充说明。
```

---

## 2. 本地 Ticket 模板（`.scratch/<slug>/issues/NN-<slug>.md`）

> 严格匹配 `dsh-ticket-dag` 中 `src/tracker.mjs` 的解析规则：
> - 文件名：`01-xxx.md`、`02-xxx.md`（两位数字前缀，按依赖序排列）
> - `**Blocked by:**` 必须引用前置票的数字编号（如 `01, 02`）或 `None`
> - 验收项放在 `## Acceptance criteria` 下，使用 `- [ ]` 复选框

```markdown
# <NN>: <Ticket Title>

**What to build:** 从用户或契约视角描述本张票交付的端到端行为，避免写成逐层代码清单。

**Blocked by:** None | 01, 02

**Status:** ready-for-agent

## Scope & Boundaries

- 建议角色预设：`architect` | `implementer` | `frontend` | `scribe` | `verifier`
- 关联的 Spec 决策与边界说明（不包含 Out of Scope）

## Acceptance criteria

- [ ] 可验证的外部行为或契约验收条件 1
- [ ] 可验证的外部行为或契约验收条件 2
```

### 末尾验收票专用补充（当 `UI 形态 != none` 或有可运行宿主产物时必建）

- 编号为最后一张票（如 `06-acceptance-current-os.md`），`**Blocked by:**` 列出所有功能票与收口票编号。
- 验收条件必须包含：
  - [ ] 在当前宿主 OS 启动或刷新真实运行态并走通核心交互流（Web 检查控制台无报错；桌面端用 `agent-desktop` 校验界面元素）
  - [ ] 将关键画面截图保存至**主仓库根目录**下的 `<mainRepo>/.scratch/<slug>/acceptance/`（遵循协调器注入的非提交产物目录，勿留在临时 worktree 中），并用 `read_image` 逐张核对布局、对齐、文案与数据展示无异常
  - [ ] 验收报告写入 `<mainRepo>/.scratch/<slug>/acceptance/report.md`，每条验收项附实际命令/操作、结果（`VERIFIED` / `PARTIAL` / `MISSING` / `NOT RUN`）与截图绝对路径
  - [ ] （若有多平台目标）完成平台无关单测与跨平台静态排查，未在场 OS 标 `NOT RUN (deferred: <OS>)` 并附后续验收清单
  - [ ] 清理本次验收启动的临时进程，并确认工单 worktree 中无残留未追踪/被忽略的临时截图文件
