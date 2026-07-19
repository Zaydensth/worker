# Model fleet — hand-off rules

Four tiered sub-agents live in ~/.claude/agents/. Route every task to the cheapest
model that can do it correctly; reserve the top tier for real strategy. Price per
1M tokens (in / out): Fable 5 $10/$50 · Opus 4.8 $5/$25 · Sonnet 5 $3/$15 · Haiku 4.5 $1/$5.

## Default pattern (B): a mid-tier model leads and escalates up once
Main loop = Opus 4.8 (`/model opus`) for execution-heavy work — it carries the
sustained work and pulls in the strategist (Fable) only for hard calls.
(Pattern A: set `/model fable` when the task is planning-heavy — Fable plans and
delegates execution down to the workers.)

## ESCALATE UP → strategist (Fable 5), ~once per task, only for:
- a genuinely hard design / strategy decision or a novel approach
- diagnosing a failure whose cause is not obvious (needs deep reasoning)
- a final go / no-go before an irreversible action (a push, a publish, a submit)
→ take its plan or decision, then execute it yourself.

## KEEP — do it yourself (the Opus main loop / executor):
implementing a plan, multi-step execution and loops, verifying worker output,
routine decisions and diagnosis, and deciding what to delegate.

## DELEGATE DOWN → worker-sonnet (Sonnet 5):
code from a clear spec, moderate analysis, reading a subsystem in parallel,
drafting scripts, verification runs, drafting summaries and docs.

## DELEGATE DOWN → worker-haiku (Haiku 4.5):
grep / glob / search, log scraping, extracting fields or numbers from output,
format and existence checks, status polling — high-volume mechanical "find and report".

## Rule of thumb
Cheapest model that can be correct. ALWAYS verify a worker's output before you
trust it — an error caught late costs more than the model you saved.

---
Install: copy the agents and merge these rules into your global instructions —
`cp agents/*.md ~/.claude/agents/` and `cat CLAUDE.md >> ~/.claude/CLAUDE.md`.
These rules are track-neutral: they apply to any Claude Code work (env / image /
text / anything), not just one project.
