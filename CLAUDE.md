# Model fleet — hand-off rules

Read this before doing expensive work yourself. Four tiers; the cheapest model that
can do the job correctly wins. Price per 1M tokens (input / output):
**Fable 5 $10/$50 · Opus 4.8 $5/$25 · Sonnet 5 $3/$15 · Haiku 4.5 $1/$5.**

## Default operating pattern (B): Opus leads, escalates to Fable once
Main loop = Opus 4.8 (`/model opus`). It carries the sustained work and pulls in
Fable only for hard strategy. (Pattern A — Fable as the main loop, delegating
execution down — is available too: set `/model fable` when the task is
planning-heavy.)

## ESCALATE UP → strategist (Fable 5), ~once per task, only for:
- designing a new teacher / training recipe / overall approach
- diagnosing a failure whose cause is not obvious (needs deep reasoning)
- final go/no-go before pushing miner code or an on-chain submit
→ spawn `strategist` with full context, take its plan, then execute it yourself.

## KEEP — do it yourself (the Opus main loop / `executor`):
implementing a plan, automation / OODA loops, VPS ops, verifying worker output,
routine diagnosis, deciding what to delegate.

## DELEGATE DOWN → worker-sonnet (Sonnet 5):
code from a clear spec, moderate analysis, parallel reading of a subsystem,
drafting gen-scripts, CPU-verify, drafting summaries.

## DELEGATE DOWN → worker-haiku (Haiku 4.5):
grep / log-scrape / tail, extracting numbers from JSON, format & SHA checks,
status / cron polling — high-volume mechanical "find and report".

## Rule of thumb
Cheapest model that can be correct. ALWAYS verify a worker's output before you
trust it — an error caught late costs more than the model you saved.
