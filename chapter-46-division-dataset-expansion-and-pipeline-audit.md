# Chapter 46 — Division Dataset Expansion and Pipeline Audit

**Status:** Ready to implement in Kaggle  
**Type:** Diagnostic / dataset-building experiment  
**Depends on:** Chapters 34, 36, 37, 38, and diagnostic findings from Chapters 42–45

## Research Question

**Across multiple training samples, where do ground-truth cell divisions fail in the current detection → filtering → tracking → division pipeline?**

Chapters 42–45 explained one known division in detail. That case demonstrated at least three distinct failure modes:

1. a useful daughter candidate can exist in the broad Ch34 detection set and then be removed by Ch36 filtering;
2. the strongest evidence for a daughter can occur one frame away from the annotated division time;
3. even when daughter detections survive, ordinary continuation-oriented tracking can attach them to trajectories that began before the true division.

Those findings are important, but they come from one diagnosed event. Chapter 46 must determine whether they generalize.

This chapter should **measure the pipeline before changing it**.

---

## Hypothesis

Division failures are distributed across multiple upstream stages rather than being dominated by the final division classifier.

In particular, we expect some GT divisions to fail because of:

- insufficient candidate recall;
- filtering loss;
- temporal displacement of daughter evidence;
- continuation-oriented tracking topology;
- failure to generate the true parent/daughter division candidate;
- or failure to score/select a candidate that was successfully generated.

Chapter 46 should not assume which failure mode is dominant. Its purpose is to find out.

---

## Non-Goal

This chapter is **not** intended to:

- train another division classifier;
- tune Chapter 36 filtering thresholds;
- introduce another daughter-rescue heuristic;
- modify the tracker;
- use GT information to repair the final inference graph;
- optimize the Kaggle leaderboard score directly.

Any of those may become Chapter 47, but only after the audit identifies the highest-value bottleneck.

---

# 1. Reuse the Existing Pipeline

Do not redesign the detector or tracker for this experiment.

The audit should reproduce the relevant existing stages as faithfully as practical:

```text
training sample + GT lineage
        ↓
Ch34-style broad detection set
        ↓
Ch36-style track-aware filtering
        ↓
Ch37-style refined tracking
        ↓
Ch38-style division candidate generation / scoring
        ↓
GT division audit
```

The diagnostic logic from Chapters 42–45 should be generalized from one known division to every usable GT division in the selected training samples.

If exact reuse of an earlier artifact is impossible in Kaggle, rebuild the necessary stage inside this notebook and explicitly document the difference. Do not silently substitute a new algorithm.

---

# 2. Discover Ground-Truth Divisions Programmatically

Do not hard-code the Chapter 42 division.

For every selected training sample, inspect its GT graph and identify all parent nodes with two or more outgoing daughter relationships consistent with a division event.

Create a canonical division table with at least:

| Field | Meaning |
| --- | --- |
| `sample_id` | training sample |
| `division_id` | stable audit identifier |
| `parent_gt_id` | GT parent node/cell |
| `parent_t` | parent timepoint |
| `parent_z/y/x` | parent physical or voxel coordinates |
| `daughter_a_gt_id` | first GT daughter |
| `daughter_b_gt_id` | second GT daughter |
| `daughter_t` | expected daughter timepoint |
| daughter coordinates | GT daughter locations |

If GT contains unusual events with more than two daughters, flag them rather than silently coercing them into binary divisions.

### Validation

Print:

- number of training samples inspected;
- GT divisions per sample;
- total GT divisions found;
- any events excluded and why.

---

# 3. Define Detection Matching Once

The audit requires a consistent definition of whether a GT cell is represented by a predicted candidate.

Use the same physical-coordinate conventions and distance logic established in the earlier detection/ranking notebooks whenever possible.

For every GT parent/daughter target, calculate:

- nearest Ch34 candidate distance;
- nearest Ch36 surviving candidate distance;
- nearest Ch37 track-node distance;
- rank/score of the best relevant Ch34 candidate where available.

Do not reduce the analysis immediately to a boolean. Preserve the continuous distances so thresholds can be changed later without rerunning the expensive pipeline.

Then derive thresholded flags such as:

```python
parent_detected_ch34
daughter_a_detected_ch34
daughter_b_detected_ch34
parent_survives_ch36
daughter_a_survives_ch36
daughter_b_survives_ch36
```

The exact distance threshold must be printed in the notebook and recorded in the output metadata.

---

# 4. Audit Temporal Daughter Evidence

Chapter 43 showed that the best daughter candidate may occur one frame away from the annotated daughter time.

Generalize that diagnostic.

For each daughter, inspect at least:

```text
t_expected - 1
t_expected
t_expected + 1
```

where valid.

For each offset record:

- nearest candidate distance;
- candidate score/rank;
- candidate coordinates;
- whether it survives filtering;
- whether it becomes part of a refined track.

Derive:

```python
best_daughter_a_dt
best_daughter_b_dt
```

where `dt` is the temporal offset of the strongest spatially plausible evidence.

Also derive:

```python
daughter_a_temporally_displaced
daughter_b_temporally_displaced
```

This will quantify whether the temporal issue found in Chapter 43 is exceptional or systematic.

---

# 5. Audit Filtering Loss

For each GT target with an adequate Ch34 candidate, determine whether a corresponding candidate remains after Ch36 filtering.

Distinguish:

```text
NO_CANDIDATE
```

from:

```text
CANDIDATE_FILTERED
```

This distinction is essential.

A detector recall failure and a filtering failure require different solutions.

Record the Ch34 candidate's relevant score/rank/features when possible so later analysis can ask why useful daughter candidates were removed.

---

# 6. Audit Track Topology

For parent and daughter detections that reach the Ch37-style tracking stage, identify the corresponding predicted track IDs.

For each GT division record:

- parent predicted track ID;
- daughter A predicted track ID;
- daughter B predicted track ID;
- start time of each daughter-associated track;
- whether daughters share a track;
- whether either daughter-associated trajectory begins before the GT division;
- whether daughter tracks appear as plausible births around the division time.

Derive flags such as:

```python
daughter_a_track_predates_division
daughter_b_track_predates_division
daughters_share_track
track_topology_compatible
```

A division is **track-compatible** only if the predicted trajectories preserve enough structure for a downstream division model to represent the event without rewriting large portions of the tracks.

Do not use GT to split or repair tracks in Chapter 46. This stage is diagnostic only.

---

# 7. Audit Division Candidate Generation

Run the Ch38-style division candidate generator against the refined tracks.

For each GT division determine whether a candidate corresponding to the correct parent and both daughter trajectories appears in the generated candidate set.

Record:

```python
true_division_candidate_generated
true_candidate_score
true_candidate_rank
true_candidate_selected
```

If exact GT-to-track correspondence is ambiguous, retain the ambiguity and flag the row for inspection rather than forcing a positive/negative label.

This separates two downstream failure modes:

- **candidate-generation failure** — the correct hypothesis never exists;
- **selection/scoring failure** — the correct hypothesis exists but loses.

---

# 8. Assign a Primary Failure Stage

Assign each audited division a mutually interpretable primary failure category.

Recommended precedence:

```text
NO_PARENT_CANDIDATE
NO_DAUGHTER_A_CANDIDATE
NO_DAUGHTER_B_CANDIDATE
DAUGHTER_FILTERED
TEMPORAL_LOCALIZATION
TRACK_TOPOLOGY
DIVISION_CANDIDATE_NOT_GENERATED
DIVISION_CANDIDATE_NOT_SELECTED
SUCCESS
AMBIGUOUS
```

The exact implementation may use more detailed subcategories, but preserve a single `primary_failure_stage` field for aggregate analysis.

The precedence must be documented so a division with multiple problems is categorized consistently.

Also retain all individual boolean flags. The primary category must not destroy evidence of secondary failures.

---

# 9. Build the Audit Dataset

The principal artifact of Chapter 46 should be a table with **one row per GT division**.

Minimum useful schema:

```text
sample_id
division_id
parent_gt_id
daughter_a_gt_id
daughter_b_gt_id
parent_t
daughter_t

parent_ch34_distance
daughter_a_ch34_distance
daughter_b_ch34_distance
parent_detected_ch34
daughter_a_detected_ch34
daughter_b_detected_ch34

parent_survives_ch36
daughter_a_survives_ch36
daughter_b_survives_ch36

best_daughter_a_dt
best_daughter_b_dt
daughter_a_temporally_displaced
daughter_b_temporally_displaced

parent_track_id
daughter_a_track_id
daughter_b_track_id
daughter_a_track_predates_division
daughter_b_track_predates_division
daughters_share_track
track_topology_compatible

true_division_candidate_generated
true_candidate_score
true_candidate_rank
true_candidate_selected

primary_failure_stage
notes
```

Save the audit table as a CSV artifact if Kaggle storage/workflow permits.

---

# 10. Required Summary Tables

At minimum produce:

## A. Division counts by sample

| Sample | GT divisions | Fully auditable | Excluded/Ambiguous |
| --- | ---: | ---: | ---: |

## B. Stage survival

| Stage | Divisions reaching stage | Percent of total |
| --- | ---: | ---: |
| Parent + both daughters have Ch34 candidates | | |
| Both daughters survive Ch36 | | |
| Track topology compatible | | |
| Correct division candidate generated | | |
| Correct division selected | | |

## C. Primary failure distribution

| Failure stage | Count | Percent |
| --- | ---: | ---: |
| Candidate generation | | |
| Filtering | | |
| Temporal localization | | |
| Tracking topology | | |
| Division candidate generation | | |
| Division scoring/selection | | |
| Success | | |
| Ambiguous | | |

Do not populate these tables with hypothetical percentages. They must come from actual execution.

---

# 11. Required Visualizations

Keep visualization diagnostic rather than decorative.

Recommended plots:

1. bar chart of primary failure-stage counts;
2. stage-survival/funnel chart;
3. distribution of nearest daughter-candidate distances at `dt=-1, 0, +1`;
4. fraction of daughters whose best evidence occurs at each temporal offset;
5. candidate score/rank for retained vs filtered GT-near daughter candidates;
6. per-sample division success/failure counts.

If sample size is very small, favor tables and individual-event inspection over misleading statistical plots.

---

# 12. Sanity Check Against the Known Chapter 42 Event

The generalized audit should rediscover the known diagnostic event without special-case logic.

For sample `44b6_12dfb391`, verify that the audit can locate the known division involving:

- parent `172000000050` at t=66;
- daughter A `173000000050` at t=67;
- daughter B `173000000051` at t=67.

The audit should reproduce the qualitative findings from Chapters 42–44:

- one daughter had a substantially better candidate before aggressive filtering;
- another had stronger evidence at a nearby frame;
- ordinary re-tracking did not automatically produce correct division topology.

If the generalized pipeline does not reproduce those findings, stop and diagnose the discrepancy before trusting aggregate statistics.

---

# 13. Decision Rule for Chapter 47

Chapter 46 must end with a recommendation derived from the measured audit.

Use evidence like this:

### If candidate recall dominates

Chapter 47 should improve multiscale candidate generation or daughter-sensitive detection.

### If filtering dominates

Chapter 47 should develop division-preserving / context-aware candidate filtering.

### If temporal displacement is common

Chapter 47 should model division events over a temporal window rather than at a single frame.

### If track topology dominates

Chapter 47 should introduce division-aware tracking, birth hypotheses, or explicit branching in the association model.

### If correct division candidates are usually generated but ranked poorly

Chapter 47 should focus on division scoring/learning with the expanded positive dataset.

### If failure modes are strongly mixed

Prioritize the **earliest high-frequency failure stage**, because downstream models cannot recover information that has already been discarded.

Do not choose Chapter 47 before the audit results exist.

---

# 14. Notebook Completion Checklist

Chapter 46 is complete only when all of the following are true:

- [ ] multiple training samples are inspected, if the available data supports them;
- [ ] GT divisions are discovered programmatically;
- [ ] the known Chapter 42 division is recovered as a sanity check;
- [ ] Ch34 candidate coverage is measured for parent and daughters;
- [ ] Ch36 filtering survival is measured;
- [ ] daughter evidence at nearby frames is audited;
- [ ] Ch37 track topology is audited;
- [ ] Ch38 division candidate generation is audited;
- [ ] every auditable GT division receives a failure-stage classification;
- [ ] an audit dataset is saved;
- [ ] aggregate stage-survival and failure tables are produced;
- [ ] no hypothetical metrics are presented as results;
- [ ] the notebook states which bottleneck should become Chapter 47 and why.

---

# Expected Outcome

Chapter 46 should transform the project from:

> “We understand why one known division failed.”

into:

> “We know quantitatively where division events fail across the available training data, and therefore know which component should be improved next.”

That is the decision this experiment exists to make.