---
name: loop
description: Use when iterating on code, a feature or an artifact until it meets a quality bar, when running builder/critic, generator/evaluator or self-refine loops, when an agent loop stops too early, never stops, thrashes, or its critic starts rubber-stamping, or when an agent may game its own checks. Triggers on "nash loop", "iterate until", "keep improving", "build-review loop".
---

# Nash: Loop

A builder–critic loop is a **repeated game**, and repeated games invite collusion. After a few rounds the critic, having seen the builder's reasoning, softens and starts approving. The builder learns to satisfy the critic rather than the goal, and may even edit the checks themselves. Nash Loop uses mechanism design to keep both players honest, plus an **optimal stopping** rule so the loop ends at the right time.

**Core rule: the checks are fixed before round 1, and the critic is never the same one twice.**

Usage: `/nash:loop <goal>` (max 5 rounds) · `/nash:loop 8 <goal>`

## 1. Write the contract (before any building)

- **Hard checks:** executable commands with **numeric** pass conditions. Turn vague goals into numbers yourself. "Fast" becomes something like "median of 5 runs under X µs/call"; "well-tested" becomes something like "mutation score at least 80% (`/nash:test`)". The user's yes on the contract covers these thresholds.
- **Acceptance tests (frozen):** tests you write from the goal before round 1. **The builder may not edit them.** Before each critique, check that they're unchanged, using `git diff --stat -- <files>` or file hashes if there's no git. Any change means a tampered round: revert it and tell the builder. If the builder should also write tests, it adds them in a **separate, unfrozen file**, and the hard checks judge them by mutation score.
- **Rubric:** 3–5 criteria scored 0–10, drawn from what the user actually wants.

Show the contract to the user for a yes before round 1, unless they said to run unattended.

## 2. Each round

1. **Build.** The builder gets the goal, the contract, the output of any failing hard checks, and **only the top 3 blocking issues** from the last critique.
2. **Check and snapshot.** Run the hard checks yourself. Snapshot the round with a commit or a copy, so any round can be restored.
3. **Critique with a fresh critic.** Dispatch a new critic agent each round. It gets the contract, the current code and tests, and the hard-check results. It gets no history and none of the builder's explanations: a critic that can't hear the builder's excuses can't be talked into agreeing. It returns a rubric score per criterion plus its blocking issues, ranked. Map each issue to a short canonical ID, such as `unicode-nfkd`, so you can tell when the same issue survives across differently worded critiques.
4. **Audit (inspection game).** Before round 1, pick the audit round at random (1–3) and keep it to yourself. In that round, **you** make a copy of the artifact with one planted defect. The defect must pass the hard checks (otherwise the critic gets a free signal) but break a rubric criterion, for example dropping unicode normalization for one script. It must leave no trace: no comment, no odd diff. Dispatch **two** fresh critics, one on the real artifact and one on the planted copy. If the planted-copy critic doesn't list a blocker naming that behavior, the real critique for this round is void. Re-run it with a stricter prompt or a different model. The re-run doesn't use up a round. Because critics can't know which round is audited, rubber-stamping never pays.
5. **Keep the best version so far:** the most hard checks passing, then the highest rubric total.

## 3. Stopping rule

Stop at the first of these:

| Stop when | Why |
|---|---|
| All hard checks pass **and** the critic has no blocking issues | Done |
| All hard checks pass, and the rubric total rose less than 5% from round n−2 to round n (needs 3 or more rounds) | Diminishing returns: further rounds cost more than they add |
| The same canonical issue ID is blocking in 2 consecutive rounds | Thrashing: report it and ask, don't spin |
| The round cap is reached | Budget |

On stopping, restore the **best** snapshot if it isn't the current one.

## Report

```
| Round | Hard checks | Rubric total | Top blocking issue | Audit |
STOPPED: <rule that fired>
BEST: round n, with any remaining issues
TAMPERING: none | <round → file reverted>
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Reusing one critic across rounds | Fresh critic every round. Memory leads to collusion. |
| Sending the critic the builder's explanation | Send the artifact only. Persuasion isn't evidence. |
| Letting the builder "fix" a failing test | Tests are frozen. A failing test means the work is wrong, not the test. If the test really is wrong, stop and ask the user. |
| Shipping the last round | Ship the best round. Late rounds can regress. |
| Looping without a stopping rule | Use the table. "Until it's good" isn't a rule. |
