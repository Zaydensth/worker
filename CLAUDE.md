# Model fleet — hand-off rules

Seven tiered sub-agents live in `~/.claude/agents/`. **One model decides, one executes what
was decided, one does the volume.** Route down for anything with a written spec and a
checkable output; escalate up for a decision that is expensive if wrong, a failure whose
cause is not obvious, or an irreversible action.

## Tier 1 — Fable 5.1 decides, and it is mandatory

| Point | When | Produces |
|---|---|---|
| **LOCK** | before a plan or pre-registration is frozen | `FABLE-PLAN:` line in the lock file |
| **RE-SCOPE** | the data contradicts the plan | a new plan, not an improvised patch |
| **GO / NO-GO** | before a merge, push, publish or submit | an explicit verdict |
| **POST-MORTEM** | after the result lands | the synthesis |

This replaces the older "at most once per task" ceiling: consulting tier 1 is a **gate**, not
a budget line. The unit is a **plan, not a step** — one locked plan covers dozens of cells,
and cells inside it need no further tier-1 call.

**Anchor it to an artifact.** Each call appends one line to `FABLE_LEDGER.md` and writes
`FABLE-PLAN:` into the lock file; the next step greps for it and refuses without it. Break
glass by recording `VERDICT=UNAVAILABLE` or `VERDICT=BYPASS-USER` — never by silently
proceeding, and never by idling a machine you are paying for.

## The ladder

| Tier | Agent | Model | Reach for it when |
|---|---|---|---|
| decide | `strategist` | `claude-fable-5-1` | the four points above. Bring the numbers; take back a plan or a verdict, not an edit. |
| execute | `executor` | `claude-opus-5-5` | reading the plan in detail and carrying it out: multi-file critical-path work, VPS orchestration, a fix the main loop failed twice. |
| execute | `verifier` | `claude-opus-5-5` | before a merge, a push, a registration, or anything that spends a budget you don't get back. It tries to *refute* the claim and returns VERIFIED or REFUTED with evidence — never a rewrite. |
| execute | `arm-runner` | `claude-opus-5-5` | one pre-registered A/B arm, end to end, with its verdict applied. |
| work | `worker-sonnet` | `claude-sonnet-5` | code from a clear spec, moderate analysis, reading a subsystem, drafting scripts and docs — **and all mechanical work**: grep / glob, log tails, extracting fields and numbers, format and existence checks, status polling. |
| work | `gate-auditor` | `claude-sonnet-5` | evaluate ship gates for a SHA and write the evidence line. |
| work | `tournament-intel` | `claude-sonnet-5` | pull the public record after a round, replay, draft the memory note. |

Tier 3 is **not a direction-setter**. It runs what the data already settles, and it owns every
loop — through `Monitor`, `CronCreate` or `/loop`, never a sleep loop.

The `worker-haiku` tier was removed on 2026-09-24. Mechanical work moves up to `worker-sonnet`,
or down to plain `Bash` in the calling session when it is pure `grep` / `jq` / `tail` — that
path costs no model tokens at all and is deterministic, which is usually the better answer.

## Backup model

Tier 2 asks for `claude-opus-5-5`; the backup is `"fallbackModel": ["claude-opus-5"]` in
`~/.claude/settings.json`. It must be an array (a string is silently ignored), it is
session-wide (main loop + every sub-agent; no per-agent fallback exists), and it fires on an
unavailable or overloaded model — **not** on 429 / usage limits.

## KEEP — do it yourself (the main loop)

Carrying out an agreed plan, multi-step execution, routine diagnosis, deciding what to
delegate, and **verifying every worker's output before trusting it**.

## Effort

Valid values: `low` · `medium` · `high` · `xhigh` · `max`. Absent means the model's default —
`medium` for Opus 5.5 — so `effort: max` on every agent does real work. An unrecognised value — including a session-mode name such as `ultracode` — is
an unknown frontmatter field and is **silently ignored**, so the agent falls back to its
default with no error. Every agent here is pinned to `max`; there is no exception.

Verify a mass effort change by reading the frontmatter back, not by trusting that the `sed`
exited 0. **And verify `model:` from the transcript, not the file** — a typo'd model ID does
not error (on 2.1.280 it was silently served as Sonnet), and what is served can differ from what
was asked (Opus sub-agents ran on Opus 5 with 5.5 as advisor):

```sh
grep -ho '"model":"[^"]*"' ~/.claude/projects/<slug>/<session>/subagents/agent-*.jsonl | sort -u
```

## Rules that outrank convenience

- **Route down, then verify up.** The failure mode is not overspending; it is a cheap-model
  error caught late — wrong at tier 3, believed at tier 2, shipped at tier 1.
- Never cheap for a decision that is expensive to get wrong.
- **Angka atau tidak terjadi** — never report "done" without the measurement that proves it.
  State the command you ran and the number it produced.
- Rank work by what it protects: first what makes you **fail outright**, then what converts
  the clock into useful work, then schedule completion, then metric honesty, and only then
  hyper-parameters.
- `disallowedTools` on the leaf agents is load-bearing. It is what stops a worker from
  spawning its own fan-out. Do not remove it to "unblock" something.
- A lever whose default leaves it never running is not a feature — it is dead code.

---
Install: `cp agents/*.md ~/.claude/agents/` and `cat CLAUDE.md >> ~/.claude/CLAUDE.md`.
These rules are track-neutral: they apply to any Claude Code work, not one project.
