# Codex Handoff

Keep the newest entry first. Update after every substantive task.

## 2026-09-08 — Complete bounded ABPolicy Phase B audit

- **Date:** 2026-09-08
- **Task:** After the user formally accepted FlowMP as complete, answer only five ABPolicy questions: dense-to-sparse target construction, flow shapes, `n_ctrl=8`, training loss, and receding-horizon continuity/refitting.
- **What was done:** Audited the pinned source without installing dependencies or running Piper; traced configuration, HDF5 loader, B-spline fitter, CFM training, CondDiT conditioning, Euler sampling, asynchronous delay handling, prefix refit, reconstruction, and execution chunking; verified real local HDF5/action shapes and ran read-only isolated B-spline/refit checks; added one main-workspace audit document.
- **Important findings:** Training windows contain eight history plus 32 future absolute actions, `[B,40,7]`, compressed by cubic least-squares fitting to `[B,8,7]`. CFM operates directly on these 56 control coefficients with Gaussian `x0`, normalized control-point `x1`, linear `x_t`, target `x1-x0`, and image/qpos conditioning. Inference uses ten Euler steps, rebuilds `[8,7]` to `[40,7]`, refits only the first four coefficients to an `8+delay` executed prefix, discards that prefix, and queues 16 actions. These B-spline coefficients are not Timed Semantic Anchors.
- **Files changed:** Main workspace `docs/ABPOLICY_PHASE_B_AUDIT.md`; Context Repo `docs/CURRENT_STATUS.md` and `docs/CODEX_HANDOFF.md`. ABPolicy itself was not modified.
- **Commands/tests run:** Checked revision/worktree; read relevant config/training/model/loader/fitter/inference code; inspected all three local HDF5 files; decoded one real sample; executed `40×7 -> 8×7 -> 40×7`, an in-memory `n_ctrl=6` comparison, and weighted prefix refitting; checked official TorchCFM source for the dependency-owned CFM formula; confirmed the upstream worktree remained clean.
- **Artifacts/results:** Real sample shapes were qpos `[8,7]`, dense action `[40,7]`, and two images each `[1,3,480,640]`. Eight-control reconstruction RMSE was `0.01101999`; six-control RMSE was `0.01542381` for the same sample. Prefix RMSE decreased from `0.2000954` to `0.0023741`, while coefficients 4–7 remained exactly unchanged. These are code-path checks, not research results, so `EXPERIMENT_LOG.md` was not updated.
- **Unresolved problems:** The full realtime entrypoint imports two missing utility modules; model forward/checkpoint execution remains unverified by design; padding masks are not used in loss; large-delay bounds are not guarded; least-squares refitting does not prove the README's claimed strict continuity.
- **Recommended next step:** Perform only the bounded SanD-Planner Phase C audit, then stop reading references and implement the project's small V0.
- **Relevant git commit:** `Complete bounded ABPolicy audit` (this entry is contained in that commit).

## 2026-09-07 — Add and verify FlowMP dummy conditioning

- **Date:** 2026-09-07
- **Task:** Perform the required FlowMP controlled modification after Step 1, without changing upstream source or moving to ABPolicy.
- **What was done:** Added one isolated teaching script that changes the position field from `u_theta(t,x_t)` to `u_theta(t,x_t,c)`; constructed scalar route-ID conditions from two existing 3D trajectory files; propagated `c` through data selection, training, forward, and every RK4 evaluation; trained for 600 bounded CUDA steps; sampled both conditions from identical noise; appended the result to the existing FlowMP audit.
- **Important findings:** The condition changes the first-layer input width from four to five. Training conditions were `[64,1]`, broadcast to `[64,64,1]`, and the resulting network input was `[64,64,5]`. Same-noise `c=0`/`c=1` samples had mean paired output distance `2.586351`, and generated centroids closely matched their corresponding route-distribution centroids. This verifies condition plumbing, not obstacle conditioning or coherent trajectory generation.
- **Files changed:** Main workspace `scripts/learn_flowmp_condition.py` and existing `docs/FLOWMP_STEP1_AUDIT.md`; Context Repo `docs/CURRENT_STATUS.md` and `docs/CODEX_HANDOFF.md`. Generated artifacts are under `artifacts/flowmp_condition/`.
- **Commands/tests run:** Compiled the script; ran 600 CUDA optimizer steps with 512 real route examples subsampled to 64 points; performed paired same-noise sampling with 20 RK4 steps; asserted summary shape, loss reduction, condition-weight update, and nonzero paired condition effect; visually inspected the PNG.
- **Artifacts/results:** First/last 50-step mean losses were `4.911681` and `2.907413`; maximum condition-column weight update was `0.647869`; target/generated centroids agreed closely for both route labels. This is educational reference-code validation, not a V0 research experiment, so `EXPERIMENT_LOG.md` was not updated.
- **Unresolved problems:** The upstream pointwise architecture still does not model cross-point dependencies, and the dummy route ID is not scene geometry. FlowMP is awaiting the user's 10–15 minute mastery exercise before ABPolicy.
- **Recommended next step:** Give the user the focused four-location code walkthrough and mastery exercise; review their answer before allowing ABPolicy.
- **Relevant git commit:** `Record FlowMP Step 1 and conditioning audit` (this entry is contained in that commit).

## 2026-09-07 — Verify FlowMP Step 1 on the local 3D path

- **Date:** 2026-09-07
- **Task:** Restore context and perform only FlowMP Step 1: revision/environment audit, minimal 3D data load, tensor trace, forward/backward, sampling, and condition audit.
- **What was done:** Verified the pinned FlowMP revision; reused the existing Isaac Lab Python environment without starting Isaac; extracted the notebook path into an isolated main-workspace teaching script; loaded `trajs_3.npy`; ran one upstream CFM optimizer update for each of the seven fields; sampled with upstream RK4; saved a plot, arrays, and JSON summary; wrote a detailed Step 1 audit.
- **Important findings:** The real data shape is `[5120,256,9]` with `xyz + first spline derivatives + second spline derivatives`. Position CFM uses `x0=noise`, `x1=position target`, linear `xt`, target `u=x1-x0`, and MSE against `u_theta`. Obstacles, start, goal, and environment ID do not enter any network: the notebook loads obstacles but leaves them unused. Seven fields train independently; generated position/velocity/acceleration have no consistency coupling. Position processing is pointwise and lacks trajectory index. `FlowMotionPlanning3D.train` also breaks normal `model.eval()` behavior by overriding PyTorch's signature.
- **Files changed:** Main workspace `scripts/reproduce_flowmp_3d_minimal.py` and `docs/FLOWMP_STEP1_AUDIT.md`; Context Repo `docs/CURRENT_STATUS.md` and `docs/CODEX_HANDOFF.md`. Generated artifacts are under `artifacts/flowmp_step1/`.
- **Commands/tests run:** Checked Git revision/status; inspected all required context files and relevant FlowMP source/notebook cells; queried host/Python/PyTorch/CUDA/GPU; inspected every local 3D `.npy`; compiled and ran the minimal script on CUDA with batch 4, hidden 32, two sampled paths, 64 sample points, and four RK4 steps; visually inspected the generated PNG; directly reproduced the `model.eval()` `TypeError`.
- **Artifacts/results:** Seven deterministic one-step losses summed to `38.533102036`; the first position-network weight changed by `0.001000002`; sampling chain shape was `[2,5,64,9]`; all sampled values were finite. These are execution/debug evidence, not a research result, so `EXPERIMENT_LOG.md` was not updated.
- **Unresolved problems:** No trained-quality trajectory was claimed. The controlled dummy-condition modification, training/sampling propagation proof, and FlowMP mastery check remain pending.
- **Recommended next step:** Review and run the teaching script, then implement the isolated dummy-condition controlled modification; do not start ABPolicy yet.
- **Relevant git commit:** `Record FlowMP Step 1 and conditioning audit` (this entry is contained in that commit).

## 2026-09-05 — Define the current V0 sparse-planner milestone

- **Date:** 2026-09-05
- **Task:** Add a binding current-phase document that prevents work from expanding beyond the minimum sparse-planner hypothesis test.
- **What was done:** Added `CURRENT_PHASE.md`; fixed V0 to one box, vector conditioning, eight timed anchors for root/left foot/right foot, bounded reference-repo work, synthetic data, and two named experiments; updated session rules and current status.
- **Important findings:** Isaac Lab and downstream motion completion/control are not blockers for the standalone V0. V0 anchors are timed semantic positions/orientation values, not B-spline control points.
- **Files changed:** `AGENTS.md`, `README.md`, `docs/CURRENT_PHASE.md`, `docs/CURRENT_STATUS.md`, `docs/CODEX_HANDOFF.md`, and `docs/DECISIONS.md`.
- **Commands/tests run:** Read the required context documents and the complete user-provided current-phase specification; checked Markdown whitespace with `git diff --check`; inspected the final diff and repository status.
- **Artifacts/results:** Binding V0 scope and completion criteria documented; no model or research experiment was run.
- **Unresolved problems:** Phase A/B/C reference work, V0 implementation, EXP-001, and the optional multimodal experiment remain outstanding.
- **Recommended next step:** Perform only Phase A: run and document the smallest FlowMP 3D CFM example, then stop.
- **Relevant git commit:** `Define V0 sparse planner milestone` (this entry is contained in that commit).

## 2026-09-05 — Initialize persistent research context

- **Date:** 2026-09-05
- **Task:** Create a separate repository for durable research context and cross-session handoff.
- **What was done:** Initialized the repository; restored the canonical 2026-09-03 research mainline; added current status, roadmap, reference-repository roles, paper relationships, open questions, append-only templates, and agent rules.
- **Important findings:** The primary workspace is not itself a Git repository. Its local documentation reports a partially verified software baseline and a remaining Isaac Lab lifecycle blocker. The canonical mainline was recovered from the original saved user-provided text rather than reconstructed.
- **Files changed:** `.gitignore`, `AGENTS.md`, `README.md`, and all files under `docs/`.
- **Commands/tests run:** Inspected local Git roots/status/remotes; searched for and read the complete saved mainline; read the main workspace environment/status documents and the FlowMP, ABPolicy, and SanD-Planner READMEs; validated repository structure and Markdown; scanned tracked content and filenames for likely secrets and prohibited binary payloads; ran `git status`.
- **Artifacts/results:** This repository and its initial commit.
- **Unresolved problems:** GitHub remote is not configured; Isaac Lab lifecycle remains blocked; the first V0 research protocol and cross-component interface contracts are not frozen.
- **Recommended next step:** Provide the intended GitHub repository URL, then define the V0 experiment contract before starting planner implementation.
- **Relevant git commit:** `Initialize persistent research context` (this entry is contained in that initial commit).
