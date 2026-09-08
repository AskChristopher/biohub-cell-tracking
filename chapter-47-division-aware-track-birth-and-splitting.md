# Chapter 47 — Division-Aware Track Birth and Splitting

**Status:** Complete — negative/informative result  
**Type:** Targeted topology experiment  
**Executed notebook:** Chapter 47 V4  
**GitHub commit:** `5f58e6bc13964914a48228a7833eb49bb0908173`

## Why Chapter 47 Existed

Chapter 46A V12 reproduced the historical Chapter 34 → 35 → 36 → 37 pipeline end-to-end on three samples containing 13 binary ground-truth divisions. The Chapter 46 multi-sample audit then showed that the dominant measured failure was tracking topology rather than detection.

Measured primary outcomes before Chapter 47:

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

Chapter 34 had adequate parent and daughter candidates for 100% of the audited events. The working hypothesis was therefore that a post-tracking topology transform might repair divisions without changing detection, filtering, or historical tracking.

## Research Question

> Can the current tracking output be transformed into biologically plausible division topology by detecting likely branch events and creating new daughter tracks, without using ground truth to decide where to split?

## Hypothesis

If finished Chapter 37 trajectories retain enough geometric evidence of true cell division, then a label-free post-hoc heuristic should be able to identify a small set of plausible one-parent→two-daughter branch events and improve topology without creating an uncontrolled number of false-positive divisions.

## Experimental Constraint

Chapter 47 was a **topology-only experiment**.

The upstream V12 artifacts were held fixed. The split proposal used Chapter 37 refined track geometry rather than GT division locations.

The final V4 implementation intentionally used only the three V12 files already proven available in `46DataV12`:

- `chapter34_top200_detections_multisample.csv`
- `chapter36_filtered_detections_multisample.csv`
- `chapter37_refined_track_nodes_multisample.csv`

No GEFF/Zarr reopening was required for the structural experiment.

## Method

### 1. Reconstruct Chapter 37 tracks

Build per-track start/end/duration summaries and physical coordinates from the refined track-node table.

### 2. Generate label-free branch candidates

For mature tracks, search near the track end for two plausible post-split branches. A branch can be either:

- the parent's own continuation suffix; or
- a distinct track beginning shortly after the proposed split.

Candidate plausibility used only tracking geometry such as:

- parent-to-branch distance
- temporal gap
- daughter separation
- branch persistence
- whether one branch is the current parent continuation and the other is a distinct emerging track

### 3. Score candidates with a fixed heuristic

The score combined interpretable components for distance, temporal proximity, persistence, separation, and structural pattern. The threshold was fixed before interpreting the result.

### 4. Select non-conflicting events

The notebook enforced a per-sample cap of 25 selected divisions and prevented obvious branch reuse conflicts.

### 5. Apply a reversible topology transform

The original Chapter 37 track ID was preserved for every node. New daughter track IDs were written separately, continuation edges were rebuilt from transformed sequences, and parent→daughter lineage edges were written as a separate table.

## Chapter 47 V4 Result

The notebook executed successfully on all three V12 samples.

| Measure | Result |
| --- | ---: |
| Samples | 3 |
| Chapter 37 tracks | 1,347 |
| Label-free split candidates | 5,375 |
| Selected divisions | 75 |
| Lineage edges | 150 |
| Nodes moved into daughter tracks | 1,111 |
| Continuation edges rebuilt | 12,473 |

The 75 selected divisions are the critical result. The selection rule was capped at **25 per sample**, and all three samples reached the cap.

The same three samples contain only **13 known binary GT divisions** in total.

Therefore the post-hoc heuristic failed the experiment's false-positive sanity gate. It found branch-like geometry far more often than true divisions occur.

## Interpretation

This was **not an implementation failure**.

The notebook successfully:

- generated label-free candidates;
- scored and selected them;
- created daughter track IDs;
- moved post-split nodes;
- rebuilt continuation edges;
- created lineage edges;
- saved the transformed artifacts.

The failure is architectural and scientific: **finished Chapter 37 track geometry is not selective enough to distinguish true divisions from ordinary branch-like configurations.**

The experiment suggests that important evidence has already been collapsed by the time Chapter 37 trajectories are finalized.

## Why We Will Not Tune Chapter 47 Against the Same 13 Events

A simple response would be to raise the threshold until the predicted division count moves closer to 13. That would risk overfitting the tiny diagnostic set and would violate the purpose of this fixed-heuristic experiment.

The negative result is more valuable as a clean architectural signal.

## Decision

**Do not continue adding post-hoc geometric heuristics to Chapter 37 output as the primary division strategy.**

Move the division decision earlier into temporal assignment.

The next experiment is:

# Chapter 48 — Division-Aware Tracking at Assignment Time

Chapter 48 should compare competing local assignment hypotheses while the evidence is still available:

1. **ordinary continuation:** one parent → one child detection;
2. **division:** one parent terminates → two daughter detections begin.

Useful assignment-time evidence includes:

- competing detections in the next frame;
- parent velocity and predicted position;
- assignment costs;
- simultaneous plausibility of two daughter detections;
- daughter separation;
- new-track birth timing;
- ±1-frame temporal tolerance;
- whether a one-to-one continuation would leave another highly plausible daughter unmatched or force an implausible trajectory.

The goal is not to create more branches. The goal is to make division an explicit competing assignment hypothesis with a cost/score that can remain selective.

## Chapter 47 Portfolio Significance

Chapter 47 is an important negative result in the project story:

1. Chapter 46 localized the dominant failure to topology.
2. Chapter 47 tested the cheapest plausible correction: infer divisions after tracking from finished geometry.
3. The transform worked technically but generated 5,375 candidates and selected 75 divisions across samples containing 13 known binary divisions.
4. That result rejected the post-hoc geometry path as insufficiently selective.
5. The architecture now moves upstream to assignment-time division reasoning.

This is exactly the kind of experimental result worth preserving: the failure narrows the design space and directly determines the next system architecture.
