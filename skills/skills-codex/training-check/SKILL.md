---
name: training-check
description: "Monitor identified training runs over a bounded interval for NaNs, divergence, stalls, or plateaus using W&B or logs. Use when ongoing training checks are requested; use monitor-experiment for one status snapshot."
metadata:
  argument-hint: '[wandb-run-or-monitoring-brief]'
---

# Training Check

Apply [ARIS task scope and run limits](../shared-references/effort-contract.md#task-scope-and-run-limits) when interpreting defaults, checkpoints, and downstream calls.

You are now in **interactive watch** / 交互式训练监控模式.

Keep the current session open and report directly in the current terminal. The user is watching this terminal for updates. By default, run a training health check every 30 minutes, output a concise but complete analysis report after each check, state the next check time, then continue monitoring.

This skill checks training **quality**, not basic process health. Process health checks such as whether a tmux session exists or whether the GPU is idle can be handled by watchdog-style tooling; this skill focuses on whether the run is still worth continuing.

## Monitoring authority and duration

Monitoring alone does not authorize stopping or restarting a training job. `AUTO_STOP` is true only when the user has delegated intervention for the named run and its stop criteria; otherwise `STOP` is a recommendation and the job keeps running. A supplied stop command is an implementation detail, not authorization. Never kill unrelated sessions or delete checkpoints.

Use the requested monitoring duration or check count. If neither is supplied, perform the initial check and one follow-up, then report the state and end monitoring. Use a cancellable host scheduler/wait capability for long intervals; do not claim a background monitor remains active after the session ends. Stop monitoring when the identified run finishes, the limit is reached, or the user cancels.

## Inputs To Establish First

Before the first check, identify or ask for the minimum monitoring context:

- WandB run path or URL, if available.
- Fallback log path, SSH command, or local command for reading recent training logs.
- Training target, expected baseline, and key metrics that define success.
- How the training was launched, so it can be stopped if needed.
- Project notes path for recording decisions and evidence.

If a source is unavailable, say so clearly and continue with the available source. If both WandB and fallback logs are unreachable, report the connectivity issue, classify the round as `WAIT`, and check again later. Do not infer that training is bad only because data is unreachable.

## Per-Round Check

Every round, read WandB first when configured. If WandB is unreachable, read the fallback logs. Inspect at least:

- Training loss trend over recent checkpoints or steps.
- Eval metrics and whether they improve, flatten, or degrade against baseline.
- NaN or Inf in loss, gradients, activations, or logged metrics.
- Sudden loss spikes, divergence, or repeated failed evaluations.
- Learning rate schedule behavior.
- Gradient norm, if logged.
- Plateau patterns that suggest the run is no longer useful.

Output one report in the current terminal with this structure:

```text
## Training Check - <local timestamp>

- Data source: wandb_ok | log_fallback | unreachable
- Run: <wandb run or training identifier>
- Recent metrics: <loss/eval/lr/grad summary>
- Anomalies: <NaN/Inf/spike/divergence/plateau findings>
- Evidence: <WandB URL, log lines, metric values, or files inspected>
- Decision: CONTINUE | WAIT | STOP
- Reason: <why this decision is justified>
- Next check: <local timestamp, normally 30 minutes later unless ending>
```

Use the decisions as follows:

| Decision | Meaning | Action |
|----------|---------|--------|
| `CONTINUE` | Run looks healthy enough to keep training. | Keep monitoring and check again in 30 minutes. |
| `WAIT` | Evidence is inconclusive, noisy, too early, or temporarily unreachable. | Do not stop training; keep monitoring and check again later. |
| `STOP` | Training meets the configured stop criteria. | Stop only if `AUTO_STOP` is authorized; otherwise report the recommendation. Save evidence and finish this check. |

## Stop Behavior

When the decision is `STOP` and `AUTO_STOP` is authorized:

- Stop the training task.
- If the context contains `stop_command`, run `stop_command` first.
- If no `stop_command` is available, choose the appropriate stop action from how the training was launched, such as stopping the relevant tmux session, local process, remote process, scheduler job, or notebook job.
- Save evidence: WandB URL, key metrics, relevant log snippets, files inspected, and the reason for stopping.
- Append a project note for debugging and future analysis.
- Output `FINAL_SUMMARY` in the terminal.
- End the interactive monitoring loop.

Never stop on the first sign of ordinary metric noise. Look for sustained trends, hard failures, or clear divergence. Always preserve enough evidence for a later agent or human to understand why the run was stopped.

## Interactive Loop Guidance

- The normal interval is 30 minutes.
- If a round is `CONTINUE`, announce the next check time and wait until then.
- If a round is `WAIT`, explain what evidence is missing or noisy and check again later. Use a shorter interval only when the run looks suspicious but not yet stop-worthy.
- If an anomaly recovers, say so explicitly and continue monitoring.
- Keep the user-facing report short enough to read in a terminal, but include concrete metric values and evidence paths.
