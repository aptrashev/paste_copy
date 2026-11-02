---
name: dev-loop
description: Coordinate architect, implementer, reviewer, and tester subagents for non-trivial code changes, with bounded review-and-fix iterations. Use when the user invokes /dev-loop, asks for an agent development loop, or requests a non-trivial implementation, refactor, bug fix, or architectural change in this project.
---

# Development loop

Own the final result as the coordinator. Use native subagent delegation tools;
do not simulate several roles inside one response when delegation is available.
Keep dependent phases sequential and parallelize only independent read-only
checks.

## Scale the team

- For a tiny, obvious, low-risk edit, skip the loop and explain that one agent
  is sufficient unless the user explicitly requested the full loop.
- For a normal non-trivial change, use architect, implementer, reviewer, and
  tester.
- Add a security specialist only for authentication, authorization, secrets,
  payments, untrusted input, privacy, or another meaningful security boundary.
- Add a UX or product specialist only when user-visible behavior or requirements
  are genuinely ambiguous.

## Run the loop

1. **Frame the task.** Inspect the repository, restate the objective, derive
   testable acceptance criteria, and identify explicit non-goals. Continue with
   clearly labeled reversible assumptions; ask the user only for a decision
   that would materially change the implementation.

2. **Architect — read-only.** Spawn one architect with the objective,
   constraints, acceptance criteria, relevant repository context, and this
   contract: inspect but do not edit. Require a concise design, affected areas,
   risks, migration or compatibility concerns, and a verification plan. Ask it
   to surface blocking decisions rather than invent product requirements.

3. **Implementation — sole writer.** After considering the architecture report,
   spawn one implementer. Give it the accepted plan, acceptance criteria,
   constraints, and validation commands. It owns all file edits for the loop,
   keeps the change focused, runs relevant tests, and reports files changed,
   commands run, failures, and assumptions. Do not let another agent edit files
   concurrently.

4. **Independent checks — read-only and parallel.** After the implementer stops,
   spawn reviewer and tester concurrently when capacity allows:

   - Reviewer: inspect the actual diff and surrounding code for correctness,
     regressions, maintainability, security, and missing tests. Do not edit.
     Report only actionable findings labeled `P0`, `P1`, or `P2`, with file and
     line references, plus an explicit `no blocking findings` result when clean.
   - Tester: run focused and broader relevant checks, exercise important edge
     cases, and distinguish product defects from environment failures. Do not
     edit. Report exact commands and outcomes.

5. **Fix.** Send consolidated, evidence-backed `P0` and `P1` findings and real
   test failures back to the same implementer. Include `P2` findings only when
   their value exceeds their scope or complexity cost. The implementer fixes
   them and reruns affected tests.

6. **Recheck.** Reuse the reviewer and tester for the changed areas. Allow at
   most two fix-and-recheck rounds by default. The user may request another
   limit. If blocking findings remain after the limit, stop and report them
   instead of silently looping or claiming completion.

7. **Accept.** The coordinator inspects the final diff, confirms that no agent
   is still writing, and runs any final inexpensive check needed to reconcile
   conflicting reports.

## Completion gate

Call the task complete only when:

- acceptance criteria are satisfied;
- no known `P0` or `P1` finding remains;
- focused tests pass;
- the broadest relevant verification passed, or its limitation is explicitly
  reported;
- the diff contains no unrelated changes introduced by the loop.

The final handoff must state the outcome first, then list changed files, tests
and their results, review status, assumptions, and any remaining risks. Do not
commit, push, deploy, publish, or mutate external systems unless the user
explicitly requested it.
