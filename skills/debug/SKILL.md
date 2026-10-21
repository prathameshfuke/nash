---
name: debug
description: Use when a bug, test failure, crash, flaky test or wrong output has an unclear root cause, when a previous fix didn't work, when an agent keeps patching symptoms or locks onto its first guess, or when the user wants several hypotheses investigated in parallel. Triggers on "nash debug", "why is this failing", "root cause".
---

# Nash: Debug

An agent debugging alone falls in love with its first hypothesis and spends its effort confirming it. Nash Debug runs a **hypothesis market**. Competing agents each own a different theory and put a probability on it. You then run the experiment that best separates the theories, and the result pays out to whoever was right. Confidence is scored with a proper scoring rule (Brier), so overclaiming costs the agent that does it.

**Core rule: no fix until one hypothesis survives an experiment that could have killed it.**

Usage: `/nash:debug <symptom>` (4 hypotheses) · `/nash:debug 6 <symptom>`

## 1. Pin the symptom

Write down three things: the exact repro (a command), the expected result, and the actual result. If there is no repro yet, getting one is the first job. Logs, CI artifacts, and a diff between environments are fair game for getting it.

**Flaky symptom?** The repro is then a *rate*, for example "`for i in $(seq 50); do <cmd>; done` fails about 15/50". Every experiment below must predict a rate on N runs, not a single pass or fail. One run of a flaky test proves nothing. If the failure only happens in CI, experiments are CI re-runs or a temporary instrumentation commit, and you ask the user before pushing anything.

## 2. Name the focal hypothesis

Write your own first guess in one line: **Focal: …**. It enters the market as one hypothesis among several, not as the answer.

## 3. Open the market (blind, in parallel)

Dispatch one agent per **cause class** in a single message. The focal guess is one of them. Fill the remaining slots with the classes most plausible for this symptom:

| Class | Looks at |
|---|---|
| Recent change | `git log` and diffs since it last worked; candidates for bisect |
| Data and state | inputs, fixtures, DB rows, caches, global or shared state |
| Environment | config, env vars, dependency versions, OS, clock, timezone |
| Timing and concurrency | ordering, races, async, retries, timeouts, flakiness |
| Wrong expectation | the test, spec, or assertion is itself wrong |
| Boundary | serialization, API contracts, encoding, units between components |

Agents **read and run diagnostics only. They make no fixes.** Each agent returns:

```
HYPOTHESIS: <one line>
MECHANISM: <cause → … → observed symptom, step by step>
P: <0.0–1.0 that this is THE root cause>
EXPERIMENT: <one cheap command or check>
IF TRUE: <predicted result>   IF FALSE: <predicted result>
```

## 4. Trade until it clears

1. Add an **OTHER** hypothesis at P = 0.15, then normalize all P values so they sum to 1. That is the market. OTHER is there so the market can't hit 0.9 on a wrong set of theories. Compound causes, such as shared state *plus* ordering, are allowed as one hypothesis. A **round** is one experiment.
2. Choose the next experiment. It must be one where hypotheses holding most of the probability predict **different** results, ideally splitting the probability mass near 50/50. Among those, pick the cheapest. An experiment that every hypothesis predicts the same result for tells you nothing, so skip it.
3. Run it yourself. Kill every hypothesis whose prediction was clearly wrong. For flaky rates, down-weight hypotheses whose prediction was off instead of killing them, unless they predicted 0 failures and you saw some. Renormalize.
4. Repeat until a single hypothesis reaches P ≥ 0.9 (with OTHER below 0.1), and its own predicted result came true.

After 3 rounds with no winner, stop and report the market to the user: the surviving hypotheses, what killed the others, and the next experiment. Don't keep guessing.

## 5. Fix and settle

Write a test that fails for the confirmed reason, fix the root cause (not the symptom), and watch the test pass. For flaky bugs, the test must **force** the condition (fixed seed, injected ordering or clock) so it fails every time before the fix. Verify the fix with N consecutive passes of the original symptom, for example 30. Then settle the market with a Brier score for each agent, (P − outcome)², where outcome is 1 for the confirmed hypothesis and 0 for the rest, and lower is better. This shows which class of thinking was calibrated on this bug.

## Report

```
SYMPTOM / REPRO
ROOT CAUSE: <hypothesis>, confirmed by <experiment → result>
KILLED: <hypothesis → the experiment that killed it>
FIX: <file:line>   REGRESSION TEST: <name>
CALIBRATION: agent → P → Brier
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Running the experiment that confirms the favorite | Pick the one where theories disagree. Confirming evidence that every theory predicts is worth nothing. |
| Agents "just trying a fix" | Diagnostics only until the market clears. A fix that happens to work hides the real cause. |
| All agents looking at the code diff | One cause class each. The bug is often in data, environment, or the test. |
| Declaring a cause because nothing contradicts it | It must have survived an experiment that could have killed it. |
