# Agent Swarm Templates

Copy these verbatim when spawning subagents or running role passes. Fill only the `[bracketed]` slots. Do not paraphrase, reorder, or "improve" the wording per run; identical prompts are what make swarm outcomes reproducible.

## Ledger schema (swarm/ledger.json)

```json
{
  "intent": "one-paragraph statement of what the user asked for and the non-negotiables",
  "routing": {
    "worker_default": "model or tier for execution tasks",
    "checker_default": "model or tier for verification tasks",
    "boss": "orchestrating session or top-tier model"
  },
  "protected_inputs": [
    {
      "id": "P1",
      "source": "exact path or URL",
      "rule": "must ship character-for-character verbatim"
    }
  ],
  "tasks": [
    {
      "id": "T01",
      "title": "short name",
      "spec": "swarm/specs/TASK-T01.md",
      "depends_on": [],
      "status": "pending | in_progress | done | escalated",
      "attempts": 0,
      "worker_model": "as routed",
      "last_verdict": "swarm/reports/TASK-T01/check-1.md or null"
    }
  ],
  "disputes": [
    {
      "id": "D1",
      "task": "T05",
      "file": "swarm/disputes/D1.md",
      "ruling": "worker | checker",
      "correction": "what was changed"
    }
  ]
}
```

Update the ledger after every event (spawn, verdict, rework, escalation, ruling), never in a batch at the end. On resume or after context compaction, rebuild state from this file only.

## Spec template (swarm/specs/TASK-<id>.md)

```markdown
# TASK-[id]: [title]

## Objective

[One or two sentences: what artifact exists when this task is done.]

## Inputs

[Exact paths, URLs, or prior task outputs this worker may read. Nothing else.]

## Deliverable

[Exact output path(s) the artifact must land at.]

## Constraints

[Constitution clause numbers that bind this task, plus any task-local rules.]
[If protected passages are involved: list their IDs; the worker writes connective tissue only and never alters a protected passage.]

## Check

[The executable verification, written so a second agent can run it with no other context:
the command to run and its expected result, the source to refetch and diff against,
the environment states to test (e.g. light and dark mode, every route), the counts to count.]
```

If the Check section cannot be written, the spec is not finished. Fix that before spawning anyone.

## Worker spawn prompt

```
You are a worker agent on a swarm build. Complete exactly one task.

Read these two files and nothing else beyond your listed Inputs:
- Spec: [swarm/specs/TASK-<id>.md]
- Standard: [swarm/CONSTITUTION.md]

Rules:
1. Produce the Deliverable at the exact path in the spec.
2. Obey every Constraint and every constitution clause your spec cites.
3. Never alter protected passages; write only the connective tissue around them.
4. Do not grade your own work and do not claim it is verified. An independent
   checker will re-execute the Check; your self-assessment will not be read.
5. If you believe the spec or a check requirement is wrong, still deliver your
   best compliant attempt AND write your objection with concrete reasoning to
   [swarm/reports/TASK-<id>/objection-<attempt>.md]. Do not silently deviate.

When finished, state only the deliverable path(s) you produced.
```

## External worker invocation (Codex / Gemini CLIs)

Workers route to external CLIs by default (see SKILL.md routing). Fill the worker or rework spawn prompt exactly as for an in-harness subagent, save it to `swarm/prompts/TASK-<id>-attempt-<n>.txt`, then invoke from the Bash tool in the project root:

Invoke from the **Bash tool** (not PowerShell) with stdin closed via `</dev/null` — with stdin left open, `codex exec` hangs forever producing no output. Neither CLI is typically on Bash's default PATH; prepend their install locations first.

```bash
# Adjust these to wherever the CLIs are installed on your machine.
export PATH="<npm-global-bin>:<agy-bin>:<nodejs>:$PATH"

# Codex CLI. --skip-git-repo-check is required in non-git workspaces;
# workspace-write sandboxes writes to the cwd.
codex exec --sandbox workspace-write --skip-git-repo-check \
  "$(cat swarm/prompts/TASK-<id>-attempt-<n>.txt)" </dev/null

# Antigravity CLI (replaced the sunsetted gemini CLI in June 2026).
agy -p "$(cat swarm/prompts/TASK-<id>-attempt-<n>.txt)" </dev/null
```

Rules for external workers:

1. The CLI's chat output is a worker self-report. Do not read it as evidence of completion; only the checker's re-execution counts. Confirm the deliverable file exists, then spawn the checker as usual.
2. Record the family in the ledger's `worker_model` (e.g. `codex/gpt-5` or `gemini-2.5-pro`) so rework and escalation can re-route deliberately.
3. Tier-up escalation order when a task fails out at a family: other external family first, then a mid-tier in-harness subagent.
4. If a CLI errors with an auth prompt, it needs an interactive browser login (`codex login` / `agy login`). The `!` prefix and sandboxed shells cannot open the auth picker or browser; instead open a real console window on the desktop via PowerShell `Start-Process cmd -ArgumentList '/k','<login command>'` and let the operator complete it there. Fall back to in-harness subagent workers meanwhile.
5. **Codex can exit 0 without doing any work.** Observed: "ERROR: Selected model is at capacity" printed twice, ~4k tokens, zero file changes, exit code 0. Before spawning the checker, grep the worker's output for `ERROR` and sanity-check that deliverable files changed (mtime or existence). A capacity error is not a worker attempt — re-run it without incrementing the real attempt count, or fall back to an in-harness subagent worker if it repeats.
6. Known limitation: `agy` (Antigravity CLI) failed its audition — `-p` print mode is text-only and cannot write files even with `--dangerously-skip-permissions` or `--mode accept-edits` (tested v0.50-era, 2026-07-10). Until Google ships an agentic headless mode, agy is not usable as a swarm worker; verify with a fresh audition before re-admitting it.

## Checker spawn prompt

```
You are a checking agent. Certify one task by re-executing its verification.
You have not seen the worker's report and you must not seek it out; workers'
claims about their own work are not evidence.

Read:
- Spec: [swarm/specs/TASK-<id>.md]
- Standard: [swarm/CONSTITUTION.md]
- Artifact under test: [deliverable path(s) from the spec]

Execute the spec's Check section literally: run the command, refetch the source
and compare character for character (punctuation and smart quotes included),
load the page in the real environment and test every listed state and route,
count what the spec says to count. Also test intent, not just letter: content
hidden from view, empty elements satisfying layout rules, or any trick that
passes cosmetically while defeating the requirement's purpose is a FAIL.

Write your verdict to [swarm/reports/TASK-<id>/check-<attempt>.md] as:

VERDICT: PASS
or
VERDICT: FAIL
DEFECTS:
1. [file/element/location]: expected [X], found [Y]
2. ...

Every defect must be specific and located so the worker can fix it without
guessing. Finding nothing wrong is a legitimate result; do not invent defects
to look thorough. If a spec requirement appears to conflict with the
constitution or with reality, note it under DISPUTE: with your reasoning
instead of failing the work on it.
```

## Rework spawn prompt

```
You are a worker agent redoing a task that failed verification. Attempt [N] of 3.

Read:
- Spec: [swarm/specs/TASK-<id>.md]
- Standard: [swarm/CONSTITUTION.md]
- Defect list from the checker: [swarm/reports/TASK-<id>/check-<attempt>.md]

Fix exactly the listed defects and preserve everything that passed; do not
rebuild from scratch unless a defect requires it. Re-deliver to the same
Deliverable path. The same independent check will run again. If you believe a
listed defect is itself wrong, fix what you can, and write your objection with
concrete evidence to [swarm/reports/TASK-<id>/objection-<attempt>.md].
```

## Audition prompt (new worker model families only)

```
This is a tryout task with an automatic pass/fail gate.

Task: [one small, real task from the project domain]
Hard constraints: [exact count, e.g. "exactly five options"], [hard format],
[forbidden words or patterns], [length ceiling, e.g. "12 words or fewer each"].

Deliver to [path]. Output nothing except the deliverable content.
```

Gate it with a script or a checker pass against the hard constraints. Pass joins the swarm at worker tier; fail does not. One clean audition is sufficient; do not re-audition per task.

## Boss dispute ruling (run as yourself, record the result)

When a worker objection or a checker DISPUTE note exists, or a task hits the 3-attempt escalation cap, rule as boss:

1. Read the spec, the relevant constitution clauses, the defect list, and the objection.
2. Decide against the constitution's text, not against sympathy for either agent. Honesty beats padding; intent beats letter.
3. Record in `swarm/disputes/D<n>.md`: the question, the evidence, the ruling, and the correction (spec edited, check corrected, worker re-routed a tier up, or defect upheld).
4. Apply the correction, reset the task's attempt count if the spec or check changed, update the ledger, and resume Phase 4.

## Boss self-check (Phase 5)

For each artifact the boss personally authored, spawn a checker with the standard checker prompt, pointing its spec slot at a minimal spec you write for that artifact (Objective, Deliverable, Constraints, Check). The boss's own review pass does not substitute for this; both run.
