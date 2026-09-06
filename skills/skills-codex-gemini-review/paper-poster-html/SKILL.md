---
name: paper-poster-html
description: "Create or revise an academic conference poster in HTML/CSS with real paper figures, layout checks, and print-ready PDF export. Use for research posters when this format fits the request. Uses the configured Gemini review backend."
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, mcp__gemini-review__review, mcp__gemini-review__review_start,
  mcp__gemini-review__review_reply_start, mcp__gemini-review__review_status
metadata:
  argument-hint: '[paper-dir-or-pdf] [— venue: ICLR, canvas: 185x90cm landscape, venue-colors: true]'
---

> Override for Codex users who want **Gemini**, not a second Codex/Codex-MCP reviewer, to act as the reviewer. Install this package **after** `skills/skills-codex/*`.

# Paper Poster (HTML): measurement-gated poster generation

## Shared references

In this overlay, `shared-references/<file>` names a resource supplied by the **base Codex package**. Resolve that resource directory once:

1. For a merged/copied installation, use `<catalog-skills-directory>/shared-references/`. Derive the catalog's parent directory **before** following the overlay skill's symlink; do not append `../` to a resolved overlay path.
2. For direct checkout reading or a catalog that exposes only resolved paths, use `$ARIS_REPO/skills/skills-codex/shared-references/`. Preserve an explicit `ARIS_REPO`; otherwise find the containing checkout from the loaded SKILL.md real path (the ancestor with both `tools/` and `skills/skills-codex/`).

Open the named file there, following any stated section anchor. Read only resources needed for the current phase. If neither location exists, report the missing base support package and leave dependent work pending. This resolution does not change the overlay's reviewer provider.

## Gemini review execution

Use the configured Gemini bridge and preserve explicit model/effort choices supported by that bridge. Record the actual reviewer identity, raw response, job/thread ID, and verdict. `acceptance_status: accepted` describes the assurance class of a completed cross-family review; it never turns a negative verdict into PASS. Missing/unknown identity or failed review is unavailable/error evidence and cannot satisfy the gate.

Persist each job ID immediately. Poll that job with bounded waits until its terminal result or the configured review deadline; if no deadline is available, use a 15-minute monitoring cap. At the cap, report the pending job and resume its status later instead of starting a duplicate review. A timeout, authentication failure, or unavailable bridge does not authorize a provider switch. Continue independent preparation while leaving the required review pending.

For installation resources, follow the loaded SKILL.md real path to its ARIS checkout and resolve `ARIS_REPO` there, preserving an explicit setting. Project/personal skill directories may be symlinks; copied overlays still need the base package's resources. Use `$ARIS_REPO/mcp-servers/gemini-review/server.py` for the matching local bridge, not an assumed file under `~/.codex`.

Apply [ARIS task scope and run limits](#shared-references) (`shared-references/effort-contract.md#task-scope-and-run-limits`) when interpreting defaults, checkpoints, and downstream calls.

> **Gemini overlay assurance:** `review_independence: cross-family` and `acceptance_status: accepted`.

One HTML file styled for an exact print canvas (`@page { size: W H }`), rendered to PDF
via Playwright print emulation. **Iterate by measuring, not eyeballing** — the screen
preview lies; only print emulation at the correct viewport tells the truth. Core gate
machinery is adapted from [posterly](https://github.com/Chenruishuo/posterly) (MIT, ©
2026 Ruishuo Chen — see `NOTICE.md` and `LICENSES/posterly-MIT.txt` in the mainline
skill directory); ARIS adds style discipline gates, figure-provenance gates, the
cross-model review loop, and the anti-patch-loop fix vocabulary.

This overlay is identical to `skills/skills-codex/paper-poster-html/` except that the
two cross-model review calls go to **Gemini** through the local `gemini-review` MCP
bridge instead of a spawned GPT reviewer agent. Follow the base mirror for everything
not restated here (phases, gates, fix vocabulary, figure provenance, output contract). Resolve that base SKILL.md at `$ARIS_REPO/skills/skills-codex/paper-poster-html/SKILL.md`; the installed overlay path may have replaced it, so do not recursively reload the overlay as its own base.

## Reviewer constants (overlay)

- **REVIEWER_MODEL = `gemini-review`** — Gemini invoked through the local
  `gemini-review` MCP bridge.
- **Fresh review job per call** — start each review with
  `mcp__gemini-review__review_start`; never reuse a prior review job across review
  boundaries. Save the returned `jobId`, poll `mcp__gemini-review__review_status` with
  a bounded `waitSeconds` until `done=true`, and treat the completed payload's
  `response` as the reviewer output.
- The Gemini bridge cannot read your local files — paste the relevant content into the
  prompt, and pass rendered posters via `imagePaths`.
- If the `gemini-review` bridge is unavailable, stop and tell the user what to
  configure. Do not silently degrade the cross-model reviews into self-review.

## Phase 1 step 2 — Cross-model content audit (Gemini)

```text
mcp__gemini-review__review_start:
  prompt: |
    Audit a conference-poster content plan against its source paper.

    ## Poster content plan
    [PASTE poster_html/POSTER_CONTENT_PLAN.md]

    ## Paper source (relevant sections)
    [PASTE the paper sections backing the plan's claims — abstract, headline
    results tables, method equations, theorem statements]

    For EVERY claim, number, equation, and attribution in the plan, output one row:
    | claim on poster | paper location | paper says (verbatim) | match? |
    with match ∈ {OK, NUMERIC-MISMATCH, OVERCLAIM, MISSING-PRECONDITION,
    NOT-IN-PAPER, SCOPE-NARROWED}. End with a count per category.
```

Poll `review_status` until `done=true`; save the response to
`poster_html/CLAIM_EVIDENCE.md`. Fix every non-OK row or record it as a
user-acknowledged tradeoff.

## Phase 6 — Final review (Gemini, multimodal)

All hard gates PASS + polish warnings zero-or-waived + executor visual score ≥ 9
first. Then:

```text
mcp__gemini-review__review_start:
  imagePaths: ["poster_html/poster_preview.png"]
  prompt: |
    Final print-readiness audit of a conference poster (image attached).

    ## Final poster text content
    [PASTE the text content extracted from poster_html/poster.html]

    ## Gate report summary
    [PASTE the overall/hard_failures/warnings fields of poster_html/GATE_REPORT.json]

    ## Claim→evidence audit
    [PASTE poster_html/CLAIM_EVIDENCE.md]

    Check: (1) fidelity & overclaims RE-CHECKED on the final text (polish introduces
    new claims), (2) residue (\ref{, TODO, raw < in math, missing images, remote
    URLs), (3) visual rhetoric (headline numbers prominent, banner readable from
    2 m, two-hue discipline, real paper figures central and inside their cards),
    (4) gate-log coherence.
    Verdict: PRINT-READY or NEEDS-FIX with a numbered, severity-ordered issue list.
```

Poll `mcp__gemini-review__review_status` with a bounded `waitSeconds` until `done=true`;
treat the completed payload's `response` as the reviewer verdict.

The reviewer recommends; it does not edit. Any fix → back through Phase 4/5 gates —
never straight to re-review.

## Review tracing

Save both review jobs' raw responses per `shared-references/review-tracing.md` to
`.aris/traces/paper-poster-html/<date>_run<NN>/`.
