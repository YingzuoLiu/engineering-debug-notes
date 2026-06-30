# Nested Agent Tracing: Context Ownership and Record Linkage

## Background

This case came from a TruLens issue about nested app / sub-agent tracing.

In a multi-agent system, an outer agent may call an inner agent:

```text
Outer Agent
  └── Inner Agent
```

The problem was that tracing could lose this parent-child relationship. The inner app might look like a disconnected root record, or it might be silently folded into the outer record. Both make debugging, replay, dashboard display, and evaluation harder.

The goal of the PR was to make this relationship explicit in the trace.

## Core idea

This is a context ownership problem.

It is similar to memory attribution. A memory system should not only store what was remembered; it should also know where it came from, which agent wrote it, and under which task context it was created.

Tracing has the same issue. It is not enough to record that an inner app ran. The trace also needs to know which outer app called it and whether the inner app still has its own identity.

Desired shape:

```text
RECORD_ROOT
  └── NESTED_RECORD_ROOT
```

The nested root should keep its own `record_id` and `app_id`, while also linking back to the outer app using explicit parent fields.

## Scope

The original issue could include trace semantics, dashboard rendering, replay behavior, and evaluation attribution. I scoped the PR to the first layer only:

> establish explicit trace-shape semantics and parent linkage.

That is why the PR was marked as `Part of #2440` instead of `Closes #2440`.

## What changed

The PR added a new span type:

```python
NESTED_RECORD_ROOT = "nested_record_root"
```

It also added parent linkage fields:

```text
ai.observability.nested_record_root.parent_span_id
ai.observability.nested_record_root.parent_app_id
```

When an inner app starts under an active outer recording context, the inner root now:

- starts its own `record_id`
- keeps its own app identity
- is marked as `NESTED_RECORD_ROOT`
- links back to the active outer span and app

The recording context logic was updated so that nested apps can establish their own app identity instead of accidentally inheriting the outer app identity.

## Negative cases

The negative cases were important.

A standalone app should still produce a normal `RECORD_ROOT`.

If there is an active recording but no valid parent span can be resolved, the system falls back to a normal `RECORD_ROOT` instead of inventing a fake nested relationship.

Baggage-only linkage is not treated as authoritative. Having a `record_id` or `app_id` in baggage is not enough to prove that a nested relationship exists.

## Tests

A focused test file was added:

```text
tests/unit/test_otel_nested_record_root.py
```

The tests cover:

1. outer app starts a normal `RECORD_ROOT`
2. inner app under active outer recording becomes `NESTED_RECORD_ROOT`
3. inner app keeps a distinct `record_id`
4. inner app keeps a distinct `app_id`
5. nested root links back to the outer active span and app
6. standalone app remains normal `RECORD_ROOT`
7. baggage-only parent linkage is not treated as authoritative
8. unresolved parent span falls back to normal `RECORD_ROOT`

Local result:

```bash
pytest tests/unit/test_otel_nested_record_root.py tests/unit/test_otel_record_root.py -q
# 7 passed
```

## Debugging notes

One useful lesson was about editable install pollution. When using a separate worktree for `upstream/main`, Python was still importing code from the feature branch because of a previous editable install.

Before trusting cross-branch test results, check the import path:

```bash
python -c "import trulens; print(trulens.__path__)"
```

Another issue was an existing order-dependent test failure in `test_otel_span_processor.py`. It first looked like a possible regression, but the same failure reproduced on clean `upstream/main`, so it was not introduced by this PR.

The first CI run also failed because pre-commit reformatted several files. After running pre-commit locally and committing the formatting changes, all checks passed.

## Final result

PR: `Add nested record root span linkage for nested TruApp calls`

Result at the time of writing:

```text
All checks passed
Waiting for maintainer review
```

## Takeaways

- Nested agent tracing is fundamentally a context ownership problem.
- Inner agent runs need their own identity, not just inherited outer context.
- Parent-child linkage should be explicit, not inferred from timestamps or names.
- Negative cases matter because context/baggage can easily create false linkage.
- Worktree testing is only reliable if editable installs and import paths are clean.
- CI failures are not always logic failures; formatting hooks can fail after tests pass.
