# Current Status

Last updated: 2026-09-08

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

## In progress

- Phase C is next: perform only the bounded SanD-Planner audit of scene encoding, goal conditioning, sparse target, safety evaluation, and replanning. Do not install or run the full Isaac/navigation benchmark.

## Next

- Complete the bounded SanD-Planner audit.
- Implement the small synthetic-data V0 in `src/sparse_planner/`.
- Run EXP-001: obstacle height versus predicted maximum foot-anchor height.

## Blockers

- No blocker is currently established for the standalone V0 experiment.
- Isaac Lab lifecycle and downstream ARDY/Kimodo/SONIC integration remain unresolved but are explicitly non-blocking for V0.
