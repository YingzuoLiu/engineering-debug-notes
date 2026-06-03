# LangGraph API OpenAPI Validator Drift

## 1. Problem

This note studies a LangGraph API validation issue where a request is rejected before it reaches the runtime, even though the runtime-visible type contract appears to allow the requested value.

The user-visible symptom is:

```text
stream_mode=["tools"]        -> rejected by request validation
stream_mode=["lifecycle"]    -> rejected by request validation
stream_mode=["values"]       -> accepted
```

This is surprising because the installed package exposes `StreamMode` as including both `"tools"` and `"lifecycle"`, and the streaming runtime treats them as public stream mode literals.

The failure happens at the API request validation layer. The run request is rejected with a 422-style validation failure before the request can reach the streaming runtime path that knows how to handle these modes.

## 2. Why It Matters

This is not just a small enum bug. It points to a broader production agent-platform risk:

> A validator can become stricter than the runtime contract it is supposed to protect.

In agent systems, validators are usually added to improve safety and correctness. But when the validator schema drifts away from the runtime type contract, the validator can block valid workflows.

This is especially important for self-hosted agent runtimes and SDK-driven streaming UX. A frontend or SDK may register a stream mode automatically, expecting the server runtime to support it. If the request validator is stale, the workflow fails before the runtime has a chance to apply its own compatibility logic.

## 3. Relevant Runtime Layer

The relevant layer is the API contract boundary between request validation and streaming runtime execution.

A simplified flow is:

```text
client / SDK
  -> run-create request
  -> request-body validator
  -> queued run / runtime model
  -> streaming runtime
  -> event stream consumer
```

The issue appears before runtime execution:

```text
request payload
  -> bundled OpenAPI validator schema
  -> validation failure
  -> runtime never sees the request
```

This makes it different from a normal streaming-runtime bug. The runtime may already know how to treat the mode, but the API boundary blocks the request first.

## 4. Root-Cause Hypothesis

The likely cause is schema-generation or release-artifact drift.

The installed package contains multiple sources that should agree:

```text
schema.StreamMode
  includes "tools" and "lifecycle"

stream.py
  treats "tools" and "lifecycle" as public StreamMode literals

validation.py
  builds request validators from a bundled top-level openapi.json

openapi.json
  stream_mode enum omits "tools" and "lifecycle"
```

The local reproduction showed that `langgraph_api.validation` loads the validator schema from:

```text
pathlib.Path(__file__).parent.parent / "openapi.json"
```

Inside the installed wheel, that resolves to:

```text
site-packages/openapi.json
```

not:

```text
site-packages/langgraph_api/openapi.json
```

The relevant `RunCreateStateful.properties.stream_mode` enum in the bundled OpenAPI schema accepted only:

```text
values, messages, messages-tuple, tasks, checkpoints,
updates, events, debug, custom
```

and omitted:

```text
tools, lifecycle
```

This suggests the OpenAPI artifact was generated from an older or incomplete stream mode definition, while the runtime package had already moved forward.

## 5. Fix Direction

The minimal fix direction is to regenerate or update the bundled OpenAPI schema so the request validator agrees with `schema.StreamMode`.

Conceptually:

```text
for each run-create schema that exposes stream_mode:
    ensure stream_mode enum includes "tools" and "lifecycle"
```

The local proof-of-fix was to patch the installed `openapi.json` by adding `"tools"` and `"lifecycle"` to the relevant `stream_mode` enum branches. After that patch, the same validator accepted:

```text
["values"]                    -> True
["tools"]                     -> True
["lifecycle"]                 -> True
["values", "messages", "tools"] -> True
```

The safer production fix should not hand-edit installed package files. It should update the source of the generated OpenAPI artifact or the OpenAPI generation pipeline, then add a regression test that validates the generated schema against `StreamMode`.

## 6. Regression Tests

Useful regression tests should check schema/runtime contract consistency.

### Test 1: validator accepts all public StreamMode values

```text
Given: StreamMode includes "tools" and "lifecycle"
When: a RunCreateStateful request uses stream_mode=["tools"] or ["lifecycle"]
Then: request-body validation should pass
```

### Test 2: validator accepts mixed stream modes

```text
Given: a request uses stream_mode=["values", "messages", "tools"]
When: the request is validated
Then: the validator should accept the payload
```

### Test 3: generated OpenAPI enum stays in sync with StreamMode

```text
Given: schema.StreamMode defines the public stream mode literals
When: the OpenAPI schema is generated or loaded
Then: every public StreamMode literal should appear in the run-create stream_mode enum
```

This test catches the broader class of drift, not only this specific pair of values.

## 7. Takeaway

The main engineering principle is:

> Validation logic should enforce the current runtime contract, not an outdated snapshot of it.

For production agent systems, validators are part of the runtime control plane. They are not passive documentation. If the validator is generated from a stale schema, the system can reject valid agent workflows before execution begins.

A robust agent platform needs contract tests across the boundary between SDK expectations, API schemas, request validators, and runtime behavior.