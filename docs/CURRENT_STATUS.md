# Current Status

Last updated: 2026-09-10

## Current phase

V0: validate the minimal `simple box condition → CFM → 8×9 Timed Semantic Anchors → visualization` loop. `CURRENT_PHASE.md` is the binding execution scope.

## Completed

- Persistent context repository and canonical research mainline established.
- V0 output fixed to 8 future times × 9 semantic values: root, left foot, and right foot.
- FlowMP, ABPolicy, and SanD-Planner are locally available as bounded references.
- Separate research environments and limited CUDA/import/simulator checks are documented in the main workspace.
- FlowMP Step 1 is verified at revision `81e70afb58073182a854f12c858242e65713eb6e`: real 3D data load, all seven field forward/backward updates, RK4 sampling, exact CFM tensor trace, and the absence of explicit obstacle/start/goal conditioning. An isolated teaching script, audit, summary, arrays, and figure are saved in the main workspace.
- The user formally accepted the bounded FlowMP learning phase as complete after the mathematical, source, second-order, sampling, and conditioning walkthroughs.
- ABPolicy Phase B is complete at revision `86e977aa7c11e826790a2844ab667eb38c64637e`: real HDF5 shapes, `40×7 -> 8×7 -> 40×7` B-spline conversion, CFM tensor path, Euler inference, and prefix refitting were audited. The upstream repository remained unchanged.
- Bounded HRI Flow Matching vs MotionFM base-model evaluation is complete at revisions `516e8e18875b27741bdbdb8a252e904032c90723` and `08eb88dda8c096396132b9d91deae322f16aa224`. The technical recommendation is **A: HRI ConditionalUnet1D + vector global_cond + independent CFM**, with its example sampling noise corrected in any future fork. Both isolated cores passed random `[2,8,9]` forward/loss/backward/one-update/four-step Euler checks; native full entrypoints and real-data execution remain **unverified**. MotionFM used explicit text/SMPL boundary test doubles. Report: main workspace `docs/FLOW_BASE_MODEL_COMPARISON.md`; evidence: `artifacts/flow_base_audit/`. Both upstream worktrees remained clean. This is an evaluation recommendation, not a completed fork or V0 research result.

## Base-model evaluation evidence

- **Final recommendation:** **A. Recommend HRI Flow Matching**, specifically `ConditionalUnet1D + vector global_cond/FiLM + independent CFM`. Its native `[B,8,9]` interface, temporal convolutions and existing vector-conditioning path need fewer changes than MotionFM's human/text-oriented model interface. This is a candidate recommendation; no fork or V0 implementation exists as a result of this audit.
- **Verified-static:** Pinned source traces cover dataset/target tensors, backbone, temporal modeling, conditions, FM formulas, samplers, imports, licenses and minimal adaptation maps for both candidates.
- **Verified-core-runtime:** Both isolated reduced cores passed forward, FM loss, backward, one optimizer update and four Euler steps; finite gradients/samples and external `[2,8,9]` outputs were recorded. HRI's test used corrected Gaussian sampling; MotionFM used explicit CLIP/text-feature and SMPL boundary doubles. These are code-path checks, not research or comparative-quality results.
- **Unverified:** Full native training/inference entrypoints, real original-data samples, full original model configurations, checkpoints, convergence, sample quality, physical validity and V0 obstacle-conditioned behavior. Native model import probes failed on missing zarr (HRI) and clip (MotionFM).
- **Report:** [FLOW_BASE_MODEL_COMPARISON.md](FLOW_BASE_MODEL_COMPARISON.md) is the synchronized full report. Original: `/opt/ext_disk/pub/MYB/Sparse Whole-Body Planning/docs/FLOW_BASE_MODEL_COMPARISON.md`; logs and source hashes remain under the main workspace's `artifacts/flow_base_audit/`.

## In progress

- No implementation was started during the completed base-model evaluation. Stop at the user's requested audit boundary.
- The earlier bounded SanD-Planner Phase C audit remains outstanding; it was not performed or implicitly waived by this task.

## Next

- The audit and its Context Repo synchronization are complete. The immediate next step is to review the HRI recommendation with the user and wait for the next explicit execution instruction; do not start a fork or V0 automatically.
- The previously planned SanD-Planner bounded audit, small synthetic-data V0, and EXP-001 remain pending. Preserve `CURRENT_PHASE.md` and the research mainline.

## Blockers

- No blocker is currently established for the standalone V0 experiment.
- Isaac Lab lifecycle and downstream ARDY/Kimodo/SONIC integration remain unresolved but are explicitly non-blocking for V0.
