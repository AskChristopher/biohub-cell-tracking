# Chapter 48 — Division-Aware Tracking at Assignment Time

**Status:** Ready for first Kaggle experiment  
**Type:** Controlled tracking-architecture experiment  
**Depends on:** Chapters 35, 46, 46A V12, and 47

## Why Chapter 48 Exists

Chapter 46 showed that the dominant measured division failure is tracking topology: 8 of 13 audited binary GT divisions failed primarily because daughters were absorbed into continuation trajectories. Chapter 47 then tested the cheapest architectural correction: infer division splits after Chapter 37 from finished-track geometry. That experiment completed technically but failed its false-positive sanity gate, generating 5,375 candidate splits and selecting the maximum allowed 75 divisions across three samples that contain only 13 known binary divisions.

The conclusion is architectural. By the time Chapter 37 trajectories are finalized, too much assignment-time information has been collapsed. Chapter 48 therefore moves division reasoning upstream into temporal association.

## Research Question

> Can a tracker distinguish ordinary one-to-one continuation from a one-parent→two-daughter division hypothesis using evidence available at assignment time, while keeping predicted divisions bounded?

## Hypothesis

The historical Chapter 35 tracker is continuation-oriented: it uses physical-coordinate scaling, constant-velocity prediction, Hungarian one-to-one assignment, distance gating, and short-gap closing. A true division violates the one-parent→one-child assumption.

If division is represented as an explicit competing assignment hypothesis before ordinary assignments are finalized, then true branch events should produce stronger local evidence than the large number of branch-like patterns observed after Chapter 37. The tracker should therefore be more selective than Chapter 47.

## Experimental Principle

Chapter 48 is not a detector, ranker, or filter experiment.

Keep fixed:

- Chapter 34 top-200 detections
- physical coordinate convention: z=1.625, y=0.40625, x=0.40625
- historical Chapter 35 continuation gate: max link distance 6.0
- historical continuation cost ingredients: rank penalty, predicted-distance penalty, and velocity-change penalty
- same three V12 samples for the first diagnostic run

Change only the temporal-assignment decision by allowing a division hypothesis to compete with continuation.

Ground truth must not be used to choose division events or tune thresholds in the first run.

## Inputs

The first Kaggle notebook should require only the existing `46DataV12` input, specifically:

- `chapter34_top200_detections_multisample.csv`

It may also use `chapter36_filtered_detections_multisample.csv` and `chapter37_refined_track_nodes_multisample.csv` for descriptive comparison, but they are not required to make Chapter 48 assignment decisions.

This deliberately avoids reopening GEFF/Zarr ground truth in the first run and avoids another Kaggle dependency problem.

## Historical Chapter 35 Baseline

Chapter 35 uses:

- constant-velocity prediction
- Hungarian one-to-one assignment
- physical-coordinate distance gating
- `max_link_distance = 6.0`
- `max_gap = 2`
- `rank_penalty_weight = 0.35`
- `pred_dist_penalty_weight = 0.15`
- `velocity_change_penalty_weight = 0.25`

Chapter 48 should preserve ordinary continuation behavior as much as possible and add a division hypothesis before finalizing conflicting local assignments.

## Competing Hypotheses

For an active parent track at frame t, inspect detections at t+1.

### H0 — ordinary continuation

One parent continues to one detection.

Evidence:

- distance from predicted parent position
- rank quality
- change in velocity
- historical continuation assignment cost

### H1 — binary division

One parent terminates and two detections at t+1 begin daughter tracks.

Evidence should be computed only from predicted data:

- both daughters individually satisfy a tight parent-to-candidate gate
- both are plausible relative to the parent's predicted position
- daughters are spatially distinct from each other
- both candidates are competitive with the parent's best continuation
- pair cost is better than treating one candidate as continuation and the other as an unrelated birth by a fixed margin
- the parent has sufficient history for a meaningful velocity prediction

## Conservative Bias

Chapter 47 proved that plausible branch geometry is common. Therefore ordinary continuation must win by default.

A division may be accepted only when:

1. two daughter candidates simultaneously pass the parent gate;
2. their pair geometry is plausible;
3. both are competitive continuation candidates;
4. the division score clears a fixed conservative threshold;
5. the division hypothesis beats the ordinary-continuation alternative by a fixed margin;
6. neither daughter is already consumed by a better non-division assignment.

The first run must not tune these thresholds against the 13 GT events.

## First Experiment — Structural Smoke Test

Run the same three V12 samples and compare:

- ordinary baseline continuation assignments
- division-aware assignments
- number of division hypotheses generated
- number accepted
- accepted divisions per sample
- parent→daughter distances
- daughter separation
- continuation cost vs division cost/margin
- number of ordinary assignments displaced by division decisions
- track count and track-length distribution

The immediate question is selectivity, not final GT accuracy.

Chapter 47 selected 75 divisions because all three samples hit a 25-event cap. Chapter 48 should produce a dramatically smaller, naturally bounded number without relying on an artificial per-sample cap.

## Initial Sanity Gate

For the three V12 samples, the first Chapter 48 run passes the structural sanity gate only if:

- it does not require a hard per-sample division cap to remain bounded;
- predicted divisions are substantially below Chapter 47's 75 selected events;
- ordinary continuation remains the overwhelming majority of assignments;
- accepted division pairs have visibly stronger evidence than rejected pairs;
- no GT information was used during assignment.

A predicted count near the known 13 events would be encouraging but is not itself a tuning target.

## Required Outputs

- `chapter48_assignment_candidates.csv`
- `chapter48_accepted_divisions.csv`
- `chapter48_track_nodes.csv`
- `chapter48_continuation_edges.csv`
- `chapter48_lineage_edges.csv`
- `chapter48_assignment_log.csv`
- `chapter48_sample_summary.csv`
- `chapter48_structural_summary.csv`
- `chapter48_outputs.zip`

Every accepted division should preserve the evidence that caused it to win so the result remains auditable.

## Decision After First Run

### If Chapter 48 is naturally selective

Preserve the fixed rule, then add the Chapter 46 GT evaluation boundary and measure the same 13 events without changing thresholds.

### If Chapter 48 still overgenerates divisions

Do not threshold-tune against the 13 events. Diagnose which assignment-time evidence fails to separate divisions from ordinary crowded tracking. The next experiment should add a genuinely new signal, not merely tighten a number until the training examples fit.

### If Chapter 48 selects almost nothing

Inspect the rejected high-ranking hypotheses and determine whether the historical top-200 detections and frame-to-frame representation expose enough simultaneous daughter evidence at t+1. A short temporal look-ahead may be justified only if the first run shows that one-frame evidence is systematically insufficient.

## Portfolio Significance

Chapter 48 is a model-architecture response to a measured negative result. The project does not keep tuning a failed post-hoc heuristic. It uses Chapter 47 to identify where information is being lost and moves the biological branching decision into the stage where competing temporal assignments are still explicit.
