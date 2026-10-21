---
name: test
description: Use when writing, generating or evaluating tests, when test coverage looks high but bugs still ship, when AI-written tests pass but may assert nothing meaningful, or when the user wants to know how good a test suite really is. Triggers on "nash test", "mutation testing", "are these tests any good", "harden tests".
---

# Nash: Test

Tests written by the same agent that wrote the code share its blind spots, and coverage only counts the lines that ran, not whether anything was checked. Nash Test is a **zero-sum war**. Red agents plant realistic bugs (mutants) and blue agents must write tests that kill them. The mutation score (killed / total) is the honest measure of a suite.

**Core rule: a test is only worth something if a plausible bug makes it fail.**

Usage: `/nash:test <file | module | function>` (3 red agents, 5 mutants each) · `/nash:test options <target>` (score only, no blue team)

## 0. Safety

Work in an isolated copy, either `git worktree add` or a temp copy of the repo. Never mutate the user's working tree. At the end, the original tree must be byte-identical (`git status` clean, apart from new tests the user accepts).

## 1. Baseline

Run the target's tests. They must pass. Record the runtime and the test command.

## 2. Red team: plant mutants (blind, in parallel)

Dispatch red agents in a single message, each with a different **mutant class**:

| Class | Example mutants |
|---|---|
| Boundaries | `<` ↔ `<=`, off-by-one, empty or single-element cases dropped |
| Error paths | a removed null check, an exception swallowed, a wrong fallback value |
| Logic | an inverted condition, a wrong operator, a business rule skipped, a wrong default |
| State | a missing update, a wrong order of operations, a stale value reused, cache not invalidated |

Each red agent returns 5 mutants. Each mutant is a **minimal unified diff** that a real developer could plausibly commit, plus **INPUT**: a concrete call whose output differs between the original and the mutant. If no INPUT exists, the mutant is equivalent and gets dropped. Red agents do not edit files.

## 3. Score the suite

Drop duplicate mutants first, meaning the same change proposed from different classes. In testing, 2 of 15 were duplicates.

For each mutant: apply it in the isolated copy, run the tests, then revert. If any test fails, the mutant is **killed**. If all pass, it **survived**. Mutation score = killed / non-equivalent.

## 4. Blue team: kill the survivors (in parallel)

Dispatch blue agents in a single message. Each gets one or more surviving mutants (diff + INPUT) and writes a test in the project's existing style. You then verify both conditions by running the test: it **passes on the original** and **fails on the mutant**. A test that fails on the original is a bug report, not a test, so surface it.

**Never lock in a bug.** If the docs, spec, or docstring disagree with what the original code does, blue must not assert the current behavior. It reports a **SPEC VIOLATION** with the input, then kills the mutant using a different input on which spec and code agree. In testing, a blue agent asserted `bulk_discount(10) == 0.0` while the docstring says "10 or more get 10% off", which made the bug permanent.

## 5. One rematch (optional)

The red team sees the new tests and plants up to 5 new mutants aimed at their gaps. Score again. Stop after this round.

## Report

```
MUTATION SCORE: before x/y (p%) → after x/y (p%)
SURVIVORS KILLED: mutant → test name
STILL SURVIVING: mutant → why (equivalent? needs an integration test?)
TESTS ADDED: file → count   (all pass on original, each kills ≥1 mutant)
SPEC VIOLATIONS: input → spec says / code does   (real bugs found along the way)
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Trivial mutants (syntax errors, deleting whole functions) | Mutants must be bugs a reviewer could miss. Trivial ones inflate the score. |
| Blue writes `assert result is not None` | The test must fail on the mutant. Run it against the mutant to prove it. |
| Mutating the user's working tree | Isolated copy only. Revert after each mutant and verify a clean tree at the end. |
| Counting equivalent mutants as survivors | No differentiating INPUT means equivalent: drop it. |
