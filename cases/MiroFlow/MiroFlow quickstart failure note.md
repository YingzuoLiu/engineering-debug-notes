# MiroFlow Quickstart Failure Note: Tool Success Does Not Guarantee Answer Correctness

## 1. Background

I ran the MiroFlow quickstart reading task locally to understand how MiroMind’s agent runtime handles tool orchestration, trace logging, and final answer generation.

The task was simple:

> Find the first country listed in the XLSX file whose name starts with “Co”.

The associated file was:

```text
data/FSI-2023-DOWNLOAD.xlsx
```

The goal was not only to get the answer, but to inspect the full agent execution path:

```text
User task → LLM planning → tool call → file reading → evidence extraction → final answer → trace log
```

## 2. Environment and Setup

The experiment was run in a Windows + WSL2 environment.

Runtime setup:

```text
OS: Windows with WSL2 / Ubuntu
Project: MiroMindAI/MiroFlow
Python: 3.12 via uv
Package manager: uv
LLM provider: OpenRouter
Model used: deepseek/deepseek-chat
Tool enabled: tool-reading / read_file
```

The default MiroFlow quickstart configuration used Claude 3.7 Sonnet via OpenRouter, but that route failed with an endpoint error. I later switched to `DeepSeekOpenRouterClient` with `deepseek/deepseek-chat`, using OpenRouter as the model gateway.

Before running MiroFlow, I verified the OpenRouter key independently with a direct `curl` request. The key test returned HTTP 200, confirming that authentication and credits were working.

## 3. What Worked

The MiroFlow execution successfully completed the agent pipeline.

The trace showed that:

```text
tool-reading server was loaded
read_file tool was called successfully
main agent completed 2 turns
main loop completed
final summary generation succeeded
task status was completed
final boxed answer was extracted
```

The runtime therefore worked at the orchestration level. The agent was able to:

1. Receive the task.
2. Plan to read the Excel file.
3. Call the file-reading tool.
4. Receive the tool result.
5. Produce a final answer.
6. Save execution logs and trace information.

This confirms that the basic MiroFlow quickstart chain was successfully reproduced locally.

## 4. Expected Answer from Direct File Verification

To verify the answer independently, I loaded the XLSX file with pandas and filtered the `Country` column for entries starting with `Co`.

The first matches were:

```text
Country                       Rank
Congo Democratic Republic      4th
Congo Republic                28th
Cote d'Ivoire                 36th
Comoros                       45th
Colombia                      59th
Costa Rica                   150th
```

Therefore, if “first listed” means the first occurrence in the file order, the correct answer should be:

```text
Congo Democratic Republic
```

## 5. MiroFlow Final Output

However, the MiroFlow run produced:

```text
status: completed
final_boxed_answer: Colombia
```

The final answer was:

```text
\boxed{Colombia}
```

This is incorrect under the natural interpretation of the task, because `Congo Democratic Republic` appears before `Colombia` in the Excel file.

## 6. Failure Analysis

This is not a tool failure.

The file-reading tool successfully returned the spreadsheet content, and the returned content included the correct earlier entry:

```text
Congo Democratic Republic | 2023 | 4th
```

It also included:

```text
Colombia | 2023 | 59th
```

The failure happened after tool execution, during the model’s reasoning or final answer extraction phase.

In other words:

```text
Tool access succeeded.
The correct evidence was present.
The runtime marked the task as completed.
The final answer was still wrong.
```

This makes the failure more interesting than a simple environment or API issue.

The core issue is an evidence-to-answer consistency failure:

```text
The agent retrieved the right evidence but selected the wrong answer.
```

## 7. Why This Matters for Agent Runtime Design

This case shows that successful tool execution is not enough to guarantee task correctness.

For long-horizon agents, a completed trajectory can still contain a silent correctness failure. The system may report success because:

```text
the tool call succeeded,
the model produced a valid-looking answer,
the final answer matched the expected output format,
and the runtime reached completed status.
```

But none of these guarantees that the answer is actually grounded in the tool evidence.

This is especially important for deep research agents, where the final answer often depends on long traces, multiple tool calls, web search results, document reading, and intermediate reasoning steps.

A robust agent runtime should not only ask:

```text
Did the agent finish?
Did it call the tool?
Did it produce a boxed answer?
```

It should also ask:

```text
Does the final answer match the evidence returned by the tools?
Was the answer selected from the correct part of the evidence?
Did the agent skip an earlier valid candidate?
Can the final answer be traced back to a specific evidence span?
```

## 8. Connection to My Own Agent Runtime Work

This failure is similar to an issue I observed in my own travel-agent-runtime-demo project.

In that project, removing the Validator increased the apparent completion rate, but introduced silent constraint violations. The system looked more successful on the surface, but the completed plans could violate hard constraints such as budget, dates, or travel preferences.

The MiroFlow case has the same pattern:

```text
Surface-level completion looked successful,
but the final answer violated the evidence.
```

In both cases, completion rate alone is not a reliable correctness metric.

The deeper lesson is:

```text
Agent correctness requires validation beyond final response formatting.
```

For travel planning, this means validating constraints such as budget and schedule.

For research or file-reading agents, this means validating that the final answer is consistent with retrieved evidence.

## 9. Possible Improvement Direction

A lightweight improvement would be to add an evidence-to-answer verification step before finalizing the output.

For this specific task, the verifier could check:

```text
1. Extract all candidate country names starting with "Co" from the tool result.
2. Preserve their original order from the table.
3. Compare the final answer against the first extracted candidate.
4. If the final answer differs, flag the result as inconsistent or trigger a repair step.
```

A more general verifier could work as follows:

```text
Tool result → candidate extraction → answer grounding check → final answer validation
```

For tabular tasks, this can be partially deterministic.

For open-ended research tasks, the same idea could be implemented through evidence cards, citation spans, or process-level judge models.

