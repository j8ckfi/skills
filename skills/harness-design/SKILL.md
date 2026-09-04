---
name: harness-design
description: "Diagnose or design an agent harness using observed failures, current model capabilities, and measured task outcomes. Use for tool flow, context management, orchestration, and recovery decisions."
license: MIT
metadata:
  version: "1.0.0"
  source: https://alexzhang13.github.io/blog/2026/harness/
---

# Design and diagnose an agent harness

Use for an actual harness, tool-flow, or context-management problem. Start with the requested outcome, current model, tool contracts, and observed failure. A small tool addition does not automatically call for a new orchestrator or recursive architecture.

## Inspect

- Trace one representative run: instructions, retrieved context, tool calls and results, state persistence, error handling, and completion.
- Identify the concrete failure: missing context, excessive payloads, repeated work, lost authorization, incompatible tools, poor recovery, or incorrect results.
- Check current model and harness documentation. Do not assume a previous model's context limits or failure pattern still applies.

## Design choices

- Keep task intent, constraints, authorization, and decision-relevant evidence available to the model.
- Use bounded tool results and retrieve large artifacts by search, handles, or targeted reads when that improves the task. Handles must remain resolvable and carry enough provenance for decisions.
- Use supported compaction or summaries to preserve long-running state when appropriate. Compaction is a supported design option, not evidence that the harness has failed.
- Add subagents only when independent work and coordination benefits justify them and the environment permits delegation. Bound responsibilities and integrate results; agent count is not a success metric.
- Keep execution, permissions, retries, and external side effects explicit in the tool layer. Do not rely on prompt text alone to enforce authorization or resource limits.
- Prefer the simplest architecture that meets the measured workload. Ordinary long context, retrieval, file offloading, and recursive decomposition are alternatives to compare, not a fixed hierarchy.

## Verify

Compare representative tasks before and after the change using the same model and workload constraints. Measure task correctness, latency, token/cost usage, recovery, and any claimed scaling improvement. Include the failing case and meaningful larger cases when scale is the problem; do not impose an arbitrary 16× sweep on every edit.

Treat “locally in-distribution” as a research heuristic, not a directly enforceable guarantee about a model's training distribution. Results from a particular trained RLM and benchmark do not establish the best architecture for every newer model.

Report the observed bottleneck, the smallest justified change, verification results, and remaining limits. For research on context offloading and compositional generalization, see the source linked in this skill's metadata; verify its experimental setting before transferring its conclusions.
