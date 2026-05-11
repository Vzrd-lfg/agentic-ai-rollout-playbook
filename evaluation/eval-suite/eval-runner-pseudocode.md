# Eval Runner (Pseudo-Code)

> **NOT_PRODUCTION — DEMONSTRATION ONLY.** This pseudo-code illustrates the structural intent of a CI-compatible eval runner. It cannot be executed as written.

## Overview

The eval runner is the component that closes the loop between the golden set, the agent under test, the judge rubric, and the thresholds recorded in the AI System Register. It loads a golden set, invokes the agent under test for each case, scores each response with an LLM judge using the rubric defined in `judge-rubric.md`, aggregates scores per axis across all cases, compares the aggregated scores against the thresholds from the AI System Register, and exits with code 0 (all axes pass) or code 1 (any axis fails). The JSON report emitted at the end of each run is the artefact consumed by the deployment gate and stored in the eval history for longitudinal tracking.

## Pseudo-Code

```python
# NOT_PRODUCTION — DEMONSTRATION ONLY
# Illustrative eval runner — not executable as written.

def run_eval(agent_id: str, golden_set_path: str) -> EvalReport:
    """
    Load golden set, run agent, judge responses, compare against thresholds.
    Returns EvalReport with pass/fail per axis and aggregate.
    """

    # --- 1. Load configuration ---
    thresholds = load_thresholds_from_register(agent_id)
    # thresholds: {faithfulness: 3.8, accuracy: 3.5, safety: 4.0, format: 3.8}

    golden_set = load_golden_set(golden_set_path)
    # golden_set: List[TestCase], each with input, expected, ground_truth

    results = []

    # --- 2. Run agent for each test case ---
    for test_case in golden_set:
        agent_response = invoke_agent(
            agent_id=agent_id,
            input=test_case.input,
            # Note: invoke_agent is a placeholder for the actual agent API call
        )

        # --- 3. Judge the response ---
        scores = judge_response(
            response=agent_response,
            test_case=test_case,
            rubric=load_rubric(),
            # judge_response calls an LLM with the rubric as the system prompt
            # Returns: {faithfulness: int, accuracy: int, safety: int, format: int}
        )

        # --- 3a. Safety critical failure check ---
        if scores.get("safety") == 0:
            # A single safety score of 0 is a critical failure.
            # The run is terminated immediately; deployment is blocked.
            emit_critical_failure_alert(agent_id, test_case.id, agent_response)
            return EvalReport(
                agent_id=agent_id,
                run_timestamp=now(),
                total_cases=len(golden_set),
                aggregated_scores={},
                thresholds=thresholds,
                pass_fail={"safety": False},
                all_pass=False,
                critical_failure=True,
                critical_failure_case_id=test_case.id,
            )

        results.append(EvalCaseResult(
            test_case_id=test_case.id,
            scores=scores,
            response=agent_response,
        ))

    # --- 4. Aggregate scores ---
    aggregated = aggregate_scores(results)
    # aggregated: {axis: mean_score} across all test cases

    # --- 5. Compare against thresholds ---
    pass_fail = {}
    for axis, threshold in thresholds.items():
        pass_fail[axis] = aggregated.get(axis, 0) >= threshold

    all_pass = all(pass_fail.values())

    # --- 6. Emit report ---
    report = EvalReport(
        agent_id=agent_id,
        run_timestamp=now(),
        total_cases=len(golden_set),
        aggregated_scores=aggregated,
        thresholds=thresholds,
        pass_fail=pass_fail,
        all_pass=all_pass,
        critical_failure=False,
        critical_failure_case_id=None,
    )
    write_report(report)  # writes JSON to configured output path

    # --- 7. Exit with CI-compatible code ---
    return report  # caller uses report.all_pass to set exit code 0 or 1


# --- Supporting type definitions (illustrative) ---

class EvalCaseResult:
    test_case_id: str
    scores: dict   # {axis: int}
    response: str

class EvalReport:
    agent_id: str
    run_timestamp: str
    total_cases: int
    aggregated_scores: dict   # {axis: float}
    thresholds: dict          # {axis: float}
    pass_fail: dict           # {axis: bool}
    all_pass: bool
    critical_failure: bool
    critical_failure_case_id: str | None
```

## Notes on the Design

**Fail-fast on safety.** The runner terminates immediately when a safety score of 0 is encountered on any case. A critical safety failure is not averaged away across the golden set; it blocks deployment unconditionally. All other axes are aggregated before comparison against threshold.

**Threshold source.** Thresholds are read from the AI System Register at runtime, not hardcoded in the runner. This means the same runner binary gates all agents in the programme, each against its own registered thresholds. A new agent's Approval Gate review produces a register entry; the runner picks it up automatically on the next CI run.

**Judge model pinning.** The `load_rubric()` call in step 3 loads both the rubric text and the judge model identifier from a versioned configuration file. The runner records the judge model version in the eval report, so historical scores can be attributed to a specific judge model version. Changing the judge model requires a re-baseline run against the existing golden set before the new judge is used for gating decisions.

**Report persistence.** The JSON report is written to a path that the CI system archives as a build artefact. This is the longitudinal record of eval history — the mechanism by which drift is detectable across multiple deployments without requiring access to the agent's production traffic.
