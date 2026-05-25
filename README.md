# Engineering Debug Notes

This repository collects selected engineering notes from investigating public open-source Agent and Coding Agent issues.

The focus is not to criticize any project or maintainer. The goal is to understand how real agent systems fail at runtime, how those failures can be traced across system boundaries, and how fixes can be validated with narrow tests.

## Focus Areas

These notes mainly cover:

- Agent runtime reliability
- Subagent lifecycle and resource accounting
- Sandbox execution and environment propagation
- Memory compaction and context estimation
- Regression-test driven debugging
- Minimal-fix design and tradeoff analysis

## Repository Structure

```text
cases/
  codex-subagent-lifecycle.md
  openhands-sandbox-env-propagation.md
  letta-memory-compaction-token-estimate.md

templates/
  debug-note-template.md
```

## Note Format

Each case is organized around a practical debugging flow:

1. What was the user-visible failure?
2. Why does it matter for agent runtime design?
3. Which runtime layer is involved?
4. What is the root-cause hypothesis?
5. What is the minimal safe fix direction?
6. What regression test would prevent the issue from returning?
7. What broader engineering principle does the case illustrate?

## Scope and Attribution

These notes are based on public issues, public code reading, and independent debugging analysis. They are not official documentation for the upstream projects.

Unless a linked pull request explicitly states otherwise, the notes should be read as independent engineering case studies rather than claims of ownership over upstream fixes.

## Cases

| Case | Runtime area | Main idea |
|---|---|---|
| [Codex subagent lifecycle](cases/codex-subagent-lifecycle.md) | Subagent lifecycle, compaction, spawn quota | Restorable agent history should be separated from live execution-slot accounting. |
| [OpenHands sandbox env propagation](cases/openhands-sandbox-env-propagation.md) | Sandbox execution, configuration forwarding, security boundary | Important runtime flags must reach the execution layer without forwarding unrelated host secrets. |
| [Letta memory compaction token estimate](cases/letta-memory-compaction-token-estimate.md) | Memory compaction, context estimation, regression tests | Compaction decisions should be based on reusable input context size, not inflated total token usage. |
