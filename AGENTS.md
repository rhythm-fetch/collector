# collector

This repository holds only GitHub Actions workflows; the collector's code is `crates/collector` in Hiyabye/Rhythm, whose AGENTS.md owns its behavior.

- `collect.yml` is the production run. It builds `rhythm-collector` from the Hiyabye/Rhythm commit in `RHYTHM_REF` (a `main` commit; production changes only when it moves) and caches one binary per commit. Its arguments, 30-minute cron and `--articles 150` budget are user decisions (2026-11-18, Hiyabye/Rhythm#176); change them only with approval.
- Secrets live in the `collector` environment (main only): `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `CF_ACCOUNT_ID`, `CF_AI_TOKEN`, `RHYTHM_FEED_SIGNING_KEY`. Pass them only to the step that runs the collector, never to build steps, and never print them. Run logs are public.
- `keepalive.yml` re-enables every workflow monthly; keep it while any workflow is scheduled.
- Every run of `collect.yml` changes production state (R2 state and the public feed). Dispatching it, changing its inputs or moving `RHYTHM_REF` needs approval. Live checks of collector changes run from Hiyabye/Rhythm against test prefixes, never through this workflow.
- README.md holds the maintainer's operations (first run, upgrade, canary reset, pause, rollback); keep it in step with `collect.yml`.
