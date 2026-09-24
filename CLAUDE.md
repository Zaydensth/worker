# Model fleet — hand-off rules

Eight tiered sub-agents live in `~/.claude/agents/`. Route every task to the cheapest
model that can do it correctly; reserve the top tier for real strategy, and spend a
top-tier model on *verification* before anything irreversible.

## Default pattern (C): a mid-tier model leads and reaches up through `Agent`
Main loop = **Sonnet 5** (`/model sonnet`). It carries routine OODA, runbooks, monitoring,
reporting and memory, and escalates only by spawning an agent.
(Pattern B: main loop = Opus for sustained implementation. Pattern A: main loop = Fable
for planning-heavy work. Both remain valid as a stated exception, not as the default.)

| Tier | Agent | Model | Reach for it when |
|---|---|---|---|
| leader | `strategist` | Fable 5 | a genuinely hard design call, a failure whose cause is not obvious, or the final go / no-go before a push, publish or submit. **At most once per situation** — bring the numbers, take back a decision, execute it yourself. |
| verify | `verifier` | Opus 5 | before a merge, a push, a registration, or anything that spends GPU hours. It tries to *refute* the claim and returns VERIFIED or REFUTED with evidence — never a rewrite. |
| execute | `executor` | Opus 5 | multi-file work on a critical path, VPS orchestration, or a fix the leader failed twice. |
| work | `worker-sonnet` | Sonnet 5 | code from a clear spec, moderate analysis, reading a subsystem, drafting scripts and docs. |
| work | `worker-haiku` | Haiku 4.5 | grep / glob, log scraping, extracting fields or numbers, format and existence checks, status polling — high-volume mechanical "find and report". |

Domain agents (`arm-runner`, `gate-auditor`, `tournament-intel`) are narrow by design:
each refuses work outside the runbook table it implements.

## KEEP — do it yourself (the main loop)
Implementing an agreed plan, multi-step execution and loops, routine diagnosis, deciding
what to delegate, and **verifying every worker's output before trusting it**.

## Effort
Valid values: `low` · `medium` · `high` · `xhigh` · `max`; absent means `high`. An
unrecognised value — including a session-mode name such as `ultracode` — is an unknown
frontmatter field and is **silently ignored**, so the agent falls back to the default with
no error. Every agent here is pinned to `max` except one.

**`worker-haiku` has no `effort` line, deliberately.** Claude Haiku 4.5 does not support the
effort parameter and errors when it is sent. Do not add it back.

## Rules that outrank convenience
- Cheapest model that can be correct — but never cheap for a decision that is expensive to
  get wrong.
- **Angka atau tidak terjadi**: never report "done" without the measurement that proves it.
  State the command you ran and the number it produced.
- Rank work by what it protects: first what makes you **fail outright**, then what converts
  the clock into useful work, then schedule completion, then metric honesty, and only then
  hyper-parameters.
- `disallowedTools` on the leaf agents is load-bearing. It is what stops a worker from
  spawning its own fan-out. Do not remove it to "unblock" something.

---
Install: `cp agents/*.md ~/.claude/agents/` and `cat CLAUDE.md >> ~/.claude/CLAUDE.md`.
These rules are track-neutral: they apply to any Claude Code work, not one project.
