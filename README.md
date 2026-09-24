# worker — a three-tier Claude Code agent fleet

Seven sub-agents for Claude Code, arranged so that **one model decides, one model executes
what was decided, and one model does the volume**. The point isn't "always use the cheap
model" — it's that deciding, executing and grinding are three different jobs, and running all
three on one model makes at least one of them worse.

## The hierarchy

| Tier | Model | Owns | Must not |
|---|---|---|---|
| **1 — Decide** | Claude Fable 5.1 | the plan, the design call, the diagnosis when the cause isn't obvious, the go / no-go | run the work — it cannot edit files or spawn agents |
| **2 — Execute** | Claude Opus 5.5 (backup: Opus 5) | reading tier 1's plan *in detail* and carrying it out; adversarial verification before anything irreversible | re-decide. A plan that doesn't cover the case goes back to tier 1 |
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

## Opus 5.5, with Opus 5 as the backup

The tier-2 agents ask for `claude-opus-5-5`. The backup is Claude Code's own fallback chain, set
once in `~/.claude/settings.json`:

```json
{ "fallbackModel": ["claude-opus-5"] }
```

Read from the 2.1.280 source, not guessed:

- It must be an **array**. A plain string is silently ignored.
- It is **session-wide**: the main loop *and* every sub-agent use the same chain. Frontmatter
  takes exactly one model string; there is no per-agent fallback.
- It fires when the primary is **unavailable or overloaded**: model-not-found, a model
  permission error, repeated 529s, a 5xx, or a server-side model block. It does **not** fire on
  429 / usage limits, so hitting an Opus quota will not move you to Opus 5.
- It lasts one turn; the primary is retried at the start of the next user turn, and the switch
  shows up as a `model_fallback` system message.

## Check what actually runs — from the transcript

A frontmatter file tells you what was *asked for*. The sub-agent transcript records what was
*served*, per assistant message, including the effort that reached the request:

```sh
grep -ho '"model":"[^"]*"\|"advisorModel":"[^"]*"\|"effort":"[^"]*"' \
  ~/.claude/projects/<slug>/<session>/subagents/agent-*.jsonl | sort -u
```

Never ask the agent: models are unreliable narrators of their own identity and cannot see
their effort setting at all. What this turned up on one build (2.1.280), in 12 dispatches:

- `claude-fable-5-1` and `claude-sonnet-5` were served exactly as asked, and every custom
  agent's requests carried `"effort":"max"`. The frontmatter really does land.
- A **main session** on `/model claude-opus-5-5` was served Opus 5.5. Sub-agents asking for
  any Opus ID were served **Opus 5 with Opus 5.5 attached as an advisor** (the experimental
  advisor tool), whatever the parent model was. Keeping the frontmatter at 5.5 costs nothing:
  if 5.5 starts being served directly to sub-agents, they pick it up with no edit.
- A **typo'd model ID did not error**. It was silently served as Sonnet. Grep your
  frontmatter against the exact IDs you mean before trusting a fleet.

## Effort

`low` · `medium` · `high` · `xhigh` · `max`. Absent means the model's default — `medium` for
Opus 5.5, `high` for the others — so the explicit `effort: max` on every agent does real work,
and the transcript confirms it reaches the request.

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
