# Codex working agreement

Before every session, read `docs/CURRENT_MAINLINE.md`, `docs/CURRENT_STATUS.md`, and `docs/CODEX_HANDOFF.md` first. Do not modify `CURRENT_MAINLINE.md` without the user's explicit request.

After substantive work, update `CURRENT_STATUS.md` and `CODEX_HANDOFF.md`. Append to `EXPERIMENT_LOG.md` when an experiment was run and to `DECISIONS.md` when an important research or architecture decision was made. Never rewrite prior experiment or decision entries.

Mark every unverified claim as **hypothesis** or **unverified**. Claims such as “runs successfully” or “verified” require an actual command plus a log, artifact, or test result.

Do not store model checkpoints, datasets, Isaac assets, large binary files, secrets, or API keys in this repository.
