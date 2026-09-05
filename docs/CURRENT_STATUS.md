# Current Status

Last updated: 2026-09-05

## Current phase

V0: validate the minimal `simple box condition → CFM → 8×9 Timed Semantic Anchors → visualization` loop. `CURRENT_PHASE.md` is the binding execution scope.

## Completed

- Persistent context repository and canonical research mainline established.
- V0 output fixed to 8 future times × 9 semantic values: root, left foot, and right foot.
- FlowMP, ABPolicy, and SanD-Planner are locally available as bounded references.
- Separate research environments and limited CUDA/import/simulator checks are documented in the main workspace.

## In progress

- Phase A is the active task: minimal FlowMP reproduction and audit of loss, sampling, tensor shapes, conditioning, and obstacle input. No execution evidence is recorded yet.

## Next

- Complete the bounded ABPolicy and SanD code audits.
- Implement the small synthetic-data V0 in `src/sparse_planner/`.
- Run EXP-001: obstacle height versus predicted maximum foot-anchor height.

## Blockers

- No blocker is currently established for the standalone V0 experiment.
- Isaac Lab lifecycle and downstream ARDY/Kimodo/SONIC integration remain unresolved but are explicitly non-blocking for V0.
