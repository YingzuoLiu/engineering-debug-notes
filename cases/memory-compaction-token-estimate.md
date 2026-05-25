# Memory Compaction Token Estimate

This case note studies context-size estimation in a long-running memory system.

The main lesson is that a compaction policy should estimate reusable input context, not simply reuse the total usage value from the previous step.
