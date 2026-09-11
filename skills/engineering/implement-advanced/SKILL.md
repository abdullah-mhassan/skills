---
name: implement-advanced
description: "Work a phase tracker to done, one ticket per wave: dispatch each open unblocked ticket to a subagent, verify, review, and commit. Use when the user says implement all / work the phase / run the tickets."
disable-model-invocation: true
---

Work every open ticket in a phase issue tracker to `resolved`, strictly one ticket per wave. The main session is the boss: it never implements tickets itself, only dispatches, verifies, and commits.

## Loop

1. **Scan frontier.** Read `.scratch/<phase>/issues/`. The frontier is tickets whose `Status:` is still `ready-for-agent` and whose `Blocked by:` tickets are all `resolved`. Take the lowest-numbered one. If none is open, the run is done: report the phase complete.
2. **Dispatch one worker.** Spawn a single subagent for that ticket with the brief in [references/worker-brief.md](references/worker-brief.md), filled with the ticket path. One wave means one ticket; never dispatch a second while the first is unfinished.
3. **Verify first-hand.** Re-run the worker's gates yourself: typecheck, the touched single test files, the full suite, and the build. Trust the report, then confirm it.
4. **Review and fix.** Run the dual-axis review (standards plus spec) in parallel subagents. Apply real findings yourself; record dismissed ones with rationale in the ticket's `## Comments`.
5. **Close and commit.** Flip the ticket to `resolved`, append review notes and gate evidence to `## Comments`, and commit only that ticket's files to the current branch.
6. **Next wave or halt.** Rescan the frontier. Any red gate, failed review, or worker-reported blocker stops the line immediately: report which ticket is red and why, and dispatch nothing further until it is resolved.

## Rules

- Workers never commit; only the boss commits, one ticket per commit.
- Workers never touch files outside their ticket's scope; the boss restores anything stray before committing.
- A blocked ticket waits silently: it becomes frontier only when its blockers land.
