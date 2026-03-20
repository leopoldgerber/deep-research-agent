# Workflow

## Overview

The system is implemented as a **stateful workflow-agent** with the main
states:
`task_parsing → planning → execution → review → replanning → synthesis → final_output`.

The agent: - normalizes the user request; - builds a research plan; -
executes steps via tools; - evaluates coverage and quality; - replans if
needed; - produces a final or partial result.

Retries are handled as an **internal mechanism of execution**, not a
separate state.

------------------------------------------------------------------------

## Core Workflow

### Main flow

    INPUT → TASK_PARSING → PLANNING → EXECUTION → REVIEW → SYNTHESIS → FINAL_OUTPUT

### With replanning loop

    INPUT → TASK_PARSING → PLANNING → EXECUTION → REVIEW
          → REPLANNING → EXECUTION → REVIEW → SYNTHESIS → FINAL_OUTPUT

The system supports **partial results** if full completion is not
possible.

------------------------------------------------------------------------

## Workflow States

### task_parsing

-   Normalize request
-   Extract goal, constraints, output format

Transitions: - valid → planning - invalid → final_output

------------------------------------------------------------------------

### planning

-   Create step-by-step plan
-   Define execution strategy

Transitions: - success → execution - failure → final_output

------------------------------------------------------------------------

### execution

-   Execute plan steps
-   Call tools
-   Collect results

Transitions: - steps completed → review - recoverable error → retry
(internal) - failure → review

------------------------------------------------------------------------

### review

-   Evaluate coverage
-   Detect gaps
-   Check quality

Transitions: - sufficient → synthesis - gaps → replanning - limits
reached → synthesis (partial) - stop → final_output

------------------------------------------------------------------------

### replanning

-   Update plan
-   Add/remove steps
-   Change strategy

Transitions: - success → execution - limit exceeded → synthesis
(partial) - failure → final_output

------------------------------------------------------------------------

### synthesis

-   Combine results
-   Build final structure
-   Add conclusions and confidence

Transition: - always → final_output

------------------------------------------------------------------------

### final_output

-   Return result (complete / partial / failed)
-   Terminal state

------------------------------------------------------------------------

## Completion Conditions

The workflow finishes when:

-   required steps are completed;
-   coverage is sufficient;
-   enough evidence is collected;
-   iteration / retry / replanning limits are reached;
-   further actions do not improve the result.

------------------------------------------------------------------------

## Error Handling

### Recoverable

-   timeout
-   temporary API failure
-   weak or empty result

### Non-recoverable

-   invalid request
-   unavailable tool
-   impossible step
-   system limits exceeded

------------------------------------------------------------------------

## Retry Policy

-   max_retry_per_step = 2
-   only for recoverable errors
-   retry must change strategy

Possible changes: - query - tool - step scope - execution order -
extraction format

If retries are exhausted: - mark step as failed/partial - move to review

------------------------------------------------------------------------

## Compact State Diagram

```text
INPUT
  ↓
TASK_PARSING
  ├─ invalid → FINAL_OUTPUT
  └─ ok → PLANNING
           ↓
        PLANNING
          ├─ fail → FINAL_OUTPUT
          └─ ok → EXECUTION
                   ↓
                EXECUTION
                  ├─ success → REVIEW
                  ├─ retry → EXECUTION
                  └─ fail → REVIEW
                           ↓
                        REVIEW
                  ├─ enough → SYNTHESIS
                  ├─ gaps → REPLANNING
                  ├─ limit → SYNTHESIS(partial)
                  └─ stop → FINAL_OUTPUT
                                ↓
                            REPLANNING
                  ├─ ok → EXECUTION
                  ├─ limit → SYNTHESIS(partial)
                  └─ fail → FINAL_OUTPUT
                                  ↓
                                SYNTHESIS
                                  ↓
                                FINAL_OUTPUT
```
