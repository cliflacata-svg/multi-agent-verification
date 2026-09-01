---
name: agent-swarm
description: Use this skill whenever a task is too big or too high-stakes for a single agent pass and would be better run by a team of agents. Triggers include any request to "orchestrate agents", run a "multi-agent" or "swarm" or "team of agents" job, set up a "boss / workers / checkers" pattern, delegate a large multi-step build to agents, control cost by routing cheap models for execution and an expensive model for judgment, or produce work that must be verified end to end without a human reading every intermediate result. Also use PROACTIVELY when you notice a task has many dependent steps whose outputs must be trusted, when the worry is hallucinated or cheated results surviving to the final output, or when someone says "build me the whole X" and the honest answer is that one pass will not hold up. This is the recipe for staffing, checking, and shipping big work through an org chart of agents.
---

# Agent Swarm

A recipe for running big work through a team of agents instead of one. The insight is not a smarter model. It is an org chart plus a checking loop, designed so that hallucinated, cheated, or buggy results do not survive to the final output, even when no human reads any single intermediate result.

Hallucination is not solved here. It is positioned out of the picture structurally, because nothing a worker claims is trusted until a separate agent re-derives it.

This skill stacks with any per-agent discipline skill you already use: that discipline is how each individual agent works; this skill is how you wire agents together so the whole system self-checks.

**This is a runbook, not background reading.** When this skill triggers, execute the phases below in order, producing the exact artifacts named. The fixed artifacts and verbatim prompt templates are what make outcomes consistent across runs. Do not improvise substitutes for them.

## When to reach for it

Swarm when the task is large, multi-step, and the cost of a wrong-but-plausible result is high: a full build, a research deliverable with many load-bearing claims, a migration, anything where "looks done" and "is done" can diverge and a human will not audit every piece.

Do not swarm a task a single disciplined pass handles well. The overhead is only worth it when trust between steps is the actual problem.

## The three roles

The separation is the point: the agent that does the work is never the agent that certifies it.

- **Boss** (one, the most capable model available; in most harnesses that is you, the orchestrating session). Does only judgment: research, the constitution, specs, review, dispute rulings. Never produces the base work. If the boss is writing pages, code, or copy, routing has failed.
- **Workers** (many, the cheapest capable model per task). Produce the artifacts. Assume they cut corners; that is priced in. The system is not built on trusting workers, it is built so their cut corners do not survive. Prefer more than one worker model family where practical, because different families fail differently.
- **Checkers** (one per task, always independent of the worker). Certify work by re-executing verification. A checker never sees the worker's self-report. The worker can say done; the checker decides whether that is true by re-deriving ground truth.

Routing principle: route every job to the cheapest agent that can do it well. Set the subagent model per task: cheapest tier for workers, a mid or top tier for checkers when the check needs judgment, top tier (or the orchestrator itself) for boss duties. Route by capability tier, not by hard-coded model names, since names and prices change.

### External worker CLIs (token-pool longevity)

Default routing: **workers go to external CLIs, not orchestrator subagents**, because subagents spawned inside the orchestrating harness draw from the same subscription token pool as the boss and checkers. Separately paid external CLI subscriptions are independent capacity, so spending them on execution preserves the orchestrator's pool for judgment.

- **Workers**: `codex exec` and `agy` (Antigravity CLI), invoked via Bash. Both are agentic CLIs that read and write files in the working directory, so they take the standard worker/rework spawn prompt verbatim. Invocation commands are in `references/templates.md`. Using both families is a feature: different families fail differently. (The old `gemini` CLI is dead for AI Pro subscribers as of June 2026; Antigravity replaced it.)
- **Checkers**: always subagents inside the orchestrating harness, via its Agent tool (haiku for mechanical re-execution, sonnet when the check needs judgment). Never route checking to the external CLIs; the trust structure depends on checker quality, and quality assurance is what the orchestrator's pool is being preserved for.
- **Boss**: the orchestrating session. Unchanged.

Both external families still owe a Phase 3 audition the first time they join a swarm on a given kind of work. If an external CLI is unauthenticated or unavailable mid-run, fall back to in-harness subagent workers for the remaining tasks and note the routing change in the ledger; do not stall the run.

## Fixed run layout

All swarm state lives in a `swarm/` directory inside the project. Create it in Phase 0 and treat `swarm/ledger.json` as the single source of truth for run status. Every phase reads it and writes it. If your context is ever compacted or the run resumes later, rebuild your understanding from the ledger, not from memory.

```
swarm/
  CONSTITUTION.md      # the written standard, produced in Phase 1
  ledger.json          # task registry + status, updated after every event
  specs/TASK-<id>.md   # one spec per task
  reports/TASK-<id>/   # checker verdicts per attempt: check-1.md, check-2.md, ...
  disputes/            # escalation records, one file per dispute
```

Ledger schema and all spawn prompts are in `references/templates.md`. Read that file before Phase 3 and copy its prompts verbatim, filling only the bracketed slots. Improvised spawn prompts are the largest source of run-to-run variance; do not write your own.

## Phase 0: Setup

1. Create the `swarm/` directory and an empty `ledger.json` per the schema in `references/templates.md`.
2. Record in the ledger: the user's intent (one paragraph), the protected inputs if any (exact source locations of content that must ship verbatim: an author's words, a legal clause, a brand rule), and the model routing table (which tier handles worker, checker, boss duties).

## Phase 1: Research and constitution

You do not prompt big work task by task. You name what done right means once, at the top, and the system enforces it on every round.

1. Research the domain of the deliverable (standards, user needs, constraints). The user's prompt can be a few words of intent; the research is your job.
2. Write `swarm/CONSTITUTION.md`: a numbered list of testable clauses, each stating a requirement AND how it is checked (the command, the tool, the environment, the states to test, for example both light and dark mode, every route). Ten to twenty clauses is typical. A clause that cannot be re-executed by a second agent is not a clause; rewrite it until it is checkable.
3. If protected inputs exist, add a clause requiring character-for-character verification of every protected passage on every build, punctuation and smart quotes included. Workers write only connective tissue between protected passages, never the passages.

## Phase 2: Decompose into tasks

1. Slice the work by dependency, not category: each task's output feeds the next.
2. For each task, write `swarm/specs/TASK-<id>.md` with exactly these sections: Objective, Inputs (paths and sources), Deliverable (exact path the artifact must land at), Constraints (relevant constitution clause numbers), Check (the executable verification for this task: the command to run, the source to refetch and compare, the browser test to perform). If you cannot write the Check section, the spec is not done.
3. Register every task in the ledger with status `pending` and attempt count 0.

## Phase 3: Auditions (only for unfamiliar worker models)

Before a model family you have not used in this pattern joins the swarm, give it one small real task with a hard, automatically checkable gate (exact count, forbidden words, format, time budget). Use the audition template in `references/templates.md`. Pass joins, fail does not. Skip this phase entirely for families you already trust.

## Phase 4: The execution loop

For each task in dependency order (parallelize independent tasks in the same turn where the harness allows):

1. **Spawn the worker** with the worker template: it receives only its spec file and the constitution. Set status `in_progress`.
2. **Spawn the checker** with the checker template: it receives the spec, the constitution, and the deliverable path. It NEVER receives the worker's summary, chat output, or claimed status. It must re-execute the Check section from the spec (compile the build, refetch the source and diff character by character, load the page in the real environment) and write its verdict to `swarm/reports/TASK-<id>/check-<attempt>.md` as PASS, or FAIL with a numbered list of specific located defects (file, line or element, expected vs actual).
3. **On PASS**: set status `done` in the ledger, move on.
4. **On FAIL**: increment the attempt count and re-spawn the worker with the rework template, which includes the checker's exact defect list. Specific feedback is what makes retries converge; "13 quotes mismatch, here are the 13 and the exact diff of each" converges, "some quotes are wrong, redo it" does not.
5. **Retry cap**: after 3 failed attempts on the same task, stop looping. Set status `escalated` and take the boss role: read the defect history yourself and rule. Either the spec is wrong (fix the spec, reset attempts), the check is wrong (see disputes below), or the worker tier is too weak (re-route the task one tier up, reset attempts).
6. **Disputes**: after every checker verdict, glance for an `objection-<attempt>.md` in the task's reports directory (workers file these when they believe a spec or check is wrong) and for a DISPUTE note in the checker's verdict. If either exists, or you observe a checker enforcing a rule the constitution does not actually require, do not silently side with either party. Write the case to `swarm/disputes/`, rule as boss against the constitution's text, and correct whichever side is wrong. A checker can be overruled; record the correction so the next check applies the fixed rule. Failures get investigated in both directions.
7. Update the ledger after every one of these events, not at the end.

## Phase 5: Boss review, and checking the boss

No rank is above verification, including yours.

1. Run your own review pass over the assembled whole against every constitution clause.
2. Anything you personally authored (the constitution's embedded content, glue you wrote, config you edited) gets an independent checker spawned on it, same as any worker task. The most capable model still writes bugs; the design catches the boss's own defects twice, once by your review and once by a checker that is not you.
3. Any defect found here becomes a new ledger task and goes through Phase 4 like everything else.

## Phase 6: Ship and report

The run is done only when every ledger task is `done` and the final Phase 5 sweep found nothing new. Report to the user: what shipped and where, the constitution it was held to, counts (tasks total, tasks failed and reworked, disputes and their rulings), and anything assumed rather than verified, stated plainly. The reworked-task count is not embarrassing; it is the evidence the checking layer worked.

## Degraded mode: no subagents available

The same phases run in a single context by playing each role in a separate, clearly delimited pass. The property that must survive is independence: a checker pass starts from the source, not from the worker pass. Re-open the file, re-fetch the URL, re-run the code, re-count the items. Never certify from memory of what you just wrote; that memory is the worker's self-report, and the checker does not trust it. All artifacts (`swarm/`, ledger, specs, verdicts) are still produced; they are what keeps the roles honest when one context plays all of them.

## The four failures the checks must catch

Design every task's Check section so all four are covered. If your check design would not catch all four, it has a hole.

1. **Worker hallucination**: fabricated or paraphrased content that was supposed to be retrieved verbatim, close enough to fool a skim. Caught by re-deriving ground truth from the source, character for character.
2. **Worker cheating**: output that passes the letter of the check while defeating its intent (hidden text to satisfy a content rule, an empty element to satisfy a layout rule). Caught by testing in the real environment against the real intent (a screen reader, not just visual rendering), never against a proxy the worker can trick.
3. **Boss bug**: a defect in the boss's own work. Caught by Phase 5's independent check on boss output plus the boss's separate review pass.
4. **Checker error**: good work failed under a misapplied rule. Caught by the Phase 4 dispute path, where the checker itself is falsifiable and correctable.

## The one-line version

Name what done right means once, in a written standard. Route judgment to the expensive agent and execution to cheap ones. Attach an independent, re-executing check to every task, return specific located defects until it passes, cap retries and escalate, and let no agent, not even the boss or the checker, skip verification. Then the system, not any single model, is what catches the mistakes.
