# worker — a three-tier Claude Code agent fleet

Seven sub-agents for Claude Code, arranged so that **one model decides, one model executes
what was decided, and one model does the volume**. The point isn't "always use the cheap
model" — it's that deciding, executing and grinding are three different jobs, and running all
three on one model makes at least one of them worse.

## The hierarchy

| Tier | Model | Owns | Must not |
|---|---|---|---|
| **1 — Decide** | Claude Fable 5.1 | the plan, the design call, the diagnosis when the cause isn't obvious, the go / no-go | run the work — it cannot edit files or spawn agents |
| **2 — Execute** | Claude Opus 5.5 | reading tier 1's plan *in detail* and carrying it out; adversarial verification before anything irreversible | re-decide. A plan that doesn't cover the case goes back to tier 1 |
| **3 — Work** | Claude Sonnet 5 | everything the data already settles: monitoring, ops, data pulls, drafts, reports, **and every loop** | set direction. It executes the errand; it does not choose the errand |

## The fleet

| Agent | Tier | Model | Effort | Used for |
|---|---|---|---|---|
| `strategist` | 1 | `claude-fable-5-1` | `max` | the plan, the hard call, the go / no-go (plan-only) |
| `executor` | 2 | `claude-opus-5-5` | `max` | implementation, multi-file critical-path work, VPS orchestration |
| `verifier` | 2 | `claude-opus-5-5` | `max` | adversarially refute a claim before a merge / push / submit |
| `arm-runner` | 2 | `claude-opus-5-5` | `max` | run one pre-registered A/B arm end to end on a GPU box |
| `worker-sonnet` | 3 | `claude-sonnet-5` | `max` | code from a spec, parallel reading, drafts, and all mechanical work |
| `gate-auditor` | 3 | `claude-sonnet-5` | `max` | evaluate ship gates for a SHA, with evidence |
| `tournament-intel` | 3 | `claude-sonnet-5` | `max` | pull the public record after a round, replay, draft a memory |

## Tier 1 is a gate, not a habit

The failure this layout is built against is a main loop that quietly starts deciding for
itself — picking the next experiment because it is nearby, not because it is the right one.
So **tier 1 is consulted before work starts, not when tier 2 feels stuck.** Four points, and
none of them is optional:

1. **LOCK** — before a plan or a pre-registration is frozen.
2. **RE-SCOPE** — when the data contradicts the plan. Not "adjust and carry on".
3. **GO / NO-GO** — before anything irreversible: a merge, a push, a submission.
4. **POST-MORTEM** — the synthesis after the result lands.

The unit is a **plan, not a step**. One locked plan covers dozens of cells; cells inside it
need no further tier-1 call. That is what keeps a mandatory gate from becoming a tax.

**Anchor the gate to an artifact, not to intent.** A gate that lives only in a prompt is a
gate that a tired session skips. Have tier 1 emit a line the work can be checked against —
a `FABLE-PLAN:` line in the lock file and an append-only `FABLE_LEDGER.md` entry — and have
the downstream step *open the file and grep for it*:

```sh
grep -q '^FABLE-PLAN: ' "$LOCKFILE" || { echo 'REFUSE: no plan on record'; exit 1; }
```

No line, no GPU. Break glass by recording `VERDICT=UNAVAILABLE` or `VERDICT=BYPASS-USER`
explicitly — so a missing tier 1 never idles a machine you're paying for, and never
disappears silently either.

## Effort

`low` · `medium` · `high` · `xhigh` · `max`. Absent means `high` — **except Claude Opus 5.5,
whose default is `medium`**, which is why the `effort: max` line on the tier-2 agents is
load-bearing rather than decorative.

Anything else — including a session-mode name such as `ultracode` — is an **unknown
frontmatter field and is silently ignored**: the agent falls back to its default with no
error, no warning, and a config that *looks* applied. If you mean maximum, the word is `max`.
Every agent here is pinned to `max`; there is no exception row.

## Tool scoping is load-bearing

`strategist`, `verifier` and `tournament-intel` cannot `Write`, `Edit` or spawn an `Agent`.
`gate-auditor` cannot `Edit` or spawn. `worker-sonnet` and `arm-runner` cannot spawn.
Sub-agents really can delegate, so those `disallowedTools` lines are what keep a leaf a leaf
— **deleting one to "unblock" something turns a worker into an unbounded fan-out.**

## Install

```sh
cp agents/*.md ~/.claude/agents/
cat CLAUDE.md >> ~/.claude/CLAUDE.md      # or merge by hand
```

A project-scoped `.claude/agents/` overrides `~/.claude/agents/` for a per-repo variant.

## Why it works

The plan is a small fraction of the tokens; execution and volume are the bulk. Sub-agents run
in isolated context, so a worker doesn't pay to re-read the whole session — and the main-loop
model, which *does* re-read context every turn, is the single biggest cost lever. Putting
Sonnet there and reaching up through `Agent` is what makes a mandatory tier-1 gate affordable:
four Fable calls in a cycle cost less than an afternoon of Fable running the main loop.

Every effort is pinned to `max`, so the savings come from **model choice and context
isolation**, not from effort tuning. If a route is high-volume and latency-sensitive, lower
that agent's effort rather than its model.

## Route down, then verify up

The failure mode is not overspending. It is a cheap-model error caught late: a wrong number
extracted at tier 3, believed at tier 2, shipped at tier 1. Every cheap output a decision
rests on gets re-derived by the tier above it before anyone acts on it.
