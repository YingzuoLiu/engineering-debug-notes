# Memory Compaction Token Estimate

## 1. Problem

This note studies a memory-runtime issue in a long-running agent system.

The system keeps an estimate of how large the current context is. After a model step finishes, the runtime updates this estimate and later uses it to decide whether memory compaction should run.

The problem is that the runtime may confuse two different values:

```text
all tokens used by the previous model step
```

and

```text
the reusable input context that will matter for the next step
```

Those values can be very different.

For example, a model call may have a small input context but a long generated answer. If the runtime treats the whole previous usage as current context size, it may believe the context is much larger than it really is.

That can trigger memory compaction earlier than necessary.

## 2. Why It Matters

Memory compaction is one of the core mechanisms that allows an agent to keep working across a long session.

A compaction policy usually tries to balance two risks:

```text
compact too late
  -> context may become too large

compact too early
  -> useful details may be summarized away too soon
```

The second failure mode is subtle. The agent may still appear to run normally, but repeated early compaction can gradually remove details such as file names, user constraints, intermediate decisions, or earlier observations.

This means the issue is not only a local accounting mistake. It affects long-horizon reliability.

A bad estimate can cause the memory system to compress information at the wrong time. Over multiple steps, that can lead to summary drift and weaker task continuity.

## 3. Relevant Runtime Layer

The relevant layer is the memory manager and context estimation logic.

This is not mainly a planner problem. It is also not mainly a tool-calling problem.

The simplified runtime flow is:

```text
model step finishes
  -> usage information is returned
  -> runtime updates context-size estimate
  -> memory policy checks whether compaction is needed
  -> older context may be summarized
```

If the estimate is wrong, the compaction decision can be wrong even when the model output itself is valid.

## 4. Root-Cause Hypothesis

The likely cause is using a broad usage number as a proxy for reusable context size.

A previous model step may include:

```text
input context
+ generated output
+ other model-side usage
```

But memory compaction should mainly care about what will continue to occupy the next request's input context.

The root mismatch is:

```text
previous-step usage
  is treated as
current reusable context size
```

That mismatch can inflate the estimate and make the runtime trigger compaction too aggressively.

## 5. Fix Direction

A narrow fix direction is to estimate reusable context size from input-side usage when that value is available.

Conceptually:

```text
if input-side usage is available and positive:
    use it as the context estimate
else:
    fall back to the previous broad usage value
```

This is a conservative fix.

It avoids two problems:

1. It prevents generated output from inflating the context estimate when input-side usage is available.
2. It avoids setting the estimate to zero when a model provider does not report input-side usage.

A more precise alternative would be to count the current internal message context directly. That could be more accurate, but it may also add more runtime cost and touch more code paths.

For a small runtime fix, preferring input-side usage with a fallback is a safer first step.

## 6. Regression Tests

Two regression tests are useful.

### Test 1: prefer input-side usage

```text
Given: input-side usage is 1000
And: broad total usage is 6000
When: the runtime estimates reusable context size
Expected: the estimate is 1000
```

This catches the original inflation problem.

### Test 2: fallback when input-side usage is missing

```text
Given: input-side usage is missing or zero
And: broad total usage is 6000
When: the runtime estimates reusable context size
Expected: the estimate falls back to 6000
```

This protects compatibility with providers that do not report all usage fields consistently.

## 7. Tradeoff

The key tradeoff is precision versus change size.

Using input-side usage is simple and low-risk, but it still depends on what the model provider reports.

Counting the internal context directly may be more precise, but it can require more changes and may introduce extra latency.

For a targeted bug fix, the smaller change is usually easier to review and safer to land.

## 8. Takeaway

The main engineering principle is:

> Agent memory policy should be driven by reusable context size, not simply by the total usage of the previous model step.

Long-horizon agents do not only need more memory. They need reliable memory-control signals.

If those signals are wrong, the agent may forget important details because the runtime compressed memory at the wrong time, not because the model failed to reason.
