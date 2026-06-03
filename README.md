# Engineering Debug Notes

This repository collects selected engineering notes from investigating public Agent and Coding Agent issues.

The goal is to understand how real agent systems fail at runtime, how those failures can be traced across system boundaries, and how fixes can be validated with focused tests.

## Focus Areas

- Agent runtime reliability
- Subagent lifecycle and resource accounting
- Runtime configuration propagation
- API contract and validator consistency
- Memory compaction and context estimation
- Regression-test driven debugging
- Minimal-fix design and tradeoff analysis

## Repository Structure

```text
cases/
  codex-subagent-lifecycle.md
  runtime-config-propagation.md
  langgraph-api-openapi-validator-drift.md
  memory-compaction-token-estimate.md

templates/
  debug-note-template.md
```

## Case Format

Each note follows a practical debugging structure:

1. Problem
2. Why it matters
3. Relevant runtime layer
4. Root-cause hypothesis
5. Fix direction
6. Regression test
7. Takeaway

## Cases

| Case | Runtime area | Main idea |
|---|---|---|
| [Codex subagent lifecycle](cases/codex-subagent-lifecycle.md) | Subagent lifecycle, compaction, spawn quota | Restorable agent history should be separated from live execution-slot accounting. |
| [Runtime config propagation](cases/runtime-config-propagation.md) | Cross-layer runtime setup | Runtime configuration must reach the layer that actually applies it, while propagation remains explicit and minimal. |
| [LangGraph API OpenAPI validator drift](cases/langgraph-api-openapi-validator-drift.md) | API schema, request validation, streaming contract | Validation logic should enforce the current runtime contract, not an outdated OpenAPI snapshot. |
| [Memory compaction token estimate](cases/memory-compaction-token-estimate.md) | Memory compaction, context estimation | Compaction decisions should be based on reusable input context size, not total usage from the previous step. |

## Scope

These notes are independent engineering case studies based on public issues and code reading.