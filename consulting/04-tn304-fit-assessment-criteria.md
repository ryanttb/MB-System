# TN304 Dataset Fit Assessment Criteria

This document defines a repeatable, evidence-based process to decide whether TN304 data can satisfy Prometheus requirements.

Use this after required outputs are explicitly defined.

## 1) Assessment goal

Determine if TN304 data, processed through MB-System, can produce required outputs with acceptable quality, completeness, and effort.

## 2) Required inputs to start

- Final list of required outputs (fields, units, resolution, coordinate system, time base).
- Acceptable quality thresholds (accuracy, completeness, continuity, noise tolerance).
- Delivery constraints (turnaround time, reproducibility, compute/storage limits).

## 3) Fit dimensions and pass/fail rules

### A. Data availability fit

- **Question:** Are all required source fields present in TN304 raw data?
- **Evidence:** `mbinfo`, `mblist`, `mbnavlist`, `mbsvplist`, format metadata.
- **Pass rule:** 100% of mandatory fields exist, or approved substitution path exists.

### B. Processing feasibility fit

- **Question:** Can MB-System process TN304 data to required outputs without custom code?
- **Evidence:** successful run of inspect -> edit/configure -> `mbprocess` -> product generation.
- **Pass rule:** baseline workflow succeeds on representative lines with no blocking failures.

### C. Quality fit

- **Question:** Do produced outputs meet engineering/science quality thresholds?
- **Evidence:** QA metrics, artifact review, before/after comparisons, anomaly counts.
- **Pass rule:** all critical metrics meet threshold; non-critical exceptions documented and accepted.

### D. Operational fit

- **Question:** Is the workflow supportable by Prometheus engineers?
- **Evidence:** build reproducibility, runtime, operator complexity, failure recovery steps.
- **Pass rule:** at least two engineers can repeat workflow from docs with consistent outputs.

## 4) Recommended execution sequence

1. **Capability scan**
   - Identify format IDs and survey scope.
   - Build field inventory for representative files.
2. **Pilot processing run**
   - Run minimal MB-System workflow on a small TN304 subset.
   - Generate at least one derived product (e.g., grid).
3. **Field-level validation**
   - Validate required fields and transformations against acceptance criteria.
4. **Scale check**
   - Run on larger subset; collect runtime and failure modes.
5. **Decision review**
   - Summarize fit/gaps and recommendation: go, conditional-go, or no-go.

## 5) Decision matrix template

| Requirement | Mandatory? | TN304 source field present | MB-System path/tool | Quality status | Effort/risk | Decision |
|---|---|---|---|---|---|---|
| Example: bathymetry grid at target resolution | yes | yes/no | `mbprocess` + `mbgrid` | pass/fail | low/med/high | go/conditional/no-go |

## 6) Risk register template

| Risk | Likelihood | Impact | Mitigation | Owner | Status |
|---|---|---|---|---|---|
| Missing required field in vendor datagrams | medium | high | define alternate derivation or scope reduction | data lead | open |

## 7) Suggested deliverables for Prometheus review

- Completed decision matrix.
- Reproducible command log/scripts for pilot and scale runs.
- Sample outputs plus QA summary.
- Short recommendation memo with assumptions and unresolved risks.
