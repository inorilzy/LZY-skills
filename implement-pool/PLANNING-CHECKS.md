# Planning regression checks

Use these cases when changing the planning/approval contract in SKILL.md. These are document-level expected behaviors, not evidence of a live harness run. Runtime UI, route validation and scheduling still require integration testing.

| Input / state | Required behavior |
|---|---|
| New invocation, spec and tickets available, no model requested | Query current-session routes, verify an exact permitted candidate, explicitly set its model as an editable initial choice, stage with approval required, disclose the choice, and create no execution goal. |
| User wants to select models in the plan UI | Use a verified initial model to reach the staged UI; user may change it before approval. No preliminary chat model-choice question when a permitted candidate exists. |
| Verified route rejected before plan save | Inspect team state, quote the exact error once, distinguish explicit selection from inheritance, stop without repeated alternate-route attempts or a new goal. |
| Candidate detail query returns route denied | Exclude that discovery candidate and inspect other permitted candidates; name the failing tool/arguments, and do not invoke the team-creation stop rule for this query alone. |
| Captain uses magpie-responses; verified worker uses magpie | Treat provider IDs as distinct; ask once for explicit cross-provider authorization, then pass the authorized provider/model pair. Editing this skill is not execution authorization. |
| Catalog displays a full provider-prefixed route | Inspect the exact provider/model pair in the tool's expected format; avoid a doubled provider prefix. |
| Settings UI and session catalog differ | Use current-session inspection; explain settings may apply only to new sessions. |
| No permitted candidate, or only a different provider is available | Report the limitation or ask for required provider authorization; do not inherit a known rejected route. |
| User specified a route or edited a staged member | Preserve the selection; do not replace it with the captain's initial-model policy. |
| Plan exists despite a creation error | Preserve it; follow the permitted correction path rather than creating a duplicate. |
| Missing assignee and no team created | Correct arguments and retry once under rule 8, preserving routes, size and approval. |
| Capacity or service error | Stop; no smaller pool or one-off subagent fallback. |
| User edits models but has not approved | Keep plan staged; no goal activation or execution. |
| Later explicit approval | Read current plan, preserve model edits, create/resume the matching goal, then approve if not already running. |
| Web Approve & Run already started team | Confirm state, activate the goal, continue the existing team without a second approval call. |
| Initial request says "开始干" or asks for immediate execution | Stage with approval required, present the plan and end the turn; wait for later explicit approval or Web Approve & Run before goal activation or execution. |
| New or resumed roster | New members get unique short names rather than role-number labels; resumed members retain their existing names. |
| Automatic goal continuation while awaiting approval | No dispatch or repeated approval questions; obey goal-control permissions. |
| User asks to update/push this skill | Edit the skill only; do not resume the implementation goal. |

## Scope of this change

Frontmatter and invocation routing are unchanged. Existing worktree isolation, merge-before-dependency completion, fixed pool size, and deployment authorization remain in force. A documentation change does not fix a runtime that validates inherited routes before displaying the plan.
