---
name: "research-review"
description: "Provide critical feedback on research ideas, papers, or experiment results using the selected reviewer. Use for a research review; do not automatically implement findings or start an improvement loop. Uses the configured Claude review backend."
---

> Override for Codex users who want **Claude Code**, not a second Codex agent, to act as the reviewer. Install this package **after** `skills/skills-codex/*`.
>
> For a completed review whose actual reviewer is a different model family from the Codex executor, the overlay trace/audit records:
>
> ```yaml
> review_independence: cross-family
> acceptance_status: accepted
> ```

# Research Review via `claude-review` MCP (high-rigor review)

## Shared references

In this overlay, `shared-references/<file>` names a resource supplied by the **base Codex package**. Resolve that resource directory once:

1. For a merged/copied installation, use `<catalog-skills-directory>/shared-references/`. Derive the catalog's parent directory **before** following the overlay skill's symlink; do not append `../` to a resolved overlay path.
2. For direct checkout reading or a catalog that exposes only resolved paths, use `$ARIS_REPO/skills/skills-codex/shared-references/`. Preserve an explicit `ARIS_REPO`; otherwise find the containing checkout from the loaded SKILL.md real path (the ancestor with both `tools/` and `skills/skills-codex/`).

Open the named file there, following any stated section anchor. Read only resources needed for the current phase. If neither location exists, report the missing base support package and leave dependent work pending. This resolution does not change the overlay's reviewer provider.

## Claude review execution

Use the configured Claude bridge and preserve explicit model/effort choices supported by that bridge. Record the actual reviewer identity, raw response, job/thread ID, and verdict. `acceptance_status: accepted` describes the assurance class of a completed cross-family review; it never turns a negative verdict into PASS. Missing/unknown identity or failed review is unavailable/error evidence and cannot satisfy the gate.

Persist each job ID immediately. Poll that job with bounded waits until its terminal result or the configured review deadline; if no deadline is available, use a 15-minute monitoring cap. At the cap, report the pending job and resume its status later instead of starting a duplicate review. A timeout, authentication failure, or unavailable bridge does not authorize a provider switch. Continue independent preparation while leaving the required review pending.

For installation resources, follow the loaded SKILL.md real path to its ARIS checkout and resolve `ARIS_REPO` there, preserving an explicit setting. Project/personal skill directories may be symlinks; copied overlays still need the base package's resources. Use `$ARIS_REPO/mcp-servers/claude-review/server.py` for the matching local bridge, not an assumed file under `~/.codex`.

Apply [ARIS task scope and run limits](#shared-references) (`shared-references/effort-contract.md#task-scope-and-run-limits`) when interpreting defaults, checkpoints, and downstream calls.

> **Claude overlay assurance:** this route is a different model family from the Codex executor and records `review_independence: cross-family` plus `acceptance_status: accepted`.

Get a multi-round critical review of research work from an external LLM with maximum reasoning depth.

## Constants

- **REVIEWER_MODEL = `claude-review`** — Claude reviewer invoked through the local `claude-review` MCP bridge. Set `CLAUDE_REVIEW_MODEL` if you need a specific Claude model override.
- **REVIEWER_BACKEND = `claude-review`** — reviews route through the claude-review MCP (Claude family; cross-family for a Codex executor).

## Context: $ARGUMENTS

## Prerequisites

- Install the base Codex-native skills first: install `skills/skills-codex/*` through the project/personal Codex skill catalog.
- Then install this overlay package: use `skills/skills-codex-claude-review/*` as the selected overrides for the matching base skill names.
- Register the local reviewer bridge:
  ```bash
  codex mcp add claude-review -- python3 "$ARIS_REPO/mcp-servers/claude-review/server.py"
  ```
- This gives Codex access to `mcp__claude-review__review_start`, `mcp__claude-review__review_reply_start`, and `mcp__claude-review__review_status`.


## Review scope

Return a review and prioritized recommendations. Do not edit the manuscript, implement experiments, or invoke an improvement pipeline unless the user also requested that work. One initial review normally suffices; use at most two targeted follow-ups for unresolved questions when they materially improve the requested review, unless the user sets another round limit. Reviewer agreement is not a reason to keep a completed review running.

## Workflow

### Step 1: Gather Research Context
Before calling the external reviewer, compile a comprehensive briefing:
1. Read project narrative documents (e.g., STORY.md, README.md, paper drafts)
2. Read any memory/notes files for key findings and experiment history
3. Identify: core claims, methodology, key results, known weaknesses

### Step 2: Initial Review (Round 1)
Send a detailed prompt with ultra reasoning:

```
mcp__claude-review__review_start:
  prompt: |
    [Full research context + specific questions]
    Please act as a senior ML reviewer (NeurIPS/ICML level). Start from the
    assumption that the work is broken somewhere — your job is to find where.
    Be adversarial. Trust nothing the author tells you — verify everything
    yourself. Identify:
    1. Logical gaps or unjustified claims
    2. Missing experiments that would strengthen the story
    3. Narrative weaknesses
    4. Whether the contribution is sufficient for a top venue

    === SCOPE LIMITS (these bound what you PROPOSE, never what you look for) ===
    Report anything that is actually wrong here — including a rare-looking case, if
    this repo actually produces it. Then keep the fix in scope:
    1. This is a RESEARCH-WORKFLOW tool, not a security paper. Verification is
       welcome; over-defense is not. Assume a cooperating operator on their own
       machine — a malicious local user is NOT in the threat model.
    2. Do NOT propose SHA / hash / content-fingerprint / digest-binding schemes.
       Reporting a real defect in hashing code that already exists is fine.
    3. NO speculative machinery: do not add feature flags, migration frameworks,
       compat layers, wrappers, pins, or similar mechanisms unless evidence shows
       a current repo defect they fix or an explicit existing invariant they must
       preserve. "Load-bearing", "compatibility", and "not scaffolding" are labels,
       not evidence. Point to the failing path/artifact or invariant, and check the
       proposal's factual premises, such as whether a named package version exists.
    4. NO corner-case obsession: exotic encodings, symlink races, RTL text and
       millisecond races are out of scope unless you can show the case arises here.
    5. Where a rubric or checklist is genuinely needed, do not over-mechanize
       judgement. A clear sentence a human reads beats a scored table nobody
       maintains.
    Exception: code that runs remote commands, starts a network service, or installs
    an MCP server runs on the user's machine with their credentials — trust-boundary
    findings there are in scope and the default is strict.
    Say plainly when something is correct. Do not manufacture findings.
    Be brutally honest. If, after genuinely trying to break it, the work
    holds up and is ready, say so clearly.
```

After this start call, immediately save the returned `jobId` and poll `mcp__claude-review__review_status` with a bounded `waitSeconds` until `done=true`. Treat the completed status payload's `response` as the reviewer output, and save the completed `threadId` for any follow-up round.

### Step 3: Iterative Dialogue (Rounds 2-N)
Use `mcp__claude-review__review_reply_start` with the saved completed `threadId`, then poll `mcp__claude-review__review_status` with the returned `jobId` until `done=true` to continue the conversation:

```
mcp__claude-review__review_reply_start:
  # the bridge grants the reviewer no tools by default; this prompt passes
  # artifact paths, so it has to ask for read-only access explicitly
  tools: "Read,Grep,Glob"
  threadId: [saved reviewer id from Step 2]
  prompt: |
    Please continue the review using the revised materials below.

    Revised files:
    - /absolute/path/to/file1
    - /absolute/path/to/file2

    Focus on unresolved weaknesses and whether the revision actually fixed them.
```

After this start call, immediately save the returned `jobId` and poll `mcp__claude-review__review_status` with a bounded `waitSeconds` until `done=true`. Treat the completed status payload's `response` as the reviewer output, and save the completed `threadId` for any follow-up round.

For each round:
1. **Respond** to criticisms with evidence/counterarguments
2. **Ask targeted follow-ups** on the most actionable points
3. **Request specific deliverables**: experiment designs, paper outlines, claims matrices

Key follow-up patterns:
- "If we reframe X as Y, does that change your assessment?"
- "What's the minimum experiment to satisfy concern Z?"
- "Please design the minimal additional experiment package (highest acceptance lift per GPU week)"
- "Please write a mock NeurIPS/ICML review with scores"
- "Give me a results-to-claims matrix for possible experimental outcomes"

### Step 4: Convergence
Stop when the requested review is complete, the round limit is reached, or another round would repeat unchanged evidence. For an explicitly requested iterative discussion, useful completion signals include:
- Both sides agree on the core claims and their evidence requirements
- A concrete experiment plan is established
- The narrative structure is settled

### Step 5: Document Everything
Save the full interaction and conclusions to a review document in the project root:
- Round-by-round summary of criticisms and responses
- Final consensus on claims, narrative, and experiments
- Claims matrix (what claims are allowed under each possible outcome)
- Prioritized TODO list with estimated compute costs
- Paper outline if discussed

Update project memory/notes with key review conclusions.

If `— composed: <canonical-report-path>` is explicitly present, fold consensus,
claims matrix, TODOs, and trace links into that report instead of writing a
standalone review document. Without the directive, write the standalone review
as documented; never infer composed mode from an existing file. `— standalone`
always wins. See
[`output-composition.md`](#shared-references) (`shared-references/output-composition.md`).

### Step 6: Review Tracing

Save a trace for every `mcp__claude-review__review_start`, `mcp__claude-review__review_reply_start`, or `oracle-pro` review call following `shared-references/review-tracing.md`. Record the reviewer route, saved threadId, prompt summary, raw response path, decisions, and action items. This preserves the Claude mainline Review Tracing semantics while using Codex-native reviewer calls.

## Key Rules

- **Always ask the Claude reviewer for strict, high-rigor feedback** in every review round.
- Give file-capable reviewers primary artifact paths to read directly. For an authorized HTTP reviewer without filesystem access, send the necessary raw source content, not only an executor summary.
- Be honest about weaknesses — hiding them leads to worse feedback
- Push back on criticisms you disagree with, but accept valid ones
- Focus on ACTIONABLE feedback — "what experiment would fix this?"
- Document the completed `threadId` for potential future resumption
- The review document should be self-contained (readable without the conversation)

## Prompt Templates

### For initial review:
"I'm going to present a complete ML research project for your critical review. Please act as a senior ML reviewer (NeurIPS/ICML level)..."

### For experiment design:
"Please design the minimal additional experiment package that gives the highest acceptance lift per GPU week. Our compute: [describe]. Be very specific about configurations."

### For paper structure:
"Please turn this into a concrete paper outline with section-by-section claims and figure plan."

### For claims matrix:
"Please give me a results-to-claims matrix: what claim is allowed under each possible outcome of experiments X and Y?"

### For mock review:
"Please write a mock NeurIPS review with: Summary, Strengths, Weaknesses, Questions for Authors, Score, Confidence, and What Would Move Toward Accept."
