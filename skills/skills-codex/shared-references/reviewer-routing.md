# Codex reviewer routing

Use the current agent model and reasoning settings by default. Respect an explicit reviewer/model choice. Do not pin a prior-generation model, force an effort tier, or downgrade on a timeout. The active host's tool schema determines available parameters and follow-up tools.

For a requested independent review, use a fresh isolated agent with no inherited conversation. Pass the review task, scope, raw artifact paths, and output contract; keep executor conclusions and other reports out. The reviewer reads the evidence directly.

Illustrative pattern (adapt to the exposed native tool):

```text
spawn_agent:
  task_name: independent_review
  fork_turns: none
  message: |
    Review the supplied artifacts against the stated criteria.
    Read these files directly: [absolute paths].
    Return grounded findings and the requested verdict fields.
```

Model and effort are omitted to inherit the current configuration. If explicitly overridden, check the available enum and model first. Follow-up rounds use the host's follow-up capability with the saved reviewer ID and revised artifacts. Do not call a tool merely because an old example names it.

Keep the ARIS assurance semantics:

- A fresh GPT-family reviewer of a GPT-family executor is `review_independence: same-family` and positive conclusions remain `acceptance_status: provisional`.
- Independent context reduces shared-context bias; it does not prove cross-family independence or correctness.
- A user-selected compatible Claude/Gemini overlay may supply cross-family review. Keep actual model, effort, provenance, and verdict in the trace. Never invent a successful independent review.
- When a requested reviewer is unavailable, report `REVIEW_UNAVAILABLE` for that review and finish independent preparation/revision work. Only a phase whose defined acceptance condition requires that review remains pending.
- Do not repeatedly submit unchanged artifacts to obtain a pass. Re-review after relevant changes, within the requested run limits; terminate unproductive loops with the unresolved finding.

Oracle or external reviewer services are optional and require the user's selected route and available capability. Do not silently substitute a different provider or upload private artifacts to one. Existing authorization for a configured review route need not be requested again.

Use [reviewer independence](reviewer-independence.md), [review tracing](review-tracing.md), and [external cadence](external-cadence.md) when those operations apply. Mainline skills under `skills/` and alternate-client overlays have their own routing; this file governs the base Codex package only.
