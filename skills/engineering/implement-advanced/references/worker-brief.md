# Worker brief (one ticket, one subagent)

Fill `{TICKET}` with the issue path and `{PHASE_SPEC}` with the phase spec path, then paste the whole brief as the subagent prompt.

---

Implement `{TICKET}` in this repo. Read `{PHASE_SPEC}`, `CONTEXT.md` (domain vocabulary; never use its listed avoid-terms), and the ticket's full body plus comments first.

Follow the `implement` skill: TDD at pre-agreed pure seams only (colocated vitest in the style of the existing suites — observable behavior, never implementation details); database, session, render, byte, and network paths are verified by hand against a dev database, never unit-tested. Reuse existing seams; never re-derive a locked rule owned by another module. No schema redesign unless the ticket demands it; ORM only; no secrets in code.

Run typechecking regularly, single test files regularly, and the full test suite plus the production build once at the end. If no local database exists, start one and run migrations plus seed (leave it running); remove any throwaway fixtures afterwards.

Run the dual-axis review on your own diff before finishing. Do NOT commit anything.

Return in your final message: files created and modified, design decisions with edge resolutions, test counts, typecheck/lint/build results, first-hand bilingual verification matrix where the ticket requires it, and anything deliberately deferred.
