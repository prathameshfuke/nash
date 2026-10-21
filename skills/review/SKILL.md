---
name: review
description: Use when reviewing a diff, pull request, branch or files for bugs, when AI code reviews come back full of nitpicks, style comments, false alarms or vague "consider" suggestions, or when the user wants a thorough, multi-agent or adversarial code review. Triggers on "nash review", "bug hunt", "find bugs", "review this PR".
---

# Nash: Review

AI reviewers are paid in words, so they flood you with nitpicks and plausible-sounding false alarms. Nash Review is a **bug bounty**, a mechanism where the only way to score is a bug that is proven real. Noise stops paying, so it stops appearing.

**Core rule: no proof, no points.** A finding is a bug only when a failing test, repro command, or concrete input→wrong-output trace shows it.

Usage: `/nash:review` (current diff) · `/nash:review <PR | branch | path>` · `/nash:review 3 <target>` (number of hunters)

## 1. Scope and deal hunting grounds

Read the target diff, and for each changed function also read its callers. Then deal each hunter one **hunting ground** that this diff actually touches. Skip grounds that don't apply. The default is up to 5:

| Ground | Hunts for |
|---|---|
| Trust boundary | injection, authz gaps, secrets, unsafe deserialization, SSRF |
| State and concurrency | races, stale reads, double-submit, ordering, leaks |
| Edge inputs and errors | empty, null, huge, unicode or negative inputs; swallowed errors; partial failure |
| Data integrity | migrations, lost writes, wrong units or timezones, rounding, idempotency |
| Contracts and callers | a changed signature or behavior that breaks a caller, API, or test |
| Performance | N+1 queries, unbounded loops or memory, missing indexes, hot-path allocations |

**Focal sweep first.** Spend 2 minutes listing the bugs *you* spot immediately, the ones every hunter would find. Hand that list to every hunter as **CLAIMED, worth 0**. In testing, a "stay in your ground" rule alone didn't work: every hunter still reported the same 2 obvious bugs. Claiming them up front pushes hunters past the focal bugs and into their ground. The claimed bugs still go in the report, and you prove them yourself.

## 2. Announce the bounty, dispatch blind and in parallel

Dispatch all hunters in **one message**. Give each one the target, read and run access to the repo, its ground, and this text verbatim:

```
BOUNTY RULES
+severity (low 1, med 3, high 8, critical 20) per PROVEN bug in your ground.
Unproven claims score confidence × severity if later proven, and −2 × confidence × severity if disproven.
Style, naming, "consider refactoring", anything outside your ground, and anything on the CLAIMED list score 0. Don't submit them.
PROOF commands must be runnable as written from the repo root.
Zero findings is a valid, scoring-neutral answer. Don't invent bugs to fill space.

Per finding, return:
BUG: <one line>
WHERE: <file:line>
TRIGGER: <concrete input/state that breaks it>
PROOF: <failing test you wrote and ran, a repro command and its output, or "unproven">
CONFIDENCE: <0.0–1.0>   SEVERITY: <low|med|high|critical>
```

Hunters may write tests in a scratch location. They must not change product code.

## 3. Adjudicate

1. **Re-run every PROOF yourself.** A proof that doesn't reproduce makes the finding unproven.
2. **Challenge unproven findings at medium severity or above.** Dispatch one fresh verifier per finding with the instruction "try to disprove this, and if you can't, try to prove it". The verdict is PROVEN, DISPROVEN, or PLAUSIBLE.
3. **Dedupe by root cause,** not by line.
4. **Score** with the bounty rules, so the scoreboard shows which grounds found real bugs.

## 4. Report

```
PROVEN (n)    | Sev | file:line | Trigger | Proof (test name / command)
PLAUSIBLE (n) | Sev | file:line | Why it couldn't be settled
REJECTED: n false alarms disproven   (this count is the noise you didn't have to read)
SCOREBOARD: ground → points
```

Don't fix anything unless asked. Offer to turn each PROVEN finding's test into a regression test.

## Common mistakes

| Mistake | Fix |
|---|---|
| Hunters all review "everything" | One ground each. Overlap makes every hunter chase the same obvious bug. |
| Reporting a hunter's proof without re-running it | Always re-run it. Hunters sometimes claim tests failed that they never ran. |
| Padding the report with unproven medium and low findings | They go in PLAUSIBLE with a reason, or they're dropped. |
| Grading a "missing test" as a bug | Missing coverage isn't a defect. Use `/nash:test` for that. |
