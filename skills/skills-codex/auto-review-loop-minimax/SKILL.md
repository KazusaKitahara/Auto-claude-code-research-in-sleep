---
name: "auto-review-loop-minimax"
description: "Run a bounded research review and revision loop with MiniMax as the reviewer. Use when the user requests a MiniMax improvement loop; a single MiniMax review does not imply implementation."
---

# Auto Review Loop (MiniMax Version): Autonomous Research Improvement

Apply [ARIS task scope and run limits](../shared-references/effort-contract.md#task-scope-and-run-limits) when interpreting defaults, checkpoints, and downstream calls.

Autonomously iterate: review → implement fixes → re-review, until the external reviewer gives a positive assessment or MAX_ROUNDS is reached.

## Context: $ARGUMENTS

## Loop boundaries

Resolve the requested target, permitted edits, and concrete round/time/compute limits before starting. Reuse prior authorization for in-scope fixes. A review-only request ends with findings; an iterative repair request runs the loop. Stop on completion, cancellation, the first limit, or no material progress in two successive rounds, and report remaining issues. A reviewer error is not permission to switch providers, rerun a possibly dispatched paid call, or count a missing review as a pass.

## Constants

- MAX_ROUNDS = 4
- POSITIVE_THRESHOLD: score >= 6/10 AND verdict ∈ {"ready", "almost"} — both must hold, matching the operative STOP CONDITION below. Verdict vocabulary is {"ready", "almost", "not ready"}. (Earlier wording used "or" + a stale verdict set; the AND form is authoritative.)
- REVIEW_DOC: `review-stage/AUTO_REVIEW.md` (cumulative log) *(fall back to `./AUTO_REVIEW.md` for legacy projects)*
- REVIEWER_MODEL = `MiniMax-M3` — Model used via MiniMax API

## API Configuration

This skill uses MiniMax API for external review. Two methods are supported:

### Method 1: MCP Tool (Primary)

If `mcp__minimax-chat__minimax_chat` is available, use it:

```
mcp__minimax-chat__minimax_chat:
  message: |
    [Review prompt content]
  model: "MiniMax-M3"
  system: "You are a senior machine learning researcher..."
```

### Method 2: Direct HTTP (Fallback)

If MCP is unavailable before dispatch and direct HTTP is authorized, use this request pattern:

```bash
# Write a JSON object with a messages array to .aris/review-request.json using a file-writing tool.
# These JSON files are local evidence artifacts, not shell templates.
python3 - <<'PY_REVIEW'
import json
import os
from pathlib import Path
from urllib.request import Request, urlopen

payload = json.loads(Path(".aris/review-request.json").read_text())
payload["model"] = os.environ.get("MINIMAX_MODEL", "MiniMax-M3")
url = "https://api.minimax.io/v1/chat/completions"
request = Request(url, data=json.dumps(payload, ensure_ascii=False).encode("utf-8"),
                  headers={"Content-Type": "application/json",
                           "Authorization": "Bearer " + os.environ["MINIMAX_API_KEY"]},
                  method="POST")
try:
    with urlopen(request, timeout=120) as response:
        raw = response.read()
except Exception:
    raise SystemExit("Review delivery unconfirmed; inspect state before retrying")
Path(".aris/review-response.json").write_bytes(raw)
print("Raw review saved to .aris/review-response.json")
PY_REVIEW
```

**API Key**: Use `MINIMAX_API_KEY` from the process environment or the installed MCP server’s secret configuration. Do not read or print the entire host settings file.

**Transport**: This skill uses the configured MiniMax Chat Completions route. Check the installed server and provider API rather than inferring support from a Codex endpoint.

## Transport and evidence limits

A remote chat API cannot read local paths. Send the necessary primary artifact contents through the authorized route, with source names, alongside the task; do not substitute only executor-written summaries. Preserve the raw response and actual provider-reported model, and identify the executor/reviewer families. Unknown or same-family semantic review remains provisional and cannot satisfy a required cross-family acceptance gate. Use the relevant acceptance and reviewer-independence contracts.

Use the selected model and only parameters supported by its endpoint. Retry only an explicitly rejected pre-dispatch capability request; do not switch providers or resend after an ambiguous timeout. For a possibly dispatched call, report review unavailable and preserve its state. A missing provider blocks that review, while independent local preparation can continue.

## State Persistence (Compact Recovery)

Long-running loops may hit the context window limit, triggering automatic compaction. To survive this, persist state to `review-stage/REVIEW_STATE.json` after each round:

```json
{
  "round": 2,
  "status": "in_progress",
  "last_score": 5.0,
  "last_verdict": "not ready",
  "pending_experiments": ["screen_name_1"],
  "timestamp": "2026-03-13T21:00:00"
}
```

**Write this file at the end of every Phase E** (after documenting the round). Overwrite each time — only the latest state matters.

**On completion** (positive assessment or max rounds), set `"status": "completed"` so future invocations don't accidentally resume a finished loop.

## Workflow

### Initialization

1. **Check for `review-stage/REVIEW_STATE.json`** *(fall back to `./REVIEW_STATE.json` if not found — legacy path)*:
   - If neither path exists: **fresh start** (normal case)
   - If it exists AND `status` is `"completed"`: **fresh start** (previous loop finished normally)
   - If it exists AND `status` is `"in_progress"` AND `timestamp` is older than 24 hours: **inspect before resuming** (stale state may still refer to live jobs; preserve it, check the target and pending jobs, and resume only if this invocation requests that work)
   - If it exists AND `status` is `"in_progress"` AND `timestamp` is within 24 hours: **resume**
     - Read the state file to recover `round`, `last_score`, `pending_experiments`
     - Read `review-stage/AUTO_REVIEW.md` to restore full context of prior rounds *(fall back to `./AUTO_REVIEW.md`)*
     - If `pending_experiments` is non-empty, check if they have completed (e.g., check screen sessions)
     - Resume from the next round (round = saved round + 1)
     - Log: "Recovered from context compaction. Resuming at Round N."
2. Read project narrative documents, memory files, and any prior review documents
3. Read recent experiment results (check output directories, logs)
4. Identify current weaknesses and open TODOs from prior reviews
5. Initialize round counter = 1 (unless recovered from state file)
6. Create/update `review-stage/AUTO_REVIEW.md` with header and timestamp

### Loop (repeat up to MAX_ROUNDS)

#### Phase A: Review

Send comprehensive context to the external reviewer.

**Check MCP availability first**, then use appropriate method:

**If MCP available (Primary):**
```
Use mcp__minimax-chat__minimax_chat tool with:
- system: "You are a senior machine learning researcher serving as a reviewer for top-tier conferences like NeurIPS, ICML, and ICLR. Provide rigorous, constructive feedback."
- prompt: [Full review prompt with context]
- model: "MiniMax-M3"
```

**If MCP NOT available (Fallback):**
```bash
# Write a JSON object with a messages array to .aris/review-request.json using a file-writing tool.
# These JSON files are local evidence artifacts, not shell templates.
python3 - <<'PY_REVIEW'
import json
import os
from pathlib import Path
from urllib.request import Request, urlopen

payload = json.loads(Path(".aris/review-request.json").read_text())
payload["model"] = os.environ.get("MINIMAX_MODEL", "MiniMax-M3")
url = "https://api.minimax.io/v1/chat/completions"
request = Request(url, data=json.dumps(payload, ensure_ascii=False).encode("utf-8"),
                  headers={"Content-Type": "application/json",
                           "Authorization": "Bearer " + os.environ["MINIMAX_API_KEY"]},
                  method="POST")
try:
    with urlopen(request, timeout=120) as response:
        raw = response.read()
except Exception:
    raise SystemExit("Review delivery unconfirmed; inspect state before retrying")
Path(".aris/review-response.json").write_bytes(raw)
print("Raw review saved to .aris/review-response.json")
PY_REVIEW
```

**Note**: Each round is a standalone API call. For round 2+, include the summary of previous reviews and changes in the prompt itself.

#### Phase B: Parse Assessment

**CRITICAL: Save the FULL raw response** from the external reviewer verbatim (store in a variable for Phase E). Do NOT discard or summarize — the raw text is the primary record.

Then extract structured fields:
- **Score** (numeric 1-10)
- **Verdict** ("ready" / "almost" / "not ready")
- **Action items** (ranked list of fixes)

**STOP CONDITION**: If score >= 6 AND verdict ∈ {"ready", "almost"} (exact match — "not ready" does NOT qualify) → stop loop, document final state.

#### Phase C: Implement Fixes (if not stopping)

For each action item (highest priority first):

1. **Code changes**: Write/modify experiment scripts, model code, analysis scripts
2. **Run experiments**: Deploy to GPU server via SSH + screen/tmux
3. **Analysis**: Run evaluation, collect results, update figures/tables
4. **Documentation**: Update project notes and review document

Prioritization rules:
- Skip fixes requiring excessive compute (flag for manual follow-up)
- Skip fixes requiring external data/models not available
- Prefer reframing/analysis over new experiments when both address the concern
- Always implement metric additions (cheap, high impact)

#### Phase D: Wait for Results

If experiments were launched:
- Monitor remote sessions for completion
- Collect results from output files and logs

#### Phase E: Document Round

Append to `review-stage/AUTO_REVIEW.md`:

```markdown
## Round N (timestamp)

### Assessment (Summary)
- Score: X/10
- Verdict: [ready/almost/not ready]
- Key criticisms: [bullet list]

### Reviewer Raw Response

<details>
<summary>Click to expand full reviewer response</summary>

[Paste the COMPLETE raw response from the external reviewer here — verbatim, unedited.
This is the authoritative record. Do NOT truncate or paraphrase.]

</details>

### Actions Taken
- [what was implemented/changed]

### Results
- [experiment outcomes, if any]

### Status
- [continuing to round N+1 / stopping]
```

**Write `review-stage/REVIEW_STATE.json`** with current round, score, verdict, and any pending experiments.

Increment round counter → back to Phase A.

### Termination

When loop ends (positive assessment or max rounds):

1. Update `review-stage/REVIEW_STATE.json` with `"status": "completed"`
2. Write final summary to `review-stage/AUTO_REVIEW.md`
3. Update project notes with conclusions
4. If stopped at max rounds without positive assessment:
   - List remaining blockers
   - Estimate effort needed for each
   - Suggest whether to continue manually or pivot

## Key Rules

- **Large file handling**: For a demonstrated file-size limit, use a permitted literal-safe writer or smaller chunks. Preserve the intended content and current filesystem permissions; a permission denial is not a file-size problem.

- Be honest — include negative results and failed experiments
- Do NOT hide weaknesses to game a positive score
- Implement fixes BEFORE re-reviewing (don't just promise to fix)
- If an experiment takes > 30 minutes, launch it and continue with other fixes while waiting
- Document EVERYTHING — the review log should be self-contained
- Update project notes after each round, not just at the end
- For round 2+, always include previous review context in the prompt
- Prefer MCP tool over curl when available (more reliable)

## Prompt Template for Round 2+

**MCP Method (Primary):**
```
mcp__minimax-chat__minimax_chat:
  model: "MiniMax-M3"
  system: "You are a senior machine learning researcher serving as a reviewer for top-tier conferences like NeurIPS, ICML, and ICLR. Provide rigorous, constructive feedback."
  message: |
    [Round N/MAX_ROUNDS of autonomous review loop]

    ## Previous Review Summary (Round N-1)
    - Previous Score: X/10
    - Previous Verdict: [ready/almost/not ready]
    - Previous Key Weaknesses: [list]

    ## Changes Since Last Review
    1. [Action 1]: [result]
    2. [Action 2]: [result]
    3. [Action 3]: [result]

    ## Updated Results
    [paste updated metrics/tables]

    ## Current Research Context
    [brief summary of claims, methods, current state]

    Please re-score and re-assess:
    1. Score this work 1-10 for a top venue
    2. List remaining critical weaknesses (ranked by severity)
    3. For each weakness, specify the MINIMUM fix
    4. State clearly: is this READY for submission? Yes/No/Almost

    Be brutally honest. If the work is ready, say so clearly.
```

**curl Fallback:**
```bash
# Write a JSON object with a messages array to .aris/review-request.json using a file-writing tool.
# These JSON files are local evidence artifacts, not shell templates.
python3 - <<'PY_REVIEW'
import json
import os
from pathlib import Path
from urllib.request import Request, urlopen

payload = json.loads(Path(".aris/review-request.json").read_text())
payload["model"] = os.environ.get("MINIMAX_MODEL", "MiniMax-M3")
url = "https://api.minimax.io/v1/chat/completions"
request = Request(url, data=json.dumps(payload, ensure_ascii=False).encode("utf-8"),
                  headers={"Content-Type": "application/json",
                           "Authorization": "Bearer " + os.environ["MINIMAX_API_KEY"]},
                  method="POST")
try:
    with urlopen(request, timeout=120) as response:
        raw = response.read()
except Exception:
    raise SystemExit("Review delivery unconfirmed; inspect state before retrying")
Path(".aris/review-response.json").write_bytes(raw)
print("Raw review saved to .aris/review-response.json")
PY_REVIEW
```


## Output Protocols

> Follow these shared protocols for all output files:
> - **[Output Versioning Protocol](../../shared-references/output-versioning.md)** — write timestamped file first, then copy to fixed name
> - **[Output Manifest Protocol](../../shared-references/output-manifest.md)** — log every output to MANIFEST.md
> - **[Output Language Protocol](../../shared-references/output-language.md)** — respect the project's language setting
