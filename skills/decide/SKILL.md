---
name: decide
description: Use when running several parallel agents or subagents for different opinions, perspectives, approaches, designs or brainstorms on one problem, when parallel agents keep returning the same answer, or when the user wants options compared and a winner picked. Triggers on "nash", "multiple perspectives", "council", "get N opinions", "fan out agents", "compare approaches", "which approach is best".
---

# Nash: Decide

Same-model agents given the same prompt converge on the same answer: the **focal point** (Schelling). Ten agents give you one opinion ten times. Nash turns the fan-out into a game where crowding the focal point does not pay, then picks a winner with a tournament that LLM judges can't easily bias.

**Core rule: diversity is assigned, never hoped for.** Personas alone don't break convergence, because agents with different personas still reason the same way. Banned levers do.

Usage: `/nash:decide <problem>` (6 agents) · `/nash:decide 4 <problem>` · `/nash:decide options <problem>` (skip the tournament)

## 1. Name the focal point (you, before any dispatch)

You are the same model as your agents, so your first instinct is the herd's answer. Write it down in one line: **Focal answer: …**. Then list its 2-4 core levers, meaning the mechanisms it relies on (e.g. "cache", "parallelize calls").

## 2. Deal the niches

Build N niches. Each niche is a **mechanism-level constraint**, not a persona:

| Slot | Niche |
|---|---|
| 1 | **Anchor:** the focal answer, done as well as possible (keeps the obvious answer in the pool, as one entry, not five) |
| 2 | **Inversion:** remove or invert the problem's premise (e.g. "don't load it at all") |
| 3..N | Each one **bans** one or more focal levers and names a different domain to draw from (data model, product/UX, process, infrastructure, buying it instead of building it, deleting it) |

**Bans apply to mechanisms, not words.** A renamed banned lever is still banned: a "read model" refreshed from events is still a precomputed snapshot. Write this into each banned niche's prompt.

**Pre-dispatch collision test.** For each niche, write the one-line answer you would predict it gives. If two predictions share a mechanism, re-deal one before dispatching. Niches whose domains sound different still collide: product/UX and inversion both drift to "lazy-load the widgets". This step costs nothing and prevents most collisions. If you can't make N distinct niches, use fewer agents and say so.

## 3. Dispatch blind, in parallel

Dispatch all agents in **one message**, so they run in parallel and none sees another's answer. Each agent's prompt holds: the problem, the user's context, its niche and banned levers, and this contract:

```
Return exactly:
PITCH: <one line, the core mechanism>
APPROACH: <≤150 words>
AVOIDED: <the obvious answer you were steered away from, and why yours can still win>
COST: <effort / risk / what it breaks>
CHECK: <one claim that could be verified by running code, reading files, or doing math>
Stay inside your niche even if you think the obvious answer is better. A tournament
judges on merit. Arguing your niche well is how you win.
```

Give agents tools (read files, run code) when the problem is in a codebase.

## 4. Collision check

Compare the PITCH lines **by mechanism, not wording**. Ask: would both have the same diff, the same cost, the same failure mode? When two share a core mechanism, keep the stronger one and re-dispatch the other into an unused niche. Do **one** re-roll round only. Report any collisions that remain.

## 5. Tournament (skip in `options` mode)

1. **Anonymize.** Relabel the entries A, B, C…, strip the niche names, and trim each to the contract.
2. **Score.** Dispatch 3 judges in parallel. Each gets every entry, in a **different shuffled order**, plus a rubric built from what the user actually cares about. Each judge ranks every entry and names its fatal flaw. Judges may run the CHECK claims.
3. **Rank.** Combine with a Borda count (the score is the sum of rank positions).
4. **Final.** If all 3 judges picked the same #1, skip this step: shuffled unanimity already controls for order. Otherwise the top 2 meet head-to-head twice, with the order swapped. A result counts only if both orders agree. If they disagree, it's a **split decision**: report both entries and let the user choose.

## 6. Report

```
FOCAL ANSWER: <line>   (did it win? say so plainly)
WINNER: <entry>, and why it beat the runner-up in one line
| Rank | Entry | Pitch | Fatal flaw | Borda |
STEAL LIST: the best idea from each losing entry that could be grafted onto the winner
CONFIDENCE: high (judges agree and the final held in both orders) / split / low
```

If the Anchor wins outright, that's a real result: the obvious answer survived real competition. Say so.

## Common mistakes

| Mistake | Fix |
|---|---|
| Niches are personas ("act as an SRE") | Ban levers. A persona with the same levers reaches the same answer. |
| Dispatching agents one after another | Send one message with all agents. Sequential agents anchor on earlier output. |
| Showing agents each other's answers in round 1 | Never. Visible peers cause conformity and sycophantic collapse. |
| One judge, fixed order | 3 judges with shuffled orders, and a final with swapped order. LLM judges favor the first answer, the longest answer, and answers from their own model. |
| Treating unanimous agreement as proof | Agreement among same-model agents is correlated, so it's weak evidence. Verify the CHECK claims. |
| Adding debate rounds | Most of debate's gain comes from voting. Skip the rounds. |
