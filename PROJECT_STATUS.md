# BioHub Cell Tracking — Project Status

**Status:** Active research prototype  
**Last reconstructed:** September 2026  
**Competition:** BioHub Cell Tracking  
**Current notebook frontier:** Chapter 45 — Division-Aware Track Splitting

## Project Objective

Build an end-to-end system for detecting cells in 3D microscopy volumes, linking detections across time, identifying cell divisions, and constructing biologically plausible cell lineages.

The project has evolved experimentally rather than as a single fixed model. The notebook sequence records that evolution: data exploration → candidate detection → learned candidate ranking → temporal tracking → track refinement → division detection → lineage construction → diagnosis and rescue of division failures.

## Current Technical State

The project has a working prototype pipeline for:

1. generating candidate cell detections,
2. ranking/filtering candidates,
3. linking detections across time,
4. refining tracks and closing gaps,
5. generating parent/daughter division candidates,
6. constructing lineage topology,
7. diagnosing failures around known ground-truth divisions.

The central unresolved problem is **autonomous division inference**. The recent notebooks prove that correct division topology can be represented, but the system does not yet reliably discover and construct that topology without ground-truth assistance.

### Workstream status

| Workstream | Status | Current assessment |
| --- | --- | --- |
| 3D candidate detection | 🟡 Prototype works | Candidate generation is established, but true daughter cells can be weak or temporally displaced. |
| Candidate ranking/filtering | 🟡 Works but imperfect | Learned ranking improved the detection pipeline, but aggressive filtering can remove valid division daughters. |
| Temporal tracking | 🟡 Prototype works | Motion-aware assignment and gap closing handle ordinary continuation reasonably well. |
| Track refinement | 🟡 Prototype works | Weak fragments can be removed and broken trajectories reconnected. |
| Division candidate generation | 🟡 Prototype | Parent/daughter hypotheses can be generated, but reliable inference remains unresolved. |
| Division classification | 🔴 Blocked / insufficiently solved | Early supervised attempts exposed very limited positive division examples in the sample being used. |
| Lineage construction | 🟡 Proof of concept | Correct topology can be represented, including track splitting, but recent proof uses GT assistance. |
| Autonomous end-to-end lineage inference | 🔴 Not complete | This is now the major research frontier. |

## Reconstructed Research History

The repository is an experimental record rather than a clean linear software release history. Some historical notebooks failed to sync from Kaggle and were recovered separately; Chapter 2 is currently missing. The phases below therefore describe the recoverable research trajectory rather than claiming every chapter is complete or reproducible.

### Phase 1 — Understand the data and ground truth

**Chapters 1–4**

The project began with exploration of the competition data, 3D/4D microscopy volumes, cell tracks, and division events. Visualization tooling became important early because the problem cannot be understood from tabular metrics alone.

Recovered history includes:

- Chapter 1 — data exploration; archival execution includes a missing `zarr` dependency.
- Chapter 2 — missing from the recovered record.
- Chapter 3 — visualization of cell tracks and divisions.
- Chapter 4 — visualization tools.

**Milestone:** Established the data model and visual/ground-truth inspection workflow.

### Phase 2 — Establish a baseline detector

**Chapters 5–13**

The next phase moved from visual exploration to cell-center detection and learned filtering/ranking:

- Ch5 — baseline cell detection at a single timepoint.
- Ch6 — tuning and evaluating the baseline detector.
- Ch8 — learning what a true cell center looks like.
- Ch9 — candidate filtering for LoG detections.
- Ch10 — filtered detector from ranked candidates.
- Ch11 — testing across multiple timepoints.
- Ch12 — multi-timepoint candidate filter.
- Ch13 — distance-to-ground-truth ranking model.

The repository does not currently contain a Chapter 7 notebook.

**Milestone:** Detection changed from a hand-tuned candidate generator into a learned candidate-ranking problem.

### Phase 3 — Diagnose candidate coverage and improve 3D ranking

**Chapters 14–26**

This phase focused heavily on whether the detector generated the right candidates at all, how duplicate 3D candidates should be consolidated, and how the surviving candidates should be ranked.

Key experiments include:

- Ch14 — oracle candidate coverage analysis.
- Ch15 — 3D candidate consolidation / NMS.
- Ch16 — ranking consolidated candidates.
- Ch17 — oracle distance ranking diagnosis.
- Ch18 — binary candidate classifier.
- Ch19 — feature distribution diagnostics.
- Ch20 — engineered 3D patch features.
- Ch21 — attempted 3D `oracle_good` ranking model; recovered notebook contains a missing helper/function dependency.
- Ch22 — diagnostic work around the 3D oracle-good ranking strategy; recovered separately after Kaggle/GitHub sync failure.
- Ch24 — comparison of binary classification and distance-based ranking.
- Ch26 — final candidate detector for this stage of the project.

The recovered history has numbering gaps/ambiguities in this region, so chapter numbers should not be treated as evidence that every intermediate notebook survives.

**Milestone:** Candidate quality became an explicit machine-learning ranking problem, with coverage, 3D consolidation, classification, regression, and feature engineering investigated separately.

### Phase 4 — Move from detection to tracking

**Chapters 27–29**

With a candidate detector in place, the project shifted to temporal association:

- Ch27 — linking detections across consecutive timepoints.
- Ch27 self-contained variant — reproducible consecutive-timepoint linking experiment.
- Ch28 — build/evaluate a fuller cell-tracking approach.
- Ch29 — motion-aware global assignment.

**Milestone:** The project became a tracking system rather than only a cell detector.

### Phase 5 — Improve recall and ranking across samples

**Chapters 30–34**

The project returned to candidate quality after tracking exposed the cost of missed detections:

- Ch30 — improve candidate recall with multi-scale methods.
- Ch31 — stronger ranking models on multiple samples.
- Ch32 — LightGBM/XGBoost learning-to-rank experiment; recovered separately after sync failure.
- Ch33 — graph neural-network features for candidate ranking; recovered separately after sync failure.
- Ch34 — end-to-end detection pipeline.

Chapter 34 became the important upstream detection source for the later tracking/division work, producing the broad candidate set used by subsequent notebooks.

**Milestone:** An end-to-end detection pipeline was established as the upstream input for serious tracking experiments.

### Phase 6 — Build and refine motion-aware tracks

**Chapters 35–37**

#### Chapter 35 — Motion-Aware Tracking with Gap Closing

Introduced physical-coordinate scaling, Hungarian assignment, velocity prediction, and gap closing.

#### Chapter 36 — Track-Aware Detection Filtering

Used temporal information to reduce the broad detection set. In the later diagnostic sample, the process reduced approximately 20,000 candidate detections to 5,000. Later work showed that this filtering could be too aggressive around division events.

#### Chapter 37 — Track Refinement and Gap Closing

Removed weak track fragments and reconnected broken trajectories using motion-aware gap closing. In the later diagnostic sample, the refined result contained 4,642 nodes and 591 tracks.

**Milestone:** Core motion-aware tracking pipeline established.

## Phase 7 — Add divisions and lineage construction

**Chapters 38–41**

### Chapter 38 — Cell Division Detection and Lineage Construction

Generated plausible parent→daughter track relationships, combined daughter pairs into division candidates, scored non-conflicting divisions, and constructed lineage nodes/edges.

**Result:** First complete lineage prototype.

### Chapter 39 — Learning to Predict Cell Divisions

Attempted to replace hand-written division scoring with supervised prediction.

**Critical finding:** The sample under study contained only one ground-truth division, which was insufficient for meaningful supervised learning.

### Chapter 40 — Multi-Sample Division Training

Designed the right conceptual expansion: pool division candidates and labels across multiple training samples. The saved execution, however, still reflected only the existing sample rather than a completed multi-sample training dataset.

### Chapter 41 — GT-Seeded Division Training Examples

Reversed the training-data construction process: begin with a known GT division and search predicted tracks for positive parent/daughter triplets and hard negatives.

**Milestone:** Division inference was isolated as a distinct learning problem, but the project exposed a serious positive-data and candidate-generation bottleneck.

## Phase 8 — Diagnose why divisions fail

**Chapters 42–45**

This is the most important recent phase because it changed the diagnosis of the problem.

### Chapter 42 — Trace the GT Division Through the Detection Pipeline

Traced a known division backward through the existing detection/filtering/tracking pipeline.

Known diagnostic division:

- parent `172000000050`, t=66
- daughter A `173000000050`, t=67
- daughter B `173000000051`, t=67

Pipeline counts for the diagnostic sample included:

- Ch34 broad detections: 20,000
- Ch36 filtered detections: 5,000
- Ch37 refined track nodes: 4,642

**Breakthrough:** Division failure was not merely a bad division classifier. Valid daughters could already be lost upstream.

### Chapter 43 — Recover Division Daughters Before Tracking

Identified two different upstream failure modes:

1. one daughter had a substantially better candidate in the broad Ch34 set, but filtering removed it;
2. another daughter was weak at the exact GT frame, while a much better candidate appeared one frame later.

An experimental rescue step was added before tracking rather than overwriting the baseline filtering pipeline.

**Breakthrough:** Detection/filtering and temporal localization are part of the division problem.

### Chapter 44 — Re-Track with Rescued Division Daughters

Re-ran tracking after restoring likely daughter detections.

The rescued detections survived, but ordinary tracking attached them to trajectories that had begun before the true division.

**Breakthrough:** Recovering the detections alone does not solve lineage. The tracker itself assumes continuation more naturally than birth/division.

### Chapter 45 — Division-Aware Track Splitting

Used the known GT division to deliberately split trajectories at the division point.

**Result:** Demonstrated that the pipeline can represent the correct daughter topology.

**Limitation:** This is a proof of concept, not an inference-ready solution, because GT information seeds the split.

**Milestone:** Correct division topology is representable; autonomous discovery of when and where to perform it remains unsolved.

## Most Important Finding So Far

The current division problem is **not one isolated classifier problem**.

At least three failure layers have been demonstrated:

1. **Detection/filtering:** a true daughter candidate may exist but be removed.
2. **Temporal localization:** the strongest daughter evidence may appear one frame away from the annotated division time.
3. **Tracking topology:** a conventional continuation-oriented tracker may attach a daughter detection to a pre-existing trajectory instead of treating it as a new branch.

This explains why simply training a stronger division classifier is unlikely to solve the end-to-end problem by itself.

## Current Research Question

> Across the training set, where do true divisions actually fail in the current pipeline?

Before another specialized model is added, the project needs to determine whether the dominant bottleneck is candidate generation, filtering, temporal alignment, tracking topology, division candidate generation, or division scoring.

## Recommended Next Experiment — Chapter 46

### Division Dataset Expansion and Pipeline Audit

**Goal:** Stop diagnosing a single known division and audit every recoverable GT division across multiple training samples.

For each GT division, record whether:

1. the parent has an adequate detection near the division frame;
2. daughter A has an adequate detection;
3. daughter B has an adequate detection;
4. both daughters survive the current filtering stage;
5. the tracker produces trajectories that can represent the event;
6. the division-candidate generator proposes the true relationship;
7. the scoring/selection stage retains the correct division.

Suggested audit table:

| Sample | Division | Parent detected | Daughter A detected | Daughter B detected | Both survive filter | Track-compatible | Candidate generated | Correctly selected | Failure stage |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |

### Why this should come next

Chapters 42–45 produced a detailed explanation of **one** division. The project does not yet know how representative that example is. Building another classifier now risks optimizing the wrong stage.

Chapter 46 should turn the recent qualitative discoveries into a quantitative failure dataset. That dataset can then determine whether Chapter 47 should focus on detection recall, temporal daughter rescue, division-aware tracking, candidate generation, or learned division scoring.

## Definition of Success for Chapter 46

Chapter 46 is successful when it can answer, with measured counts rather than intuition:

- How many GT divisions are available across the audited samples?
- What fraction already have adequate parent and daughter candidates?
- What fraction lose daughters during filtering?
- How often is the best daughter evidence temporally displaced?
- How often does ordinary tracking create the wrong topology even when detections exist?
- How often does the true division enter the candidate set?
- Which pipeline stage accounts for the largest share of recoverable failures?

The chapter does **not** need to improve the Kaggle score directly. Its purpose is to choose the next optimization target using evidence.

## Project Maturity

### What has been demonstrated

- Ground-truth and volume visualization
- 3D candidate generation
- Candidate coverage analysis
- Learned candidate ranking
- 3D feature engineering
- Multi-timepoint detection work
- Motion-aware assignment
- Gap closing
- Track refinement
- Division candidate generation
- Lineage construction
- GT-seeded division examples
- Upstream division-failure diagnosis
- Daughter rescue
- Division-aware topology splitting

### What has not yet been demonstrated

- Robust autonomous division discovery across the training set
- A quantitatively validated division-aware tracking strategy
- End-to-end lineage construction without GT assistance
- A final competition-ready inference pipeline validated across representative samples

## Project Management Rule Going Forward

**Kaggle is the laboratory. GitHub is the durable project record.**

New experiments should be created because they answer a specific unresolved research question, not merely to continue the chapter numbering. Each meaningful experiment should eventually record:

- question/hypothesis,
- method/change,
- result/metrics,
- failure or lesson,
- decision for the next experiment.

Failed experiments are part of the research history when they explain why the project changed direction.

## Immediate Next Step

Build **Chapter 46 — Division Dataset Expansion and Pipeline Audit** before designing another division model.

The decision after Chapter 46 should be driven by the measured distribution of failure stages.