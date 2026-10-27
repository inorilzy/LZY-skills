---
name: implement-pool
description: "Implement the result of /to-spec and /to-tickets in code: goal-mode captain running a fixed pool of agent team members until every ticket is merged."
disable-model-invocation: true
---

You have been provided a spec. This spec should have tickets associated with it, describing how to implement the spec.

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`.

The goal is the entire spec implemented on a single **integration branch**, with every ticket resolved the way the issue tracker closes work.

You are the **captain**. Once the user has approved the team plan (step 2), you run unattended: the user may be away for hours, so the run keeps moving on its own until every ticket is `merged` or `blocked`, and the user hears from you once, at the end.

## Vocabulary

- **Agent team**: one persistent team with a roster and a shared task board (e.g. AgentTeams), led by the captain. The explorer, every implementer, and the merger are **members** of this team. Invoking this skill is a request for an agent team: never run them as one-off subagents.
- **Task graph**: the tickets and their blocking relationships, mirrored on the team's task board as one task per ticket, with each ticket's blockers as its task's dependencies. A ticket is **ready** when every blocker is `merged` and it is not `running`.
- **Pool**: at most **N** implementer members running at once (default 8; use the user's number if given). A **slot** is one place in the pool. The whole roster (N implementers, the merger, and the explorer if used) must fit the harness's team member limit.
- **Run directory**: a directory outside the repo shared by the captain and all members. It holds `ledger.md`, `decisions.md`, and exploration notes.
- **Ledger** (`ledger.md`): one row per ticket: status (`pending | running | merged | blocked`), branch, worktree, attempt count, start time, blocked reason.
- **Decision log** (`decisions.md`): one entry per non-obvious decision: question, choice, reason, how to reverse.

Communication to and from members should be sparse. Communicate primarily through **context pointers**: to the spec, tickets, run directory, and previous commits. Don't duplicate information already available via pointers.

## Captain rules

1. **Goal mode.** Run as a persisted goal (e.g. `create_goal`) with the objective "Implement spec <pointer> on integration branch <name>; every ticket merged or blocked, code review fixed, worktrees cleaned up." If a goal for this spec exists, resume it. Mark it complete after step 9. Mark it blocked only when nothing is `pending` or `running` and the same concrete blocker persists across rounds.

2. **Captain owns dispatch.** You alone pick tickets, assign them, accept or reject reports, merge, and update the tracker and ledger. Implementers work only on the ticket they were given.

3. **Ledger is the truth.** Write every status change to the ledger as it happens. On every start or resume, reconcile the ledger against reality (branches and worktrees that exist, the team's members and tasks, commits already on the integration branch) before dispatching, so each ticket runs exactly once. If a team for this spec already exists, continue it; never create a second one.

4. **Keep the pool full.** Whenever a slot frees up, fill it with a ready ticket, and fill every free slot. Ready tickets come from anywhere in the graph, including tickets just unblocked by the latest merge. When ready tickets outnumber free slots:
   - prefer tickets on the longest dependency chain, then those with the most dependents;
   - run tickets that declare the same files or an exclusive resource (fixed port, local database, shared cache) one after another.

   Merges are yours, one at a time, and use no slot.

5. **Keep the integration branch green.** After each merge, run the test suite on the integration branch. On failure, pause filling slots (running implementers continue), fix or revert that merge, then resume dispatch.

6. **Stalls and failures.** When a ticket runs far longer than its peers (e.g. over 1 hour), ask its implementer for status and choose: continue, split, or re-dispatch. When a ticket fails twice, mark it and everything downstream `blocked` with the reason, and keep running the rest.

7. **Decide ambiguity and keep going.** Resolve every uncertain question immediately, by this split:
   - **Implementer decides** what is local to its ticket (naming, internal structure, edge cases, test shape).
   - **Captain decides** what crosses tickets or interprets the spec: unclear or conflicting requirements, a hidden dependency, an oversized ticket. Choose the option most consistent with the spec, then with existing codebase conventions, then the easiest to reverse. Update the graph if needed (add the dependency, split the ticket) and continue.

   Log every non-obvious decision in the decision log.

   The one exception is an action that is irreversible or reaches outside the repo: production, deleting data, credentials, security or access changes, paid services. Leave it undone, mark only that ticket `blocked`, keep everything else running, and carry the question to the final report.

8. **Agent team or stop.** For an explicit plan-argument validation error (such as a missing assignee or invalid dependency reference), inspect team state first. If nothing was created, correct the arguments and retry once; if a staged team exists, correct that existing plan instead of creating another team. Preserve roster size, model routes, scope, and approval requirements, and log the error and correction. If state is uncertain, correction fails, or the error concerns capacity, permissions, service availability, member creation, or an unknown cause, stop immediately: dispatch nothing, end the turn, and report the exact error and options. Do the same if no agent team mechanism exists. Never fall back to subagents or shrink N without the user's explicit choice. This overrides rule 7 and the "run unattended" default.

## Steps

1. **Start.** Create or resume the goal. Read the spec and tickets, build the task graph, create the run directory, and create or reconcile the ledger. Done when every ticket has a ledger row and the graph has no unknown blockers.

2. **Create the team.** Check that the whole roster fits the harness's team member limit, then create the agent team with it: N implementers, a merger, and an explorer if step 3 is needed. Put the task graph on the task board (one task per ticket, blockers as dependencies). When using AgentTeams batch plan creation, explicitly set every task's `assignee` to a member name in the roster and validate all assignments before submitting. Preassignment does not authorize execution; the approval requirement still applies. For auto-dispatching teams, include a preparation task owned by the merger as a prerequisite for implementation; it performs step 4 after approval. Each implementer's first action is the worktree readiness check in step 5. Present the team plan to the user and end the turn; never approve it yourself. Start dispatching only after the user explicitly approves it, unless the user explicitly asked to start without review. If any of this fails, follow rule 8.

3. **Explore** (optional). Use an **explorer member** for exploration the tickets require (relevant codebase files or external docs). It saves markdown notes in the run directory, so implementers can focus on implementation.

4. **Integration branch.** Create it. If the issue tracker closes work through PRs, or the user asks for one, open a draft PR after the first merge (a branch with no commits ahead of main can't open one), marked as closing the spec and tickets.

5. **Dispatch.** Fill free slots (rule 4) by assigning tickets as tasks on the team's task board. Before implementation starts, confirm the integration branch exists and the ticket's isolated worktree and branch are ready at its current tip. The captain may authorize the assigned implementer to create that worktree as its first action; preparation must finish before code edits or tests. Each **implementer member** works only in that worktree. Its brief is a pointer to its ticket, the spec, the run directory, and [`IMPLEMENTER.md`](IMPLEMENTER.md).

6. **Accept and merge.** When an implementer reports, check it against its ticket. If it failed or is incomplete, handle it by rule 6. If it raised a blocker or a captain-level question, decide by rule 7, update the graph and ledger, and free the slot. An implementer's `ready-for-merge` report leaves the ticket task unfinished and its downstream dependencies blocked. Once accepted, transfer the task to the **merger member** through the harness's reassignment mechanism; merge and run the integration tests (rule 5). Only after a successful merge and passing tests, mark the ledger `merged`, resolve the tracker, and complete the ticket task to release downstream dependencies. Remove its worktree and return to step 5.

7. **Drain.** Repeat steps 5–6. Done when no ticket is `pending` or `running`; anything left is `blocked` with a recorded reason.

8. **Review.** Call the Skill tool with `code-review` on the integration branch. Turn each finding into a small fix ticket and run them through the pool (steps 5–7). Done when every finding is fixed or recorded as `blocked`.

9. **Finish.** If a draft PR exists, mark it ready for review. Remove all remaining worktrees. Confirm every task on the board is finished and every member is idle, then close the agent team. Mark the goal complete and send one report:
   - tickets merged;
   - tickets blocked, with reasons;
   - a pointer to the decision log, with the decisions most worth the user's review;
   - questions only the user can answer;
   - the integration branch or PR.
