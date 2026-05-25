# Case Title

## 1. Problem

Describe the user-visible failure or unexpected behavior.

Questions to answer:

- What did the user or system observe?
- What workflow or runtime path was involved?
- What made the behavior surprising?

## 2. Why It Matters

Explain why this is more than a local bug.

Map the issue to a broader engineering concern, such as runtime state consistency, resource accounting, sandbox boundary management, memory compaction policy, or regression risk.

## 3. Relevant Runtime Layer

Identify the layer where the problem appears to live.

Common layers include planner, executor, sandbox, memory manager, state registry, and UI-runtime synchronization.

## 4. Root-Cause Hypothesis

State the current hypothesis in a narrow and testable way.

Use cautious wording. For example: The likely cause is..., One possible failure path is..., or This suggests a mismatch between...

## 5. Fix Direction

Describe the smallest safe design or implementation direction.

Include tradeoffs when useful:

- What is the minimal fix?
- What should not be changed?
- What compatibility or performance risk exists?

## 6. Regression Test

Describe a test that would catch the same failure in the future.

A useful structure is: Given an initial condition, when an action happens, the expected assertion should hold.

## 7. Takeaway

Summarize the engineering principle learned from the case.

Keep it general enough to apply beyond the original project.
