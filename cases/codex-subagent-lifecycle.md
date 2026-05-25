# Codex Subagent Lifecycle and Spawn-Slot Accounting

## 1. Problem

This note studies a Codex CLI multi-agent runtime issue where new subagent creation may fail after previous subagents have already completed.

The user-visible symptom is similar to:

```text
Agent spawn failed
agent thread limit reached
```

A typical observed flow is:

1. A session spawns several subagents.
2. Those subagents complete their work.
3. The session is compacted or continues after a long run.
4. The same session later tries to spawn more subagents.
5. The runtime still reports that the thread or spawn limit has been reached.

The interesting part is that this may happen even when the earlier subagents are no longer doing useful live work.

## 2. Why It Matters

This is not just a local error message. It points to a general runtime problem in long-running agent systems:

> Historical or restorable agent records should not be confused with live execution capacity.

A completed subagent may still need to remain visible for history, inspection, or recovery. However, that does not necessarily mean it should continue consuming a live execution slot.

This distinction matters for long-horizon agents because stale resource accounting can cause the system to stop making progress even when no actual work is running.

## 3. Relevant Runtime Layer

The relevant layer is the agent runtime lifecycle layer.

The issue touches several concepts:

- subagent creation
- spawn-slot reservation
- live thread or worker accounting
- completion notification
- compaction or resume behavior
- registry state that survives within the same session

Conceptually, the runtime needs to separate:

```text
Restorable history
  Old subagent thread records, outputs, and metadata that may still be useful.

Live execution slot
  Capacity reserved for currently active or reusable execution work.
```

## 4. Root-Cause Hypothesis

The likely failure mode is a lifecycle accounting mismatch.

A simplified path is:

```text
parent agent spawns child subagent
  -> runtime reserves a spawn slot
  -> child reaches a final status
  -> parent receives a completion notification
  -> child metadata or history remains available
  -> live spawn slot is not necessarily released
  -> later spawn attempts still see the old slot as occupied
```

Compaction can make the problem more visible because compaction mainly compresses or replaces conversation history. It does not necessarily rebuild or clean the in-memory agent registry.

So a more precise framing is:

> The issue is better understood as stale live-slot accounting than as compaction directly corrupting the count.

There may also be resume-related risks if completed child edges are restored as open children. But the same-session behavior points strongly to stale in-memory lifecycle accounting.

## 5. Fix Direction

A safe design should not simply delete completed subagents.

Deleting them would risk losing useful historical information such as:

- final answer
- task path
- parent-child relationship
- inspectable thread history
- recovery or resume metadata

A better design is to separate metadata retention from quota accounting:

```text
completed subagent metadata exists
  does not imply
completed subagent consumes live spawn quota
```

A possible fix direction is:

1. When a child reaches a final status, notify the parent as usual.
2. Release only the live spawn slot.
3. Preserve metadata and history for inspection or recovery.
4. Make the release operation idempotent so repeated final-status events do not decrement the counter twice.
5. If a completed agent is later reused for new work, reacquire a slot before sending new execution.

This keeps both properties:

- completed agents remain inspectable;
- live execution capacity is not leaked.

## 6. Regression Tests

Useful regression tests should cover both resource release and history preservation.

### Test 1: completed child releases live slot

```text
Given: max subagent threads is set to 1
When: child agent A is spawned and reaches a final completed state
Then: child agent B can be spawned without manually closing A
```

### Test 2: completed child remains inspectable

```text
Given: child agent A completed and its slot was released
When: the runtime lists or inspects previous child agents
Then: A's metadata or history is still available
```

### Test 3: release is idempotent

```text
Given: a counted child agent reaches final status
When: release is called more than once for the same child
Then: the live count is decremented only once
```

### Test 4: compaction does not revive completed children as live capacity

```text
Given: a child completed before compaction
When: session history is compacted
Then: the child remains restorable but does not consume a live spawn slot
```

## 7. Takeaway

The main engineering principle is:

> A subagent can be completed, visible, and restorable while no longer consuming live execution capacity.

For long-horizon agents, lifecycle state and resource accounting must be explicit. Prompt-level behavior cannot reliably fix a stale runtime registry. The runtime needs clear transitions between running, completed, released, restorable, and closed states.
