# Current Status

Last updated: 2026-09-07

## Current phase

V0: validate the minimal `simple box condition → CFM → 8×9 Timed Semantic Anchors → visualization` loop. `CURRENT_PHASE.md` is the binding execution scope.

## Completed

- Persistent context repository and canonical research mainline established.
- V0 output fixed to 8 future times × 9 semantic values: root, left foot, and right foot.
- FlowMP, ABPolicy, and SanD-Planner are locally available as bounded references.
- Separate research environments and limited CUDA/import/simulator checks are documented in the main workspace.
- FlowMP Step 1 is verified at revision `81e70afb58073182a854f12c858242e65713eb6e`: real 3D data load, all seven field forward/backward updates, RK4 sampling, exact CFM tensor trace, and the absence of explicit obstacle/start/goal conditioning. An isolated teaching script, audit, summary, arrays, and figure are saved in the main workspace.

## In progress

- Phase A remains active. Step 1 reproduction/audit and the isolated dummy route-condition controlled modification are complete. FlowMP is paused for the user's mastery exercise and review; obstacle conditioning itself remains pending for the project's own V0 rather than this upstream reference.

## Next

- Complete the FlowMP mastery exercise and review before ABPolicy.
- Complete the bounded ABPolicy and SanD code audits only after FlowMP acceptance.
- Implement the small synthetic-data V0 in `src/sparse_planner/`.
- Run EXP-001: obstacle height versus predicted maximum foot-anchor height.

## Blockers

- No blocker is currently established for the standalone V0 experiment.
- Isaac Lab lifecycle and downstream ARDY/Kimodo/SONIC integration remain unresolved but are explicitly non-blocking for V0.
