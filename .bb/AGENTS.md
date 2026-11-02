# Project agent workflow

Use the project skill `dev-loop` for every non-trivial implementation, refactor,
bug fix, or architectural change. A non-trivial task is one where design choices,
regression risk, or more than one plausible implementation deserves independent
review. A tiny, obvious, low-risk edit may stay with one agent.

When `dev-loop` applies, the coordinator must delegate the architect,
implementer, reviewer, and tester roles to subagents. Keep one writer at a time:
only the implementer may change project files during the loop. The architect,
reviewer, and tester are read-only unless the coordinator explicitly reassigns
ownership after the current writer has stopped.

Repository rules:

- Treat the live repository and the user's request as the sources of truth.
- Preserve user changes and avoid unrelated edits.
- Do not commit, push, deploy, publish, or change external systems unless the
  user explicitly requests it.
- Prefer the smallest solution that satisfies the acceptance criteria.
- Run focused tests first and the broadest relevant verification before
  declaring completion.
- Stop and ask the user only when a missing product or architecture decision
  would materially change the result. State reasonable, reversible assumptions
  when work can safely continue.
- Report remaining risks and failed checks; never describe an unverified result
  as complete.
