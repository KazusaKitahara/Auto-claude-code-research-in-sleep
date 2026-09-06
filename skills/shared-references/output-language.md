# Output Language Protocol

## Language Detection

Determine human-readable output language using this priority:
1. The user's explicit language request, including established session preferences.
2. A relevant project language setting (for example `language: zh` in the project status).
3. The current conversation language; otherwise English.

Use a venue's required submission language for the filing artifact when producing that requested artifact. Keep commentary and review reports in the user's requested language. If a requested translation is not suitable for filing, deliver the translation and label that limitation instead of silently changing its language.

## What to Localize

- Section headings and labels
- Descriptions, analysis, commentary, recommendations
- Template boilerplate text
- Status messages and warnings

## What NOT to Localize

- Code, shell commands, file paths, directory names
- Paper titles, author names, venue names, BibTeX entries
- Technical terms with no standard Chinese translation (keep English, optionally annotate: "attention mechanism (注意力机制)")
- LaTeX commands and labels remain unchanged; translate prose according to the requested document language.
- JSON state files — keys and structure remain English
- **Machine-parsed markers** — never localize the following, regardless of language setting:
  - Markdown frontmatter keys (e.g., `outcome:`, `node_id:`, `title:`, `type:`)
  - Research Wiki schema fields parsed by `tools/research_wiki.py` (e.g., `outcome: negative`, `outcome: positive`, `node_id:`)
  - `MANIFEST.md` column headers and table structure
  - Any field that downstream tools or scripts read programmatically

## Skill-Specific Rules

| Skill | Language Support | Notes |
|-------|-----------------|-------|
| /idea-creator | Full | IDEA_REPORT.md follows language setting |
| /idea-discovery | Full | Inherits from sub-skills |
| /analyze-results | Full | Result analysis follows language setting |
| /auto-review-loop | Partial | AUTO_REVIEW.md follows setting; reviewer prompts stay English |
| /experiment-plan | Full | EXPERIMENT_PLAN.md follows setting |
| /experiment-bridge | Full | EXPERIMENT_RESULTS.md follows setting |
| /research-refine | Full | FINAL_PROPOSAL.md follows setting |
| /research-refine-pipeline | Full | PIPELINE_SUMMARY.md follows setting |
| /research-pipeline | Full | Inherits from sub-skills |
| /result-to-claim | Full | Claim descriptions follow setting |
| /paper-writing | Full | Use the requested draft language or verified venue submission language |
| /paper-write | Full | Preserve commands while writing prose in the requested language |
