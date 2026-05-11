# Evaluation Suite

The eval-suite directory provides the schemas and tooling patterns for running structured evaluations against agent golden sets. It is a framework, not a collection of agent-specific artefacts: the golden sets themselves are not published here, the judge rubrics are not tuned to any specific agent's domain, and the eval runner is provided as illustrative pseudo-code rather than a runnable implementation. What this directory provides is the contracts against which golden sets are built and executed, the scoring rubric structure that an LLM judge uses to evaluate responses, and the structural pattern for a CI-compatible eval runner.

## Purpose

The eval-suite exists to make the evaluation contract explicit and portable. Any agent in the programme can be evaluated against these schemas without custom scaffolding. The golden-set schema defines what a test case must contain; the judge rubric defines what a score means; the eval runner pseudo-code defines how those two artefacts are combined in a CI pipeline. Teams building new agents adopt these schemas from day one rather than inventing their own evaluation structures, which would produce incompatible formats and make portfolio-level comparison impossible.

## Contents

| File | Purpose |
|---|---|
| golden-set-schema.md | YAML schema for a single golden test case |
| judge-rubric.md | Rubric for scoring each eval axis (0–4 scale) |
| eval-runner-pseudocode.md | Illustrative pseudo-code for a CI-compatible eval runner |

## How to Use in CI

The eval runner loads the golden set from the path specified by the CI pipeline configuration, invokes the agent under test for each case, scores each response with an LLM judge using the rubric defined in `judge-rubric.md`, aggregates scores per axis across all cases, and compares the aggregated scores against the thresholds registered in the AI System Register for the target agent. The runner exits with code 0 when all axes pass their registered thresholds and code 1 when any axis fails. This exit code convention makes the eval runner composable with any CI system — GitHub Actions, GitLab CI, Jenkins, or any pipeline that treats a non-zero exit code as a blocking failure. The runner emits a structured JSON report at a configurable output path; this report is the artefact consumed by the deployment gate.

## What Is Not Here

The curated golden sets, the domain-tuned judge rubrics, and the runnable eval runner implementation are not published in this repository. Curated golden sets require two to four weeks of expert annotation per agent; they contain agent-specific input distributions that are not generalisable. Tuned judge rubrics are iterated artefacts — the rubric text, the judge model selection, and the scoring calibration are as important as the rubric structure, and they require validation against human-scored ground truth before they can be trusted. The runnable implementation ties together a golden-set runner, an LLM-as-judge scaffold, online sampling middleware, and CI integration. These are engagement deliverables. This directory contains the schemas and structural patterns that make that engagement faster and more consistent.
