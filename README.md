# BioHub Cell Tracking

An experimental machine-learning project for detecting and tracking cells through 3D microscopy time series and reconstructing cell lineages, developed through the Kaggle BioHub Cell Tracking competition.

This repository documents the evolution of the solution rather than presenting a single finished model. The notebooks record the research process: exploring the microscopy data, building candidate detectors, learning to rank cell-center candidates, linking detections through time, refining tracks, detecting divisions, diagnosing failure modes, and experimenting with lineage topology.

> **Current state:** the core detection and motion-aware tracking pipeline is a working prototype. Correct division topology has been demonstrated with ground-truth assistance, but autonomous division inference remains the main unresolved problem.

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

One of the most important findings from this project is that division errors are not isolated to the final division classifier. A daughter cell can be lost during detection or filtering, appear most strongly one frame away from the annotated division time, or be incorrectly absorbed into an existing trajectory by a tracker designed primarily for continuation.

## Research Progression

The surviving notebook history can be grouped into eight major phases.

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

See [`PROJECT_STATUS.md`](PROJECT_STATUS.md) for the detailed reconstructed history and current research roadmap.

## Current Pipeline

The current experimental pipeline is approximately:

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
division candidate generation
        ↓
lineage topology
```

The later notebooks revealed that information can be lost at several transitions in this pipeline, particularly around division events.

## Key Technical Findings

### Detection quality is necessary but not sufficient

The broad detection pipeline can contain a good daughter-cell candidate that later filtering removes. Improving only the downstream division model cannot recover information that has already disappeared.

### Division timing can be ambiguous

For a diagnosed ground-truth division, one daughter was weak at the annotated frame while a substantially better candidate appeared one frame later. Division-aware inference therefore needs to tolerate small temporal offsets.

### Ordinary tracking has the wrong topology for division

After likely daughter detections were rescued and re-tracked, the tracker could still attach them to trajectories that began before the actual division. Conventional assignment naturally models **continuation**; a cell division requires **one parent becoming two new branches**.

### Correct topology is representable

The Chapter 45 experiment deliberately split tracks using a known ground-truth division and demonstrated that the pipeline can represent the desired lineage structure. The remaining challenge is discovering those split events autonomously.

## Selected Notebook Milestones

- **Ch14 — Oracle Candidate Coverage Analysis:** asks whether the proposal generator contains sufficiently good candidates before ranking.
- **Ch15–20 — 3D Consolidation and Ranking:** investigates NMS, ranking, binary classification, feature diagnostics, and 3D patch features.
- **Ch27–29 — Temporal Association:** moves from independent detections to cell tracking and motion-aware assignment.
- **Ch30–34 — Multiscale Detection and Ranking:** improves candidate recall and culminates in the end-to-end detection pipeline used by later experiments.
- **Ch35–37 — Motion-Aware Tracking:** establishes assignment, track-aware filtering, refinement, and gap closing.
- **Ch38 — Division Detection and Lineage Construction:** first complete lineage prototype.
- **Ch39–41 — Learning Division Events:** exposes the shortage of positive division examples and explores GT-seeded training examples.
- **Ch42 — GT Division Pipeline Trace:** demonstrates that division failures can originate upstream.
- **Ch43 — Daughter Rescue:** restores likely daughter detections before tracking.
- **Ch44 — Re-Tracking Rescued Daughters:** reveals the topology problem even when detections survive.
- **Ch45 — Division-Aware Track Splitting:** proves correct topology can be constructed, while remaining GT-assisted rather than inference-ready.

## Current Research Frontier

The next planned experiment is **Chapter 46 — Division Dataset Expansion and Pipeline Audit**.

Instead of optimizing another model against one known division, the goal is to audit ground-truth divisions across multiple training samples and determine where each event fails:

```text
GT division
   ↓
parent/daughters detected?
   ↓
survive filtering?
   ↓
track-compatible?
   ↓
true division candidate generated?
   ↓
correctly scored/selected?
```

The result should identify which component deserves the next optimization effort: candidate recall, temporal daughter rescue, division-aware tracking, candidate generation, or division scoring.

## Repository Notes

This repository originated as a Kaggle notebook research sequence. Kaggle remains the experimental environment; GitHub is the durable project record.

Not every historical chapter survives. Some notebooks failed during Kaggle-to-GitHub synchronization and were recovered separately, while Chapter 2 is missing. Numbering gaps therefore reflect the research/archive history and should not be interpreted as missing production releases.

Failed experiments are intentionally part of the project story when they explain why the approach changed.

## Project Status

This is an **active research prototype**, not a production cell-tracking library.

**Working/proven:** 3D candidate generation, learned ranking experiments, temporal association, motion-aware tracking, gap closing, track refinement, division candidate generation, lineage representation, and diagnostic tooling.

**Still unresolved:** robust autonomous division discovery, quantitatively validated division-aware tracking across representative samples, and end-to-end lineage inference without ground-truth assistance.

For the detailed status, evidence, reconstructed notebook history, and Chapter 46 plan, see **[`PROJECT_STATUS.md`](PROJECT_STATUS.md)**.

---

**Christopher Mathews**  
Machine learning / AI portfolio project
