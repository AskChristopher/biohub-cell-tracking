# BioHub Cell Tracking

An experimental machine-learning project for detecting and tracking cells through 3D microscopy time series and reconstructing cell lineages, developed through the Kaggle BioHub Cell Tracking competition.

This repository documents the evolution of the solution rather than presenting a single finished model. The notebooks record the research process: exploring microscopy data, building candidate detectors, learning to rank cell-center candidates, linking detections through time, refining tracks, detecting divisions, diagnosing failure modes, and experimenting with lineage topology.

> **Current state:** the core detection and motion-aware tracking pipeline is reproducible across multiple samples. A 13-event multi-sample audit shows that the dominant remaining division failure is **tracking topology**, not detection: continuation-oriented tracking frequently merges daughters into one trajectory or attaches a daughter to the parent track. Chapter 47 will test autonomous division-aware track birth/splitting.

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
division-aware topology inference   ← current frontier
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
| 10 | Division-aware track birth/splitting | **Next:** infer topology corrections without GT |

See [`PROJECT_STATUS.md`](PROJECT_STATUS.md) for the detailed reconstructed history and current roadmap.

## Key Technical Findings

### Detection/ranking is not the dominant division bottleneck

Chapter 46A V12 reproduced the Ch34→35→36→37 pipeline on three samples containing 13 binary GT divisions. The Chapter 46 audit found adequate Chapter 34 parent and daughter candidates for **100% of those events**.

### Filtering still hurts daughters, but it is secondary

Across the 13-event audit, filtering was the primary failure for **3/13 events (23.1%)**. Parent retention remained strong, while daughter retention was lower.

### Tracking topology is the dominant failure

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

This changes the research direction. The earlier one-event diagnosis made daughter filtering look central; the multi-sample audit shows that the larger problem is continuation-oriented track topology.

### Division timing remains ambiguous

The multi-sample audit also found frequent ±1-frame displacement of daughter evidence. Division inference should tolerate small temporal offsets rather than assuming exact frame alignment.

### Correct topology is representable

Chapter 45 used GT-seeded splitting to demonstrate that the graph representation can express parent→two-daughter lineage topology. The remaining problem is discovering where and when to perform that split autonomously.

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
- **Ch47 — Division-Aware Track Birth and Splitting:** next targeted experiment; see [`chapter-47-division-aware-track-birth-and-splitting.md`](chapter-47-division-aware-track-birth-and-splitting.md).

## Current Research Frontier

The next experiment is **Chapter 47 — Division-Aware Track Birth and Splitting**.

Research question:

> Can the current refined tracking output be transformed into biologically plausible division topology by detecting likely branch events and creating new daughter tracks, without using ground truth to choose where to split?

For causal clarity, Chapter 47 will keep the historical Ch34 detection, Ch36 filtering, and Ch37 refinement fixed. Only the topology stage changes. The same 13 GT divisions will be used for evaluation after the label-free split decisions are generated.

## Repository Notes

This repository originated as a Kaggle notebook research sequence. **Kaggle is the laboratory; GitHub is the durable project record.**

Not every historical chapter survives. Some notebooks failed during Kaggle-to-GitHub synchronization and were recovered separately, while Chapter 2 is missing. Numbering gaps therefore reflect the archive history rather than production releases.

Failed experiments are intentionally preserved when they explain why the approach changed.

## Project Status

This is an **active research prototype**, not a production cell-tracking library.

**Working/proven:** 3D candidate generation, learned ranking experiments, motion-aware tracking, filtering, gap closing, track refinement, division candidate generation, lineage representation, multi-sample artifact reproduction, multi-sample division auditing, and GT-assisted topology splitting.

**Still unresolved:** autonomous division-aware track birth/splitting, false-positive-controlled lineage inference without GT, competition-ready integration, and final leaderboard validation.

For detailed status and evidence, see **[`PROJECT_STATUS.md`](PROJECT_STATUS.md)**.

---

**Christopher Mathews**  
Machine learning / AI portfolio project
