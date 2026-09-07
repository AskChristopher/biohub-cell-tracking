# Chapter 47 — Division-Aware Track Birth and Splitting

**Status:** Ready to implement in Kaggle  
**Type:** Targeted topology experiment  
**Depends on:** Chapters 37, 45, 46, and Chapter 46A V12 multi-sample artifacts

## Why Chapter 47 Exists

Chapter 46A V12 reproduced the historical Chapter 34 → 35 → 36 → 37 pipeline end-to-end on three samples containing 13 binary ground-truth divisions. The Chapter 46 multi-sample audit then quantified where those 13 events fail.

Measured primary outcomes:

| Outcome | Events | Percent |
| --- | ---: | ---: |
| Daughters share one track | 5 | 38.5% |
| Daughter attached to parent track | 3 | 23.1% |
| One daughter filtered | 2 | 15.4% |
| Both daughters filtered | 1 | 7.7% |
| Topology compatible | 2 | 15.4% |

Grouped by pipeline layer:

- detection/ranking: 0 / 13 primary failures
- filtering: 3 / 13 primary failures (23.1%)
- tracking/topology: 8 / 13 primary failures (61.5%)
- topology compatible: 2 / 13 (15.4%)

Chapter 34 had adequate parent and daughter candidates for 100% of the audited events. The dominant problem is therefore no longer candidate generation. The tracker is forcing division daughters into continuation trajectories.

## Research Question

> Can the current tracking output be transformed into biologically plausible division topology by detecting likely branch events and creating new daughter tracks, without using ground truth to decide where to split?

## Hypothesis

A continuation-oriented tracker will frequently place two division daughters on one existing trajectory or attach one daughter to the parent trajectory. If we add a post-tracking division-aware topology step that can terminate a parent track and create two daughter births near a plausible division event, then the fraction of topology-compatible GT divisions should increase without changing Chapter 34 detection, Chapter 36 filtering, or the historical Chapter 37 refinement algorithm.

## Experimental Constraint

This is a **topology experiment**, not a new detector or filter experiment.

Do not change:

- Chapter 34 candidate generation/ranking
- Chapter 35 motion-aware tracking settings
- Chapter 36 filtering thresholds or models
- Chapter 37 refinement/gap-closing settings

Ground truth may be used only for **evaluation**, never for selecting where to split a track.

## Inputs

The implementation notebook should consume the V12 combined artifacts and the Chapter 46 audit outputs.

### Required V12 files

- `chapter34_top200_detections_multisample.csv`
- `chapter36_filtered_detections_multisample.csv`
- `chapter37_refined_track_nodes_multisample.csv`
- `chapter37_refined_track_edges_multisample.csv`
- `chapter37_refined_track_summary_multisample.csv`

### Required Chapter 46 files

- `chapter46_multisample_division_audit.csv`
- `chapter46_failure_distribution.csv`
- `chapter46_topology_summary.csv`
- `chapter46_ch47_decision_table.csv`

### Required competition input

The original BioHub Cell Tracking competition dataset must remain attached so GT GEFF data is available for final evaluation only.

### Optional but useful files

- `chapter36_filtered_track_nodes_multisample.csv`
- `chapter36_filtered_track_edges_multisample.csv`
- `chapter35_track_nodes_multisample.csv`
- `chapter35_track_edges_multisample.csv`
- `chapter46a_selected_samples.csv`
- `chapter46a_validation.csv`

These optional files make it easier to diagnose whether a proposed split is inherited from Chapter 35/36 or introduced only after Chapter 37 refinement.

## Proposed Method

### Step 1 — Reconstruct refined tracks

Load the Chapter 37 refined nodes and edges for the three V12 samples. Build a per-track representation containing:

- track ID
- start/end frame
- ordered node sequence
- coordinates and physical coordinates
- local velocity before/after each candidate split frame
- nearest neighboring tracks around each frame

### Step 2 — Generate label-free split candidates

For each mature parent track, search a small temporal window near the end of the track for evidence that one trajectory should become two.

A candidate event should be based only on predicted-tracking evidence such as:

- two nearby detections/tracks emerging after the same parent state
- spatial plausibility of both daughters relative to the parent endpoint
- daughter track starts shortly after the proposed division frame
- motion continuity from parent to each daughter
- daughters becoming spatially distinct from one another
- both daughter branches persisting for more than a trivial number of frames
- no reliance on GT node IDs, GT division frames, or GT coordinates

### Step 3 — Score candidate branch events

Build an interpretable heuristic score first rather than immediately training a classifier. Candidate components can include:

- parent→daughter A distance
- parent→daughter B distance
- temporal offset
- daughter separation
- branch persistence
- direction consistency
- assignment-cost evidence if available
- whether the current tracker has both daughter detections on the same track
- whether a candidate daughter track already began before the proposed division frame

Keep every component in the output table so failures remain diagnosable.

### Step 4 — Apply a non-destructive topology transform

Do not overwrite the original Chapter 37 artifacts.

For an accepted split candidate:

1. preserve the original parent track up to the selected split frame;
2. create two new daughter track IDs beginning after the split;
3. move the corresponding post-split node sequences into those daughter tracks;
4. preserve ordinary continuation edges within each segment;
5. add explicit parent→daughter lineage edges separately from continuation edges;
6. ensure no node belongs to multiple continuation tracks;
7. ensure a parent has at most two daughter branches in this binary-division experiment.

### Step 5 — Evaluate against the same 13 GT divisions

Reuse the Chapter 46 evaluation logic only after the label-free splits are generated.

Compare **before vs after** for:

- topology-compatible GT divisions
- daughters sharing a track
- daughter attached to parent track
- daughter tracks predating the division
- temporal displacement
- number of false-positive inferred divisions
- number of predicted divisions per sample

The baseline for this experiment is the Chapter 46 result:

- 13 GT binary divisions
- 8 tracking/topology primary failures
- 3 filtering primary failures
- 2 topology-compatible primary outcomes

## Success Criteria

Chapter 47 is successful if it demonstrates all of the following:

1. **No GT leakage:** the split location and daughter identities are chosen without GT.
2. **Topology improves:** more than 2 of 13 events become primary `TOPOLOGY_COMPATIBLE` outcomes, or the raw topology-compatible count materially exceeds the Chapter 46 baseline.
3. **Dominant failure reduced:** `DAUGHTERS_SHARE_TRACK` and `DAUGHTER_ATTACHED_TO_PARENT_TRACK` decrease meaningfully.
4. **False positives measured:** improvement is not achieved by creating an uncontrolled number of divisions.
5. **Original artifacts preserved:** the post-processing transform is reversible and produces new output tables.

A weak or negative result is still useful if it shows that topology cannot be inferred reliably from the current Chapter 37 outputs alone.

## Required Outputs

- `chapter47_division_candidates.csv`
- `chapter47_selected_divisions.csv`
- `chapter47_split_track_nodes.csv`
- `chapter47_split_track_edges.csv`
- `chapter47_lineage_edges.csv`
- `chapter47_before_after_audit.csv`
- `chapter47_failure_distribution_before_after.csv`
- `chapter47_summary.csv`

## Decision After Chapter 47

If label-free splitting materially improves the 13-event audit without excessive false positives, the next step should integrate the topology logic into the competition inference path and evaluate score impact.

If it fails because plausible daughter branches are not identifiable from Chapter 37 alone, the next experiment should move the division decision earlier, into tracking itself, rather than adding another post-hoc split heuristic.

If filtering remains the dominant residual error after topology improves, revisit Chapter 36 with a division-preserving filter only after this topology experiment is complete.

## Portfolio Significance

Chapter 47 is the point where the project moves from diagnosing a graph-topology failure to testing an autonomous structural correction. The experiment should remain explicit about causality: detection and filtering stay fixed, topology changes, and the same multi-sample audit measures whether the intervention worked.
