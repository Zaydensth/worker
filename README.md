# worker — multi-model agent fleet for Claude Code

A cost-efficient, tiered set of Claude Code sub-agents. Instead of running the most
powerful (and most expensive) model for every step, each task goes to the cheapest
model that can do it correctly, with the top tier reserved for real strategy.

## The tiers

| Tier | Role | Model | Price /1M (in/out) | Used for |
|---|---|---|---|---|
| Leader | `strategist` | Fable 5 | $10 / $50 | plan / critical thinking / go-no-go (plan-only) |
| Co-Leader | `executor` | Opus 4.8 | $5 / $25 | implementation, loops, VPS ops, verification |
| Worker | `worker-sonnet` | Sonnet 5 | $3 / $15 | code from spec, parallel reading, drafts |
| Worker | `worker-haiku` | Haiku 4.5 | $1 / $5 | grep, log-scrape, extract, poll (mechanical) |

Each role is pinned to a `model` and an `effort` in its frontmatter. Tool access is
scoped: `strategist` is plan-only (no `Write`/`Edit`/`Agent`), workers cannot spawn
sub-agents, and `worker-haiku` is read-and-report only.

## Two operating patterns

- **Pattern B (default)** — main loop = Opus (`/model opus`); it executes and
  escalates to `strategist` (Fable) about once per task for hard calls. Best when
  the work is mostly sustained execution with occasional hard strategy.
- **Pattern A** — main loop = Fable (`/model fable`); it plans and delegates
  execution down to the Opus / Sonnet / Haiku workers. Best for planning-heavy work.

## Install

Copy the agent files into your Claude Code agents directory:

```sh
cp agents/*.md ~/.claude/agents/
```

Add the hand-off rules to your global instructions:

```sh
cat CLAUDE.md >> ~/.claude/CLAUDE.md      # or merge by hand
```

A project-scoped `.claude/agents/` overrides `~/.claude/agents/` if you want a
per-repo variant of a role.

## Why it saves

The plan — where the top model's intelligence actually matters — is a small
fraction of the tokens; execution and mechanical work are the bulk. Running the
bulk on Opus / Sonnet / Haiku instead of Fable is where the savings are: a
mechanical log-scrape on Haiku is ~10× cheaper than on Fable. Sub-agents also run
in isolated context, so a worker doesn't pay to re-read the whole session, and the
main-loop model (Opus, not Fable) is the one that re-reads context each turn — the
single biggest cost lever.

## How routing works

Claude reads each agent's `description` to auto-delegate, and `CLAUDE.md` gives the
explicit hand-off rules. Keep the top tier out of routine work; always verify a
worker's output before trusting it.
