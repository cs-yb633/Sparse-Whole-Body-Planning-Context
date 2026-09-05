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
