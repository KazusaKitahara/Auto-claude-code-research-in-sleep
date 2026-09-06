---
name: training-check
description: "Monitor identified training runs over a bounded interval for NaNs, divergence, stalls, or plateaus using W&B or logs. Use when ongoing training checks are requested; use monitor-experiment for one status snapshot."
argument-hint: "[wandb-run-path]"
allowed-tools: Bash(*), Read, Grep, Glob, Write, Edit, mcp__codex__codex, mcp__codex__codex-reply
---

# Training Check

Apply [ARIS task scope and run limits](../shared-references/effort-contract.md#task-scope-and-run-limits) when interpreting defaults, checkpoints, and downstream calls.

Periodically read WandB metrics during training to catch problems early. Do not wait until training finishes to discover it was a waste of GPU time.

> ⏱ This skill is **correctly** cron-wired (see below): it polls
> machine-checkable training health (NaN / divergence / idle GPU) — the additive
> external-wait shape in
> [`shared-references/external-cadence.md`](../shared-references/external-cadence.md).
> The occasional Codex call for an ambiguous metric is a **one-shot** check per
> tick, not a multi-round verdict loop, so it stays additive — it never grows
> into a wrapped verdict skill.

## Monitoring authority and duration

Monitoring alone does not authorize stopping or restarting a training job. `AUTO_STOP` is true only when the user has delegated intervention for the named run and its stop criteria; otherwise `STOP` is a recommendation and the job keeps running. A supplied stop command is an implementation detail, not authorization. Never kill unrelated sessions or delete checkpoints.

Use the requested monitoring duration or check count. If neither is supplied, perform the initial check and one follow-up, then report the state and end monitoring. Use a cancellable host scheduler/wait capability for long intervals; do not claim a background monitor remains active after the session ends. Stop monitoring when the identified run finishes, the limit is reached, or the user cancels.

## Context: $ARGUMENTS

## Constants

- WANDB_ENTITY and WANDB_PROJECT: read from CLAUDE.md or passed as argument (format: `entity/project/run_id`)
- CHECK_INTERVAL: starts at 10 minutes, then gradually increases if consistently healthy: 10 min → 20 min → 30 min → 60 min (cap)
- REVIEWER_MODEL = `gpt-5.6-sol` — used via Codex MCP for ambiguous cases only

## When to Use

- After training is confirmed running (session alive, loss decreasing for first few steps)
- Set up via CronCreate to fire periodically during training
- **This skill checks training QUALITY, not process HEALTH.** Process health (session alive, GPU utilization) is [watchdog.py](../../tools/watchdog.py)'s job.

## Workflow

### Step 1: Read WandB Metrics

```python
import wandb
api = wandb.Api()
run = api.run("<entity>/<project>/<run_id>")
history = run.history()
```

If WandB is unreachable (API error, network issue), fall back to reading the log file directly via SSH:
```bash
ssh server "tail -100 /path/to/training.log"
```

Check these signals:
- **Loss trend**: Is training loss decreasing over the last N steps?
- **Eval metrics**: Are evaluation metrics improving (or at least not degrading)?
- **NaN / Inf**: Any NaN or Inf values in loss or gradients?
- **Spikes**: Sudden large jumps in loss (>10x normal variance)?
- **Learning rate**: Is the schedule behaving as expected?
- **Gradient norm**: Exploding or vanishing?

### Step 2: Judgment

| Signal | Judgment | Action |
|--------|----------|--------|
| NaN/Inf in loss | **Clearly bad** | Recommend stopping; act only with `AUTO_STOP` authorization |
| Loss diverging (increasing for >N steps) | **Clearly bad** | Recommend stopping; act only with `AUTO_STOP` authorization |
| Eval metrics significantly worse than baseline | **Clearly bad** | Recommend stopping; act only with `AUTO_STOP` authorization |
| Loss decreasing, metrics improving | **Clearly fine** | Continue, increase check interval |
| Loss flat but not diverging | **Unsure** | → Step 3 (Codex judgment) |
| Metrics noisy, can't tell trend | **Unsure** | → Step 3 (Codex judgment) |
| Slightly worse than baseline but still early | **Unsure** | → Step 3 (Codex judgment) |

### Step 3: Codex Judgment (only when unsure)

Only escalate to Codex when the signal is ambiguous. For clearly good or clearly bad signals, act directly.

```
mcp__codex__codex:
  model: gpt-5.6-sol
  config: {"model_reasoning_effort": "xhigh"}
  prompt: |
    TRAINING HEALTH CHECK — need your judgment on ambiguous metrics.

    Run: <entity>/<project>/<run_id>
    Current epoch/step: X / Y total
    Training loss (last 10 checkpoints): [values]
    Eval metrics (last 3 evals): [values]
    Baseline reference: [numbers from paper/reproduction]

    What I'm unsure about: [specific concern]

    Please respond with exactly one of:
    - STOP: clearly problematic, should kill training
    - CONTINUE: looks fine, check again next interval
    - WAIT: not enough data to judge, check again sooner
```

### Step 4: Act

| Decision | Action |
|----------|--------|
| **Stop** | If `AUTO_STOP` is authorized, stop only the identified training session; otherwise report the recommendation. Save the WandB run URL, key metrics, and reason for stopping. Log to project notes for debugging. |
| **Continue** | Do nothing. Will be invoked again at next interval (increase interval if consistently healthy). |
| **Wait** | Do nothing but keep the current short interval (don't increase). |

## Integration with Watchdog

Training-check and [watchdog.py](../../tools/watchdog.py) operate at different levels:

| Layer | Tool | What it checks | Frequency |
|-------|------|----------------|-----------|
| Process health | watchdog.py | Session alive? GPU active? | Every 60s (continuous) |
| Training quality | training-check | Loss trend? Metrics improving? | Every 10-60 min (periodic) |

Use both together:
- Watchdog catches crashes and idle GPUs immediately
- Training-check catches subtle quality issues (loss plateau, metric degradation)

## Rules

- Do not stop training on first sign of noise — some loss spikes are normal. Look at **trends over multiple checkpoints**.
- When stopping training, always save the WandB run URL and key metrics as evidence.
- If both WandB and log files are unreachable, report the connectivity issue and try again next interval. Do not assume training is broken.
- Gradually increase check interval when healthy (10 → 20 → 30 → 60 min). Reset to 10 min after any anomaly.
- This skill is meant to be automated via CronCreate — do not ask the user whether to set it up. Just set it.

## CronCreate Setup Example

```
After training is confirmed stable:
  CronCreate (recurring, every 10 minutes initially):
    "Run /training-check for wandb run <entity>/<project>/<run_id>"
```

As the check interval increases, delete the old CronCreate job and create a new one with the longer interval.
