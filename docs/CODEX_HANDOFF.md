# Codex Handoff

Keep the newest entry first. Update after every substantive task.

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
