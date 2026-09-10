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

## HRI Flow Matching — base-model candidate

- **Repository:** [HRI-EU/flow_matching](https://github.com/HRI-EU/flow_matching).
- **Pinned audited revision:** `516e8e18875b27741bdbdb8a252e904032c90723`.
- **Role here:** Base-model candidate only for `condition → FM → 8×9 Timed Semantic Anchors`; the completed comparison recommends its `ConditionalUnet1D + global_cond/FiLM + independent CFM` core for a future fork. No project fork has been started.
- **Evidence:** Static tensor/condition/loss/sampler audit and isolated reduced-core smoke pass. Full native entrypoints, original-data execution and planner quality remain **unverified**. Correct the examples' uniform sampling noise to Gaussian in any future adaptation.
- **Report:** [FLOW_BASE_MODEL_COMPARISON.md](FLOW_BASE_MODEL_COMPARISON.md); source/log artifacts remain in the main workspace.
- **Scope constraint:** This candidate evaluation does not change `CURRENT_MAINLINE.md` or `CURRENT_PHASE.md`, and does not authorize V0 implementation.

## MotionFM — base-model candidate

- **Repository:** [dongzhuoyao/motionfm](https://github.com/dongzhuoyao/motionfm).
- **Pinned audited revision:** `08eb88dda8c096396132b9d91deae322f16aa224`.
- **Role here:** Alternative base-model candidate only; MDM-derived temporal Transformer with a self-written FM loss and ODE sampler. It is viable for eight frame tokens but was not the final recommendation because its vector-condition API, 4D layout and text/SMPL lifecycle require more adaptation.
- **Evidence:** Static tensor/condition/loss/sampler audit and isolated reduced-core smoke pass with explicitly documented text/SMPL boundary doubles. Full native entrypoints, original-data execution and planner quality remain **unverified**.
- **Report:** [FLOW_BASE_MODEL_COMPARISON.md](FLOW_BASE_MODEL_COMPARISON.md); no pretrained human model, dataset or SMPL asset is adopted by this reference listing.
- **Scope constraint:** This candidate evaluation does not change `CURRENT_MAINLINE.md` or `CURRENT_PHASE.md`, and does not authorize a fork or V0 implementation.
