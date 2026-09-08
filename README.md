# BioHub Cell Tracking

An experimental machine-learning project for detecting and tracking cells through 3D microscopy time series and reconstructing cell lineages, developed through the Kaggle BioHub Cell Tracking competition.

This repository documents the evolution of the solution rather than presenting a single finished model. The notebooks record the research process: exploring microscopy data, building candidate detectors, learning to rank cell-center candidates, linking detections through time, refining tracks, detecting divisions, diagnosing failure modes, and experimenting with lineage topology.

> **Current state:** the core detection and motion-aware tracking pipeline is reproducible across multiple samples. Chapter 46 showed that division failure is dominated by tracking topology rather than detection. Chapter 47 then tested the cheapest post-hoc fix and found that Chapter 37 geometry alone produces far too many plausible split hypotheses. The next frontier is **division-aware tracking at assignment time**.

## The Problem

Cell tracking requires more than finding bright objects in individual microscopy frames. A useful system must determine which detections represent real cells, associate those cells across time, handle missed detections and gaps, recognize when one cell divides into two, and construct a biologically plausible lineage graph.

That makes this a coupled problem involving:

- 3D candidate detection
- candidate ranking and filtering
- temporal data association
- motion modeling and gap closing
- track refinement
- cell-division detection
- lineage construction

## Current Pipeline

```text
3D microscopy volumes
        ↓
multiscale candidate generation
        ↓
candidate ranking / filtering
        ↓
motion-aware temporal assignment
        ↓
track refinement + gap closing
        ↓
division-aware assignment hypotheses   ← current frontier
        ↓
lineage graph
```

## Research Progression

| Phase | Focus | Outcome |
| --- | --- | --- |
| 1 | Data and ground-truth exploration | Established visualization and understanding of tracks/divisions |
| 2 | Baseline detection | Moved from hand-tuned detection toward learned candidate filtering |
| 3 | 3D candidate ranking | Explored coverage, NMS, classification, regression, and 3D features |
| 4 | Temporal tracking | Linked detections and introduced motion-aware global assignment |
| 5 | Recall and stronger ranking | Built multiscale proposals and stronger ranking experiments |
| 6 | Track refinement | Added motion prediction, filtering, refinement, and gap closing |
| 7 | Division and lineage modeling | Built division candidates and the first lineage prototype |
| 8 | Division failure diagnosis | Traced failures upstream, rescued daughters, and proved track splitting |
| 9 | Multi-sample reproduction + audit | Reproduced Ch34→37 on 3 samples and quantified 13 GT divisions |
| 10 | Post-hoc division-aware splitting | Ch47 showed Chapter 37 geometry alone overgenerates division hypotheses |
| 11 | Division-aware assignment | **Next:** consider one-parent→two-daughter hypotheses during tracking |

See [`PROJECT_STATUS.md`](PROJECT_STATUS.md) for the detailed reconstructed history and current roadmap.

## Key Technical Findings

### Detection/ranking is not the dominant division bottleneck

Chapter 46A V12 reproduced the Ch34→35→36→37 pipeline on three samples containing 13 binary GT divisions. The Chapter 46 audit found adequate Chapter 34 parent and daughter candidates for **100% of those events**.

### Filtering still hurts daughters, but it is secondary

Across the 13-event audit, filtering was the primary failure for **3/13 events (23.1%)**. Parent retention remained strong, while daughter retention was lower.

### Tracking topology is the dominant measured failure

The Chapter 46 primary failure distribution was:

- daughters share one track: **5/13 (38.5%)**
- daughter attached to parent track: **3/13 (23.1%)**
- one daughter filtered: **2/13 (15.4%)**
- both daughters filtered: **1/13 (7.7%)**
- topology compatible: **2/13 (15.4%)**

Grouped by layer:

- detection/ranking: **0/13 primary failures**
- filtering: **3/13 (23.1%)**
- tracking/topology: **8/13 (61.5%)**
- topology-compatible primary outcome: **2/13 (15.4%)**

This changed the research direction. The earlier one-event diagnosis made daughter filtering look central; the multi-sample audit showed that the larger problem is continuation-oriented track topology.

### Chapter 47 ruled out the simplest post-hoc correction

Chapter 47 V4 held the upstream V12 artifacts fixed and generated division hypotheses using only Chapter 37 track geometry. The notebook completed successfully, but the heuristic was far too permissive:

- Chapter 37 tracks: **1,347**
- label-free split candidates: **5,375**
- selected divisions: **75**
- lineage edges created: **150**
- nodes moved into daughter branches: **1,111**
- continuation edges rebuilt: **12,473**

The selection cap was 25 divisions per sample, and all three samples hit that ceiling. The three samples contain only 13 known binary GT divisions in total, so the post-hoc geometry heuristic failed the false-positive sanity gate.

This is an informative negative result: **finished Chapter 37 trajectories do not retain enough selective evidence to infer divisions reliably by geometry alone.**

### Division timing remains ambiguous

The multi-sample audit found frequent ±1-frame displacement of daughter evidence. Division logic should tolerate small temporal offsets rather than assuming exact frame alignment.

### Correct topology is representable

Chapter 45 used GT-seeded splitting to demonstrate that the graph representation can express parent→two-daughter lineage topology. The remaining problem is discovering that structure during inference without GT and without creating excessive false-positive branches.

## Selected Notebook Milestones

- **Ch14 — Oracle Candidate Coverage Analysis:** asks whether the proposal generator contains sufficiently good candidates before ranking.
- **Ch27–29 — Temporal Association:** moves from independent detections to motion-aware tracking.
- **Ch30–34 — Multiscale Detection and Ranking:** culminates in the end-to-end detection pipeline.
- **Ch35–37 — Motion-Aware Tracking:** establishes assignment, track-aware filtering, refinement, and gap closing.
- **Ch38 — Division Detection and Lineage Construction:** first complete lineage prototype.
- **Ch39–41 — Learning Division Events:** exposes the shortage of positive division examples and explores GT-seeded examples.
- **Ch42 — GT Division Pipeline Trace:** shows that division failure can originate upstream.
- **Ch43 — Daughter Rescue:** restores likely daughter detections before tracking.
- **Ch44 — Re-Tracking Rescued Daughters:** exposes the topology problem even when detections survive.
- **Ch45 — Division-Aware Track Splitting:** proves correct topology is representable, but with GT assistance.
- **Ch46A V12 — Multi-Sample Artifact Generation:** reproduces Ch34→37 on 3 samples end-to-end with no stage failures.
- **Ch46 V4 — Multi-Sample Division Pipeline Audit:** audits 13 GT divisions and identifies tracking/topology as the dominant measured failure.
- **Ch47 V4 — Post-Hoc Division-Aware Track Birth/Splitting:** generates 5,375 candidates and selects 75 divisions, failing the false-positive sanity gate.
- **Ch48 — Division-Aware Tracking at Assignment Time:** next targeted experiment.

## Current Research Frontier

The next experiment is **Chapter 48 — Division-Aware Tracking at Assignment Time**.

Research question:

> Can the tracker improve lineage topology by evaluating a one-parent→two-daughter assignment hypothesis at the moment of temporal association, before continuation decisions collapse the evidence into finished trajectories?

The motivation comes directly from Chapter 47. Post-hoc geometry can find many branch-like patterns, but it cannot distinguish true divisions selectively enough. Chapter 48 should use information available during assignment—competing detections, assignment costs, parent velocity, temporal birth evidence, and simultaneous plausibility of two daughters—rather than trying to reconstruct the decision afterward.

## Repository Notes

This repository originated as a Kaggle notebook research sequence. **Kaggle is the laboratory; GitHub is the durable project record.**

Not every historical chapter survives. Some notebooks failed during Kaggle-to-GitHub synchronization and were recovered separately, while Chapter 2 is missing. Numbering gaps therefore reflect the archive history rather than production releases.

Failed experiments are intentionally preserved when they explain why the approach changed.

## Project Status

This is an **active research prototype**, not a production cell-tracking library.

**Working/proven:** 3D candidate generation, learned ranking experiments, motion-aware tracking, filtering, gap closing, track refinement, division candidate generation, lineage representation, multi-sample artifact reproduction, multi-sample division auditing, GT-assisted topology splitting, and a completed label-free post-hoc topology experiment.

**Still unresolved:** selective autonomous division inference, division-aware assignment during tracking, false-positive-controlled lineage construction, competition-ready integration, and final leaderboard validation.

For detailed status and evidence, see **[`PROJECT_STATUS.md`](PROJECT_STATUS.md)**.

---

**Christopher Mathews**  
Machine learning / AI portfolio project
