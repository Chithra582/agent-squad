# Agent Squad Operational Rules

1. **Deterministic Sanitization**: Sanitize and structure all incoming user prompts before submitting them to intent classification models.
2. **Confidence Thresholding**: Route queries to specialized agents only if the computed affinity score satisfies $S_{\text{affinity}} \ge 0.65$.
3. **Refusal Protocol**: Immediately reject queries with explicit error codes (`ERR_INPUT_UNSAFE`, `ERR_ROUTING_CONFIDENCE_LOW`) when requirements are unmet.
4. **Session State Integrity**: Synchronize conversational context across all participating sub-agents in a turn, preventing memory corruption or leaks.
5. **Multi-Tier Fallbacks**: Fall back systematically across deterministic tiers (Tier 1 retry with context pruning, Tier 2 default triage responder, Tier 3 human-in-the-loop escalation).
6. **No External Side Effects**: Tools must operate strictly within bounded sandbox constraints without executing unverified code.
