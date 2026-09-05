# Reference Repositories

These repositories are references and candidate components, not proof that their methods integrate with this project.

## FlowMP

- **Repository:** https://github.com/mkhangg/flow_mp
- **Pinned local revision observed:** `81e70afb58073182a854f12c858242e65713eb6e`
- **Role here:** Primary methodological reference for conditional flow matching over trajectories, especially second-order/acceleration-aware motion fields and multimodal planning.
- **What to reuse/compare:** Flow-matching formulation, trajectory-generation baseline, conditioning design, and smoothness/dynamic-feasibility metrics.
- **What it does not provide:** A G1 whole-body sparse-anchor representation, scene encoder for humanoid traversal, motion-prior completion, or tracker integration. Those mappings are project research work and remain **unverified**.

## ABPolicy

- **Repository:** https://github.com/teee000/ABPolicy-code
- **Pinned local revision observed:** `86e977aa7c11e826790a2844ab667eb38c64637e`
- **Role here:** Reference for representing actions with B-spline control points and for asynchronously refreshing/stitching trajectory chunks during continuous execution.
- **What to reuse/compare:** Compact smooth trajectory parameterization, past/future chunk conditioning, continuity-constrained refitting, and latency-aware receding-horizon execution.
- **What it does not provide:** Whole-body humanoid traversal; the upstream system targets a 6-DoF manipulation arm. Transfer to sparse G1 body anchors is a **hypothesis**, not a validated integration.

## SanD-Planner

- **Repository:** https://github.com/WangJinCheng1998/sandplanner
- **Pinned local revision observed:** `415589ef81970d0ec055c6eaeef5edcdf1cf6c46`
- **Role here:** Strong scene-conditioned local-planning reference connecting onboard depth, compact B-spline control points, multimodal generation, and ESDF-based collision/goal selection.
- **What to reuse/compare:** Scene encoder patterns, B-spline output/evaluation pipeline, candidate selection, collision/clearance metrics, and navigation benchmark protocol.
- **What it does not provide:** Whole-body humanoid anchors or dense G1 motion completion/tracking. Extending its mobile-robot trajectory abstraction to body-part constraints is core project research.
