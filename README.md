# Multi-Agent Verification

A pattern for running large work through a team of agents instead of one, structured so that hallucinated, cheated, or buggy results do not survive to the final output, even when no human reads any single intermediate result.

The insight is not a smarter model. It is an org chart plus a checking loop.

## The problem it addresses

Give one model a large multi-step job and you get a plausible-looking artifact with no way to know which parts are real. The failure is not that models are bad at the work. It is that the same agent that produced a result is the one telling you it is correct, and a self-report is not evidence.

Hallucination is not solved here. It is positioned out of the picture structurally: nothing a worker claims is trusted until a separate agent re-derives it from source.

## How it works

Three roles, and the separation is the whole point. The agent that does the work is never the agent that certifies it.

**Boss** (one, most capable model). Does only judgment: research, the written standard, specs, review, dispute rulings. Never produces the base work. If the boss is writing code or copy, routing has failed.

**Workers** (many, cheapest capable model). Produce the artifacts. Assume they cut corners; that is priced in. The system is not built on trusting workers, it is built so their cut corners do not survive.

**Checkers** (one per task, always independent). Certify by re-executing verification. A checker never sees the worker's self-report. The worker can say done; the checker decides whether that is true by re-deriving ground truth.

Around those roles sit six phases: setup, a written constitution of testable clauses, dependency-sliced task decomposition, auditions for unfamiliar model families, the execution loop with a retry cap and a dispute path, and a final sweep in which the boss's own output gets independently checked like anything else. All run state lives in a `swarm/` directory with a ledger as the single source of truth, so a run survives context compaction or a resume days later.

## The four failures the checks are designed to catch

1. **Worker hallucination.** Fabricated or paraphrased content that was supposed to be retrieved verbatim, close enough to fool a skim. Caught by re-deriving ground truth character for character.
2. **Worker cheating.** Output that passes the letter of a check while defeating its intent: hidden text satisfying a content rule, an empty element satisfying a layout rule. Caught by testing in the real environment against the real intent, never against a proxy the worker can trick.
3. **Boss bug.** A defect in the orchestrator's own work. Caught by an independent check on boss output, in addition to the boss's own review pass.
4. **Checker error.** Good work failed under a misapplied rule. Caught by a dispute path where the checker itself is falsifiable and correctable.

If a check design would not catch all four, it has a hole.

## What is in here

```
skills/agent-swarm/
  SKILL.md                 the runbook: roles, routing, six phases
  references/templates.md  ledger schema and verbatim spawn prompts
```

The spawn prompts are meant to be copied verbatim with only the bracketed slots filled. Improvised spawn prompts are the largest source of run-to-run variance.

It is packaged as an agent skill with YAML frontmatter, so it loads directly in harnesses that support that format. The method itself is harness-agnostic; nothing in it depends on a specific vendor. There is also a degraded mode for when no subagents are available at all, which preserves the one property that matters: a checking pass starts from the source, not from the working pass.

## Notes from actual runs

The templates carry a few findings that cost real time to discover:

- `codex exec` can exit 0 having done nothing, printing a capacity error and changing no files. Grep worker output for errors and confirm deliverable files actually changed before spawning a checker. A capacity error is not a worker attempt.
- `codex exec` hangs indefinitely if stdin is left open. Close it with `</dev/null`.
- The Antigravity CLI (`agy`) failed its audition as a worker: `-p` print mode is text-only and cannot write files even with permission flags relaxed (tested v0.50-era, July 2026). Re-audition before admitting it.

## The one-line version

Name what done right means once, in a written standard. Route judgment to the expensive agent and execution to cheap ones. Attach an independent, re-executing check to every task, return specific located defects until it passes, cap retries and escalate, and let no agent, not even the boss or the checker, skip verification. Then the system, not any single model, is what catches the mistakes.

## License

MIT. See [LICENSE](LICENSE).
