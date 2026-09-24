# worker — multi-model agent fleet for Claude Code

A cost-efficient, tiered set of Claude Code sub-agents. Instead of running the most
powerful (and most expensive) model for every step, each task goes to the cheapest
model that can do it correctly, with the top tier reserved for real strategy and for
adversarial verification before anything irreversible.

## The fleet

| Role | Agent | Model | Effort | Used for |
|---|---|---|---|---|
| Leader | `strategist` | Fable 5 | `max` | plan / hard calls / go-no-go (plan-only) |
| Executor | `executor` | Opus 5 | `max` | implementation, loops, VPS ops |
| Verifier | `verifier` | Opus 5 | `max` | adversarially refute a claim before a merge / push / submit |
| Worker | `worker-sonnet` | Sonnet 5 | `max` | code from spec, parallel reading, drafts |
| Worker | `worker-haiku` | Haiku 4.5 | *(none — see below)* | grep, log-scrape, extract, poll (mechanical) |
| Domain | `arm-runner` | Opus 5 | `max` | run one pre-registered A/B arm end to end on the GPU box |
| Domain | `gate-auditor` | Sonnet 5 | `max` | evaluate ship gates G1-G4 for a SHA, with evidence |
| Domain | `tournament-intel` | Sonnet 5 | `max` | pull the public record after a round, replay, draft a memory |

Each role pins a `model` and an `effort`. Tool access is scoped: `strategist` and
`verifier` are plan-only (no `Write`/`Edit`/`Agent`), workers cannot spawn sub-agents,
and `worker-haiku` is read-and-report only. **Those `disallowedTools` lines are
load-bearing — deleting one turns a leaf agent into one that can fan out.**

### Valid `effort` values, and the one model that rejects it

`low` · `medium` · `high` · `xhigh` · `max`. Default is `high` when the field is absent.
Anything else — including a session-mode name such as `ultracode` — is an **unknown
frontmatter field and is silently ignored**, which means the agent quietly falls back to
the default with no error. If you want maximum, the word is `max`.

**`worker-haiku` carries no `effort` line on purpose.** Claude Haiku 4.5 does not support
the effort parameter — sending it errors at the API level. Adding `effort:` back to that
file breaks the agent.

## Operating patterns

- **Pattern C (current default)** — main loop = **Sonnet 5** (`/model sonnet`). It carries
  routine OODA, runbooks and monitoring, and reaches up only through `Agent`: `verifier`
  (Opus) to refute a claim, `executor` (Opus) for critical-path multi-file work,
  `strategist` (Fable) at most once per task for a genuinely hard call.
- **Pattern B** — main loop = Opus; it executes and escalates to `strategist` about once
  per task. Use when the work is sustained implementation.
- **Pattern A** — main loop = Fable; it plans and delegates execution down. Use for
  planning-heavy work.

## Install

```sh
cp agents/*.md ~/.claude/agents/
cat CLAUDE.md >> ~/.claude/CLAUDE.md      # or merge by hand
```

A project-scoped `.claude/agents/` overrides `~/.claude/agents/` for a per-repo variant.

## Why it saves

The plan — where the top model's intelligence actually matters — is a small fraction of
the tokens; execution and mechanical work are the bulk. Sub-agents also run in isolated
context, so a worker doesn't pay to re-read the whole session, and the main-loop model is
the one that re-reads context each turn — the single biggest cost lever, which is why
Pattern C puts Sonnet there.

Note the tradeoff this repo currently makes: every effort is pinned to `max`, so the
savings come from **model choice and context isolation**, not from effort tuning. If a
route is high-volume and latency-sensitive, lower its agent's effort rather than its model.

## How routing works

Claude reads each agent's `description` to auto-delegate; `CLAUDE.md` carries the explicit
hand-off rules. Keep the top tier out of routine work, and always verify a worker's output
before trusting it — a cheap wrong answer trusted late costs more than the model it saved.
