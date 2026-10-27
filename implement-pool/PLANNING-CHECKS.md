# Planning regression checks

Use these cases when changing the planning/approval contract in SKILL.md. These are document-level expected behaviors, not evidence of a live harness run. Runtime UI, route validation and scheduling still require integration testing.

| Input / state | Required behavior |
|---|---|
| New invocation, spec and tickets available, no model requested | Stage the complete assigned plan with approval required and omitted route fields; create no execution goal; end for UI review. |
| User wants to select models in the plan UI | Preserve user selection in the UI; do not require catalog discovery or a chat model choice before ordinary staging. |
| Default route rejected before plan save | Inspect team state, quote the exact error once, explain inherited route versus explicit selection, stop without fallback or a new goal. |
| Plan exists despite a creation error | Preserve it; follow the permitted correction path rather than creating a duplicate. |
| Missing assignee and no team created | Correct arguments and retry once under rule 8, preserving routes, size and approval. |
| Capacity or service error | Stop; no smaller pool or one-off subagent fallback. |
| User edits models but has not approved | Keep plan staged; no goal activation or execution. |
| Later explicit approval | Read current plan, preserve model edits, create/resume the matching goal, then approve if not already running. |
| Web Approve & Run already started team | Confirm state, activate the goal, continue the existing team without a second approval call. |
| Explicit immediate-run request | Successful immediate team creation permits execution and goal activation; no extra review round. |
| Automatic goal continuation while awaiting approval | No dispatch or repeated approval questions; obey goal-control permissions. |
| User asks to update/push this skill | Edit the skill only; do not resume the implementation goal. |

## Scope of this change

Frontmatter and invocation routing are unchanged. Existing worktree isolation, merge-before-dependency completion, fixed pool size, and deployment authorization remain in force. A documentation change does not fix a runtime that validates inherited routes before displaying the plan.
