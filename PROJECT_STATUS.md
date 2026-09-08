# BioHub Cell Tracking — Project Status

**Status:** Active research prototype  
**Last updated:** September 7, 2026  
**Competition:** BioHub Cell Tracking During Development  
**Current notebook frontier:** Chapter 48 — Division-Aware Tracking at Assignment Time

## Project Objective

Build an end-to-end system for detecting cells in 3D microscopy volumes, linking detections across time, identifying cell divisions, and constructing biologically plausible lineage graphs.

The project has evolved experimentally rather than as a single fixed model. The notebook sequence records the causal progression from detection → ranking → tracking → refinement → division modeling → failure diagnosis → multi-sample audit → post-hoc topology correction → division-aware assignment.

## Current Technical State

The project has a working multi-sample reproduction harness for the historical Chapter 34 → 35 → 36 → 37 pipeline, a quantitative 13-event division audit, and a completed Chapter 47 topology experiment.

### Workstream status

| Workstream | Status | Current assessment |
| --- | --- | --- |
| 3D candidate detection | 🟢 Working prototype | Chapter 34 produced adequate parent and daughter candidates for all 13 audited GT divisions. |
| Candidate ranking | 🟢 Working prototype | Historical HistGradientBoosting ranking remains the selected Ch34 path. |
| Track-aware filtering | 🟡 Imperfect | Filtering caused 3/13 primary division failures and preferentially removes some daughters. |
| Motion-aware tracking | 🟡 Strong for continuation | Works across multiple samples, but continuation logic is structurally mismatched to division. |
| Track refinement | 🟡 Working prototype | Ch37 reproduced successfully across the V12 mini-batch. |
| Post-hoc division topology | 🔴 Rejected as primary path | Chapter 47 generated too many plausible splits from finished Ch37 geometry. |
| Division-aware assignment | 🔴 Current frontier | Needs explicit one-parent→two-daughter hypotheses during temporal association. |
| Autonomous lineage inference | 🔴 Not complete | Correct topology is representable but not yet inferred selectively without GT. |
| Competition-ready inference | 🟡 Not final | Needs topology integration, scoring/validation, and leaderboard iteration. |

## Chapter 46 — Multi-Sample Audit

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

The original single-event diagnosis from Chapters 42–45 made filtering appear to be the central division problem. The multi-sample evidence changed that conclusion.

Filtering still matters, but the dominant failure is **tracking topology**.

The tracker is designed to solve continuation: one detection becomes one next detection. A cell division requires a different structural operation: one parent must terminate and two new daughter branches must begin.

This explains why restoring daughter detections alone does not solve lineage construction.

## Chapter 47 — Post-Hoc Division-Aware Track Birth and Splitting

**Status:** Complete — negative/informative result.

Executed notebook commit: `5f58e6bc13964914a48228a7833eb49bb0908173`

Full experiment specification and result: [`chapter-47-division-aware-track-birth-and-splitting.md`](chapter-47-division-aware-track-birth-and-splitting.md)

### Research question

> Can the current refined tracking output be transformed into biologically plausible division topology by detecting likely branch events and creating new daughter tracks, without using ground truth to decide where to split?

### Experimental rule

Keep upstream V12 artifacts fixed and use only Chapter 37 track geometry for the split proposal. Do not tune the split rule against the 13 GT events.

### Chapter 47 V4 result

The notebook completed successfully and produced a valid transformed topology artifact set.

Measured structural outputs:

| Measure | Result |
| --- | ---: |
| Samples | 3 |
| Chapter 37 tracks | 1,347 |
| Label-free split candidates | 5,375 |
| Selected divisions | 75 |
| Lineage edges | 150 |
| Nodes moved into daughter tracks | 1,111 |
| Continuation edges rebuilt | 12,473 |

The selection cap was **25 divisions per sample**, and all three samples reached that cap, producing 75 selected divisions overall.

The same three samples contain **13 known binary GT divisions** in total. Therefore the post-hoc geometry heuristic failed the false-positive sanity gate: it identifies branch-like geometry far more often than true division events occur.

### Chapter 47 conclusion

**Post-hoc division inference from finished Chapter 37 geometry alone is not selective enough.**

This is not an implementation failure. The topology transform executed correctly and produced reversible split-node, continuation-edge, lineage-edge, and transform-log artifacts. The negative result is architectural: the finished trajectories have already collapsed information that is useful for deciding whether a parent should continue or divide.

Do not tune the Chapter 47 threshold against the same 13 GT events. The next experiment should move the division decision earlier into temporal assignment.

## Chapter 48 — Division-Aware Tracking at Assignment Time

**Status:** Next experiment.

### Research question

> Can temporal assignment distinguish an ordinary one-to-one continuation from a one-parent→two-daughter division hypothesis using evidence available before tracks are finalized?

### Why this follows from Chapter 47

Chapter 47 showed that many ordinary track configurations look branch-like after the fact. Chapter 48 should instead use evidence that exists at assignment time but is weakened or lost after refinement:

- competing detections in the next frame
- parent velocity and motion prediction
- parent→candidate assignment costs
- simultaneous plausibility of two daughters
- daughter separation
- new-track birth timing
- short temporal offsets around the division frame
- evidence that assigning only one daughter would force the other into an implausible continuation or unrelated birth

### Design principle

The tracker should compare at least two local hypotheses:

1. **Continuation:** one parent → one child detection
2. **Division:** one parent terminates → two daughter detections begin

The goal is not merely to add more branches. The goal is to make division a competing assignment hypothesis with an explicit cost/score so false-positive branches remain controlled.

### Initial success criteria

A Chapter 48 experiment should:

1. remain label-free during assignment decisions;
2. reduce the Chapter 46 topology failure modes without exploding predicted division count;
3. explicitly report false-positive burden;
4. preserve ordinary continuation performance;
5. evaluate the same 13 GT divisions first, then expand only if the rule shows signal.

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

### Phase 10 — Post-hoc topology correction

**Chapter 47**

Chapter 47 tested whether finished Chapter 37 trajectories contain enough geometry to infer division branches autonomously. V4 generated 5,375 candidate split events and selected 75, hitting the 25-per-sample cap on all three samples.

**Milestone:** the project ruled out the simplest post-hoc topology correction as insufficiently selective.

### Phase 11 — Division-aware assignment

**Chapter 48 — next**

Move division reasoning into temporal assignment, where parent motion, competing detections, and assignment costs are still available.

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
- Label-free post-hoc division candidate generation
- Reversible topology transformation
- Negative-result diagnosis of post-hoc overgeneration

### Not yet demonstrated

- Selective division-aware assignment during tracking
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

Design Chapter 48 as a controlled assignment-time experiment. Reuse the same V12 sample set and the Chapter 46 audit baseline, but expose assignment-time evidence before Chapter 37 refinement. Test whether an explicit continuation-vs-division hypothesis can reduce topology failures while keeping predicted division count bounded.
