---
name: "research-review"
description: "Provide critical feedback on research ideas, papers, or experiment results using the selected reviewer. Use for a research review; do not automatically implement findings or start an improvement loop. Base Codex semantic review is same-family provisional."
---

# Research Review via a secondary Codex agent (configured reasoning effort)

Apply [ARIS task scope and run limits](../shared-references/effort-contract.md#task-scope-and-run-limits) when interpreting defaults, checkpoints, and downstream calls.

Reviewer calls follow [the current routing contract](../shared-references/reviewer-routing.md). Tool examples use the host’s available native spawn/follow-up schema; omit model/effort unless explicitly selected, and isolate independent reviews from inherited conversation.


> **Codex assurance:** the fresh base reviewer is same-family. Record
> `review_independence: same-family` and `acceptance_status: provisional` in
> traces and deliverables. A Claude/Gemini overlay may record cross-family
> accepted; an unavailable reviewer is BLOCKED, never a fabricated PASS.

Get a multi-round critical review of research work from an external LLM with maximum reasoning depth.

## Constants

- REVIEWER_MODEL = current agent model unless the user explicitly selects another available reviewer.
- **REVIEWER_BACKEND = `codex`** — Default: Codex reviewer at the configured effort. Use `--reviewer: oracle-pro` only when explicitly requested; if Oracle is unavailable, warn and use the current configured reviewer only when that fallback is authorized. **Same-family note:** this default reviewer is a second Codex/GPT agent — valid for Type-A completeness/drive review, but not a cross-family Type-B verdict; install a `skills-codex-claude-review` / `skills-codex-gemini-review` overlay for a cross-family acquittal (see `shared-references/reviewer-routing.md`).

## Context: $ARGUMENTS

## Prerequisites

- Use `spawn_agent` and `send_input` when the user has explicitly allowed delegation or subagents.
- If delegation is not allowed, run the same review loop locally and preserve the same deliverable structure.

## Review scope

Return a review and prioritized recommendations. Do not edit the manuscript, implement experiments, or invoke an improvement pipeline unless the user also requested that work. One initial review normally suffices; use at most two targeted follow-ups for unresolved questions when they materially improve the requested review, unless the user sets another round limit. Reviewer agreement is not a reason to keep a completed review running.

## Workflow

### Step 1: Gather Research Context
Before calling the external reviewer, compile a comprehensive briefing:
1. Read project narrative documents (e.g., STORY.md, README.md, paper drafts)
2. Read any memory/notes files for key findings and experiment history
3. Identify: core claims, methodology, key results, known weaknesses

### Step 2: Initial Review (Round 1)
Send a detailed prompt at the configured reasoning effort:

```
spawn_agent:
  task_name: research_review_review
  fork_turns: none
  message: |
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

### Step 3: Iterative Dialogue (Rounds 2-N)
Use `send_input` with the returned agent id to continue the conversation:

```text
send_input:
  target: [saved reviewer id from Step 2]
  message: |
    Please continue the review using the revised materials below.

    Revised files:
    - /absolute/path/to/file1
    - /absolute/path/to/file2

    Focus on unresolved weaknesses and whether the revision actually fixed them.
```

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
[`output-composition.md`](../shared-references/output-composition.md).

### Step 6: Review Tracing

Save a trace for every `spawn_agent`, `send_input`, or `oracle-pro` review call following `../shared-references/review-tracing.md`. Record the reviewer route, saved agent id, prompt summary, raw response path, decisions, and action items. This preserves the Claude mainline Review Tracing semantics while using Codex-native reviewer calls.

## Key Rules

Use the current configured reasoning effort unless explicitly overridden; ARIS workload presets do not change model settings. See `../shared-references/reviewer-routing.md`.
- Give file-capable reviewers primary artifact paths to read directly. For an authorized HTTP reviewer without filesystem access, send the necessary raw source content, not only an executor summary.
- Be honest about weaknesses — hiding them leads to worse feedback
- Push back on criticisms you disagree with, but accept valid ones
- Focus on ACTIONABLE feedback — "what experiment would fix this?"
- Document the agent id for potential future resumption
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
