# Decisions

Append-only. Do not rewrite or delete accepted historical entries; supersede them with a new decision.

## DEC-001 — Scene-conditioned sparse whole-body planning is the research mainline

- **Date:** 2026-09-03
- **Decision:** Use a scene-conditioned planner to generate sparse spatiotemporal constraints for key G1 body parts; use a motion prior to complete dense motion and a tracker to execute it.
- **Motivation:** Separate high-level collision-aware traversal strategy from dense natural-motion synthesis and low-level physical control.
- **Alternatives considered:** Direct full 29-DoF trajectory generation; direct motor-action generation; text-to-G1 generation; training a new full motion prior.
- **Reason:** A low-dimensional, interpretable anchor representation focuses the learned planner on obstacle-dependent whole-body strategy while reusing existing motion and control priors.
- **Current status:** Active. Canonical details are locked in `CURRENT_MAINLINE.md` pending explicit user authorization to change them.

## DEC-002 — Keep research context separate from primary code

- **Date:** 2026-09-05
- **Decision:** Maintain this repository as a documentation-only persistent context and handoff repository.
- **Motivation:** Preserve one reviewable source of truth across human, Codex, and ChatGPT sessions without coupling research state to any implementation repository.
- **Alternatives considered:** Store context in the main code tree; rely on chat history alone.
- **Reason:** A small standalone Git history makes status, decisions, experiments, and handoffs durable and auditable while avoiding main-code changes.
- **Current status:** Active.

## DEC-003 — Freeze the V0 timed-semantic-anchor milestone

- **Date:** 2026-09-05
- **Decision:** For V0, use a single box and simple vector conditioning to generate eight fixed future times × nine semantic values (`root_x`, `root_y`, `root_yaw`, and XYZ for both feet). Treat these as Timed Semantic Anchors, not B-spline control points. Do not integrate Kimodo, ARDY, or SONIC before the first sparse-planner experiment succeeds.
- **Motivation:** Test whether the generative planner learns the relationship between obstacle geometry and sparse body trajectories without confounding perception, motion completion, control, or large-system integration.
- **Alternatives considered:** Full whole-body trajectories; B-spline control points; point-cloud/ESDF encoders; early downstream integration; broad reproduction of all three reference repositories.
- **Reason:** The 8×9 representation and synthetic single-box setting are sufficient for a falsifiable first experiment while keeping implementation and diagnosis bounded.
- **Current status:** Active for the current phase; supersede only with an explicit later decision.
