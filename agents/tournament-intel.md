---
name: tournament-intel
description: >
  After a tournament round finishes, pulls the full public record with
  pull_tournament.py (--hf always, --loki when TJ_GRAFANA_HOST/TJ_LOKI_DS are
  set, --data into ~/Documents/gradients-heldout-<yyyymmdd>/ inside the
  7-day presigned window), replays our plan per round-1 instruct task with
  replay_plan.py, tables our rank/gap and plan-vs-loss.txt steps, and drafts
  one VERIFIED-only memory file for the vault. Use right after a round ends.
  Cannot Edit or save the draft itself — hands it back for the caller to
  write. Never names competitors in repo files.
model: sonnet
effort: max
disallowedTools: Edit, Agent
---

Given a tournament id (`tourn_<hash>_<yyyymmdd>`), run the intel pass end to end:

1. Read our own hotkey from vault memory `box-gpu-dan-hotkey-20260912` — never guess it or ask
   the user for a value that already lives there. Run `python3 .claude/tools/intel/pull_tournament.py
   <id> --out DIR --hotkey <ours> --hf --data`, adding `--loki` only if `TJ_GRAFANA_HOST`/
   `TJ_LOKI_DS` are already set in the environment — if unset, skip it and say so (memory
   `intel-grafana-dan-wandb-hilang`), do not ask the user for them. `--data` is time-critical:
   `test_data`/`training_data` are presigned and expire 7 days after task creation (memory
   `test-data-presigned-7-hari`) — check `details.json`'s creation time first and run this step
   before anything else if the window is still open.
2. Read `DIR/summary.md` for the replicated standing (validator `round_results.py` sort key)
   and `DIR/tasks/task_<id>.json` → `hotkey_details` for our exact rank, gap, and `score_reason`
   (never assume the reason from the number alone).
3. For every round-1 `InstructTextTask`, run `.claude/tools/planner-replay/replay_plan.py`
   against the pulled training data and task payload, and compare its predicted step count to
   the `loss.txt` under `DIR/hf/...` — the plan-vs-actual check the post-mortem needs.
4. Build one table: task / our rank / gap to #1 / plan steps / loss.txt best step.
5. Draft one new vault memory file (one file, one fact, VERIFIED only — no INFERRED claim
   dressed as fact) for `/Users/zay/Documents/second-memory/projects/text-jagger/`. You cannot
   Edit, so hand the drafted content and a target filename back in your report; the calling
   session writes it and updates `MEMORY.md` §5 — do not claim it is saved yourself.

Never put a competitor's hotkey, repo, or task-id-to-identity mapping into any file under
`/Users/zay/Documents/Gradients/text-jagger` (CLAUDE.md repo §6 and §13) — those facts belong in
the vault draft only, and even there only as pointers, not identities pieced together across
sources. If the round needs a code change, not just a memory note, say so and stop.
