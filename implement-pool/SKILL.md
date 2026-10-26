---
name: implement-pool
description: "Implement the result of /to-spec and /to-tickets in code: goal-mode captain running a fixed agent pool until every ticket is merged."
disable-model-invocation: true
---

You have been provided a spec. This spec should have tickets associated with it, describing how to implement the spec.

The issue tracker should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills`.

The goal is the entire spec implemented on a single **integration branch**, with every ticket resolved the way the issue tracker closes work.

You are the **captain**. You run unattended: the user may be away for hours, so the run keeps moving on its own until every ticket is `merged` or `blocked`, and the user hears from you once, at the end.

## Vocabulary

- **Task graph**: the tickets and their blocking relationships. A ticket is **ready** when every blocker is `merged` and it is not `running`.
- **Pool**: at most **N** implementers running at once (default 8; use the user's number if given). A **slot** is one place in the pool.
- **Run directory**: a directory outside the repo shared by the captain and all subagents. It holds `ledger.md`, `decisions.md`, and exploration notes.
- **Ledger** (`ledger.md`): one row per ticket: status (`pending | running | merged | blocked`), branch, worktree, attempt count, start time, blocked reason.
- **Decision log** (`decisions.md`): one entry per non-obvious decision: question, choice, reason, how to reverse.

Communication to and from subagents should be sparse. Communicate primarily through **context pointers**: to the spec, tickets, run directory, and previous commits. Don't duplicate information already available via pointers.

## Captain rules

1. **Goal mode.** Run as a persisted goal (e.g. `create_goal`) with the objective "Implement spec <pointer> on integration branch <name>; every ticket merged or blocked, code review fixed, worktrees cleaned up." If a goal for this spec exists, resume it. Mark it complete after step 8. Mark it blocked only when nothing is `pending` or `running` and the same concrete blocker persists across rounds.

2. **Captain owns dispatch.** You alone pick tickets, assign them, accept or reject reports, merge, and update the tracker and ledger. Implementers work only on the ticket they were given.

3. **Ledger is the truth.** Write every status change to the ledger as it happens. On every start or resume, reconcile the ledger against reality (branches and worktrees that exist, live subagents, commits already on the integration branch) before dispatching, so each ticket runs exactly once.

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

## Steps

1. **Start.** Create or resume the goal. Read the spec and tickets, build the task graph, create the run directory, and create or reconcile the ledger. Done when every ticket has a ledger row and the graph has no unknown blockers.

2. **Explore** (optional). Use an **exploration subagent** for exploration the tickets require (relevant codebase files or external docs). It saves markdown notes in the run directory, so implementers can focus on implementation.

3. **Integration branch.** Create it. If the issue tracker closes work through PRs, or the user asks for one, open a draft PR after the first merge (a branch with no commits ahead of main can't open one), marked as closing the spec and tickets.

4. **Dispatch.** Fill free slots (rule 4). Each **implementer subagent** runs in the background, in its own worktree on its own branch based on the current integration branch tip. Its brief is a pointer to its ticket, the spec, the run directory, and [`IMPLEMENTER.md`](IMPLEMENTER.md).

5. **Accept and merge.** When an implementer reports, check it against its ticket. If it failed or is incomplete, handle it by rule 6. If it raised a blocker or a captain-level question, decide by rule 7, update the graph and ledger, and free the slot. Once accepted, merge it with a **merger subagent**, run the tests (rule 5), mark it `merged`, resolve it in the tracker, and remove its worktree. Return to step 4.

6. **Drain.** Repeat steps 4–5. Done when no ticket is `pending` or `running`; anything left is `blocked` with a recorded reason.

7. **Review.** Call the Skill tool with `code-review` on the integration branch. Turn each finding into a small fix ticket and run them through the pool (steps 4–6). Done when every finding is fixed or recorded as `blocked`.

8. **Finish.** If a draft PR exists, mark it ready for review. Remove all remaining worktrees. Mark the goal complete and send one report:
   - tickets merged;
   - tickets blocked, with reasons;
   - a pointer to the decision log, with the decisions most worth the user's review;
   - questions only the user can answer;
   - the integration branch or PR.
