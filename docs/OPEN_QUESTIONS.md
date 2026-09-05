# Open Research Questions

These are unresolved questions or explicit hypotheses. They are not part of the canonical mainline unless the user later promotes them through an explicit decision.

## Representation

- What is the minimum body-part set needed for each traversal mode: root/feet only, or also pelvis and torso?
- Should anchor times be fixed, predicted, event-based, or represented by B-spline control points?
- How many anchors preserve feasibility without leaking dense-motion generation into the planner?
- Which coordinate frames and rotation representation make constraints stable and interpretable?

## Data and supervision

- Which source can produce the first trustworthy V0 expert trajectories: procedural templates, trajectory optimization, RL rollouts, or Kimodo/ARDY?
- How should dense motion be converted into sparse anchors without discarding collision-critical events?
- How should multimodal traversal choices be represented and balanced in the dataset?

## Model and objectives

- **Hypothesis:** Conditional Flow Matching or Rectified Flow will preserve useful multimodality better than deterministic regression for obstacle-dependent traversal strategies.
- Should collision/clearance enter training directly, through guidance/reranking, or both?
- Is a second-order flow field materially better for sparse anchors than a first-order model?

## Downstream feasibility

- Can ARDY or Kimodo reliably satisfy the proposed body-part constraints for Unitree G1?
- What constraint violations should trigger replanning rather than motion completion?
- How should tracker feasibility or failure signals train or rerank planner outputs?
- What horizon and refresh rate satisfy both generation latency and SONIC tracking stability?

## Evaluation and deployment

- What quantitative thresholds define success for V0 and justify moving from primitive geometry to ESDF/point clouds?
- Which baselines isolate the value of sparse whole-body anchors from the value of the generative model?
- How should uncertainty and safety margins propagate from partial perception to body-part clearance?
- What simulation safety and evidence gates are required before real-robot execution?
