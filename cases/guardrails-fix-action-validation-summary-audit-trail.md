# Guardrails FIX Action Audit Trail in Validation Summary

## Context

Related issue: guardrails-ai/guardrails#1448  
Related PR: guardrails-ai/guardrails#1550 (pending review at time of writing)

This note summarizes a small observability improvement around Guardrails validation summaries.

## 1. Problem

Guardrails supports `on_fail="fix"`, where a validator can repair an invalid value and allow execution to continue.

This is useful when the fix is mechanical, such as trimming whitespace, normalizing a format, or replacing a value with a known safe value.

The confusing part is that the final `ValidationOutcome` may look clean, while the caller does not immediately see what value was changed.

In other words, the system did mutate the output, but the summary layer did not clearly show the value before validation and the value after validation.

This matters especially when a fix is not just formatting, but changes the meaning of the model output.

## 2. Why It Matters

In long-running AI systems, small silent changes can accumulate.

If an agent produces a value, a validator fixes it, and downstream logic only sees the fixed value, the system may lose sight of what the model actually produced.

This creates an auditability problem:

- The raw model output may still exist elsewhere.
- The validator logs may contain more detail.
- But the high-level validation summary does not directly expose the before/after values.

For application developers, this means they need to dig into lower-level logs to understand what happened.

A validation summary should not only say “this validator failed.” It should also help answer: “What value did the validator see, and what value did the system continue with?”

## 3. Relevant Runtime Layer

This issue lives in the validation observability layer.

The core validation path already has the information:

- `ValidatorLogs.value_before_validation`
- `ValidatorLogs.value_after_validation`
- `ValidationOutcome.validation_summaries`

The gap is not that Guardrails cannot track the data. The gap is that the data is not surfaced in the summary object that callers are likely to inspect.

So the fix should not change validation behavior. It should only enrich the summary produced from existing validator logs.

## 4. Root-Cause Hypothesis

The likely cause is a mismatch between low-level validator logs and high-level validation summaries.

`ValidatorLogs` already records the value before validation and the value after validation.

However, `ValidationSummary.from_validator_logs_only_fails(...)` only includes fields such as:

- validator name
- validation status
- property path
- failure reason
- error spans

It does not copy the before/after values into the summary.

As a result, the summary tells the caller that validation failed, but it does not show the exact mutation context.

## 5. Fix Direction

The smallest safe fix is to add two optional fields to `ValidationSummary`:

- `value_before_validation`
- `value_after_validation`

These fields are populated directly from the existing `ValidatorLogs` object.

This keeps the change small:

- No new `on_fail` tier
- No new `QUARANTINE` behavior
- No change to `FIX`, `REASK`, or `NOOP`
- No change to the core validation engine
- No new mutation model

The change only makes existing runtime information easier to inspect.

This is safer than adding a new action type because it improves observability without changing execution semantics.

## 6. Regression Test

A focused regression test can construct a `ValidatorLogs` object with:

- a failing validation result
- a value before validation
- a value after validation

Then it can generate validation summaries from that log and assert that:

- one summary is produced
- the validator name and failure reason are preserved
- `value_before_validation` is included
- `value_after_validation` is included
- alias serialization exposes `valueBeforeValidation` and `valueAfterValidation`

The test does not need to run a full Guard execution. The issue is in summary generation, so the test should stay close to that layer.

## 7. Takeaway

A validation system should not only enforce rules. It should also make its decisions inspectable.

For AI runtime systems, this is especially important because validators can change the output that downstream components receive.

When a runtime silently repairs an output, the repair itself becomes part of the execution history.

A good observability layer should preserve that history in the place developers are most likely to inspect.

The broader lesson is:

> If a runtime layer mutates state, the summary layer should expose enough information to understand that mutation.
