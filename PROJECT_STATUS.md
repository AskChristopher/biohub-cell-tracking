# BioHub Cell Tracking — Project Status

**Status:** Active research prototype  
**Last updated:** September 6, 2026  
**Competition:** BioHub Cell Tracking During Development  
**Current notebook frontier:** Chapter 47 — Division-Aware Track Birth and Splitting

## Project Objective

Build an end-to-end system for detecting cells in 3D microscopy volumes, linking detections across time, identifying cell divisions, and constructing biologically plausible lineage graphs.

The project has evolved experimentally rather than as a single fixed model. The notebook sequence records the causal progression from detection → ranking → tracking → refinement → division modeling → failure diagnosis → multi-sample audit → topology correction.

## Current Technical State

The project now has a working multi-sample reproduction harness for the historical Chapter 34 → 35 → 36 → 37 pipeline and a quantitative 13-event division audit.

### Workstream status

| Workstream | Status | Current assessment |
| --- | --- | --- |
| 3D candidate detection | 🟢 Working prototype | Chapter 34 produced adequate parent and daughter candidates for all 13 audited GT divisions. |
| Candidate ranking | 🟢 Working prototype | Historical HistGradientBoosting ranking remains the selected Ch34 path. |
| Track-aware filtering | 🟡 Imperfect | Filtering caused 3/13 primary division failures and preferentially removes some daughters. |
| Motion-aware tracking | 🟡 Strong for continuation | Works across multiple samples, but continuation logic is structurally mismatched to division. |
| Track refinement | 🟡 Working prototype | Ch37 reproduced successfully across the V12 mini-batch. |
| Division topology | 🔴 Main bottleneck | 8/13 primary failures are tracking/topology failures. |
| Autonomous lineage inference | 🔴 Not complete | Correct topology is representable but not yet inferred reliably without GT. |
| Competition-ready inference | 🟡 Not final | Needs topology integration, scoring/validation, and leaderboard iteration. |

## Most Important Result — Chapter 46 Multi-Sample Audit

Chapter 46A V12 reproduced the historical Ch34→35→36→37 pipeline on three independent samples with no stage failures.

Samples and GT binary divisions:

| Sample | GT binary divisions | Ch34 rows | Ch36 rows | Ch37 nodes | Ch37 tracks |
| --- | ---: | ---: | ---: | ---: | ---: |
| `6bba_48816121` | 5 | 20,000 | 5,000 | 4,574 | 504 |
| `6bba_09961292` | 4 | 20,000 | 5,000 | 4,596 | 408 |
| `6bba_afb141ff` | 4 | 20,000 | 5,000 | 4,722 | 435 |

Total audited GT binary divisions: **13**.

### Candidate survival

| Entity | Ch34 adequate | Ch36 adequate | Ch37 adequate |
| --- | ---: | ---: | ---: |
| Parent | 100.0% | 100.0% | 92.3% |
| Daughter A | 100.0% | 92.3% | 84.6% |
| Daughter B | 100.0% | 76.9% | 76.9% |

### Primary failure distribution

| Failure | Events | Percent |
| --- | ---: | ---: |
| Daughters share one track | 5 | 38.5% |
| Daughter attached to parent track | 3 | 23.1% |
| One daughter filtered | 2 | 15.4% |
| Both daughters filtered | 1 | 7.7% |
| Topology compatible | 2 | 15.4% |

Grouped by layer:

- **Detection/ranking:** 0 / 13 primary failures
- **Filtering:** 3 / 13 primary failures (23.1%)
- **Tracking/topology:** 8 / 13 primary failures (61.5%)
- **Topology-compatible primary outcome:** 2 / 13 (15.4%)

Additional topology diagnostics:

- either daughter temporally displaced: 11 / 13
- daughters share a track: 7 / 13 raw topology cases
- at least one daughter attached to parent track: 10 / 13 raw topology cases
- raw topology-compatible: 3 / 13

## What Changed Because of Chapter 46

The original single-event diagnosis from Chapters 42–45 made filtering appear to be the central division problem. The multi-sample evidence changes that conclusion.

Filtering still matters, but the dominant failure is now clearly **tracking topology**.

The tracker is designed to solve continuation: one detection becomes one next detection. A cell division requires a different structural operation: one parent must terminate and two new daughter branches must begin.

This explains why restoring daughter detections alone does not solve lineage construction.

## Reconstructed Research History

### Phase 1 — Data and ground-truth understanding

**Chapters 1–4**

Established microscopy visualization, track inspection, and division understanding. Chapter 3 preserved the known diagnostic division later used in Chapters 42–45.

### Phase 2 — Baseline detection and learned filtering

**Chapters 5–13**

Moved from hand-tuned LoG-style detection toward learned candidate scoring and ranking.

### Phase 3 — Candidate coverage and 3D ranking

**Chapters 14–26**

Separated proposal recall, NMS/consolidation, binary classification, distance ranking, and 3D feature engineering.

### Phase 4 — Temporal association

**Chapters 27–29**

Introduced temporal linking and motion-aware global assignment.

### Phase 5 — Stronger multi-scale detection/ranking

**Chapters 30–34**

Built the upstream detection pipeline used by all later tracking/division experiments. Chapter 32 showed the HistGradientBoosting ranking path was strong at top-200 candidate selection.

### Phase 6 — Motion-aware tracking and refinement

**Chapters 35–37**

Added Hungarian assignment, velocity prediction, gap closing, track-aware filtering, and refinement.

### Phase 7 — Division and lineage prototype

**Chapters 38–41**

Generated division candidates and lineage graphs, then exposed insufficient positive division examples for meaningful supervised learning in the original sample.

### Phase 8 — Single-event division failure diagnosis

**Chapters 42–45**

- Ch42 traced a known GT division through the pipeline.
- Ch43 restored likely daughter detections before tracking.
- Ch44 showed that rescued daughters could still be absorbed into continuation trajectories.
- Ch45 used GT-seeded splitting to prove the graph can represent correct parent→daughter topology.

### Phase 9 — Multi-sample reproduction and quantitative audit

**Chapter 46A + Chapter 46**

Chapter 46A V6–V11 progressively fixed orchestration problems without changing historical algorithms. V11 achieved the first complete one-sample Ch34→37 reproduction.

V12 expanded that to three samples and passed 3/3 end-to-end with no stage failures.

Chapter 46 V4 then audited 13 binary GT divisions and identified tracking/topology as the dominant measured bottleneck.

**Milestone:** the project moved from a qualitative one-event diagnosis to a quantitative multi-sample causal result.

## Chapter 47 — Division-Aware Track Birth and Splitting

**Status:** Planned / ready to implement.

Full plan: [`chapter-47-division-aware-track-birth-and-splitting.md`](chapter-47-division-aware-track-birth-and-splitting.md)

### Research question

> Can the current refined tracking output be transformed into biologically plausible division topology by detecting likely branch events and creating new daughter tracks, without using ground truth to decide where to split?

### Experimental rule

Keep Chapters 34, 36, and 37 fixed. Change only the topology layer so the result is causally interpretable.

GT may be used only for evaluation after label-free split decisions are made.

### Chapter 47 baseline

The same 13 GT events from Chapter 46 define the initial evaluation set:

- 8 tracking/topology primary failures
- 3 filtering primary failures
- 2 topology-compatible primary outcomes

### Chapter 47 success criteria

A useful result should:

1. avoid GT leakage in split selection;
2. materially reduce `DAUGHTERS_SHARE_TRACK` and `DAUGHTER_ATTACHED_TO_PARENT_TRACK`;
3. increase topology-compatible outcomes above the Chapter 46 baseline;
4. measure false-positive inferred divisions;
5. preserve original Ch37 artifacts and write transformed outputs separately.

## Files to Preserve for Chapter 47

### V12 combined artifacts

- `chapter34_top200_detections_multisample.csv`
- `chapter36_filtered_detections_multisample.csv`
- `chapter37_refined_track_nodes_multisample.csv`
- `chapter37_refined_track_edges_multisample.csv`
- `chapter37_refined_track_summary_multisample.csv`

### Chapter 46 audit artifacts

- `chapter46_multisample_division_audit.csv`
- `chapter46_failure_distribution.csv`
- `chapter46_topology_summary.csv`
- `chapter46_ch47_decision_table.csv`

### Optional diagnostic artifacts

- `chapter36_filtered_track_nodes_multisample.csv`
- `chapter36_filtered_track_edges_multisample.csv`
- `chapter35_track_nodes_multisample.csv`
- `chapter35_track_edges_multisample.csv`
- `chapter46a_selected_samples.csv`
- `chapter46a_validation.csv`

The original BioHub competition dataset must also remain available in Kaggle for GEFF GT evaluation.

## Project Maturity

### Demonstrated

- Ground-truth and volume visualization
- 3D candidate generation
- Candidate coverage analysis
- Learned candidate ranking
- 3D feature engineering
- Motion-aware assignment
- Gap closing
- Track-aware filtering
- Track refinement
- Division candidate generation
- Lineage construction
- GT-seeded division examples
- Upstream failure diagnosis
- Daughter rescue
- GT-assisted topology splitting
- Multi-sample pipeline reproduction
- Multi-sample division failure audit

### Not yet demonstrated

- Robust label-free division-aware track birth/splitting
- False-positive-controlled autonomous lineage inference
- Competition-ready end-to-end topology integration
- Final leaderboard-validated solution

## Project Management Rule

**Kaggle is the laboratory. GitHub is the durable project record.**

Experiments should answer a specific unresolved research question and record:

- question/hypothesis
- method/change
- result/metrics
- failure/lesson
- next decision

Failed experiments remain part of the project history when they explain why the approach changed.

## Immediate Next Step

Prepare the Chapter 47 inputs, implement the label-free branch-candidate and topology-splitting notebook, and evaluate it against the same 13 GT divisions before expanding to a larger cohort.
