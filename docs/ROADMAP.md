# Roadmap

This is an execution roadmap, not evidence of completed work. Completion claims belong in `CURRENT_STATUS.md` and require supporting evidence.

## Phase 0 — Reproducible foundation

- Resolve finite Isaac Lab headless lifecycle behavior.
- Verify Unitree G1 spawn/step with a bounded command and retained log.
- Specify interfaces among sparse planner, ARDY/Kimodo, and SONIC.

## Phase 1 — V0 hypothesis test

- Condition on one ground-truth primitive obstacle.
- Start with a fixed, small set of root and foot anchors; expand only if needed.
- Build a deterministic baseline and a minimal multimodal generative baseline.
- Evaluate collision, clearance, goal success, completion feasibility, and inference time—not trajectory MSE alone.

## Phase 2 — Representation and completion

- Compare anchor counts, body-part subsets, and timing representations.
- Test whether Kimodo/ARDY can satisfy planner constraints.
- Add pelvis/torso constraints based on measured failure modes.

## Phase 3 — Richer scene conditioning

- Progress from multiple primitives to occupancy/voxel/ESDF, then partial point clouds.
- Measure generalization to unseen obstacle geometry and traversal strategies.

## Phase 4 — Closed-loop execution

- Add receding-horizon regeneration.
- Test dense-motion tracking with SONIC/GEAR-SONIC in simulation.
- Evaluate whole-body collision, falls, traversal success, and latency end to end.

## Phase 5 — Real sensing and robot

- Integrate real depth/point-cloud observations only after simulation criteria are met.
- Establish explicit safety gates before any Unitree G1 motion.
