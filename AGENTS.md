# collector

This repository holds only GitHub Actions workflows; the collector's code is `crates/collector` in Hiyabye/Rhythm, whose AGENTS.md owns its behavior.

- `collect.yml` is the production run. It builds `rhythm-collector` from the Hiyabye/Rhythm commit in `RHYTHM_REF` (a `main` commit; production changes only when it moves) and caches one binary per commit. Its arguments, 30-minute cron and `--articles 150` budget are user decisions (2026-11-18, Hiyabye/Rhythm#176); change them only with approval.
- Secrets live in the `collector` environment (main only): `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `CF_ACCOUNT_ID`, `CF_AI_TOKEN`, `RHYTHM_FEED_SIGNING_KEY`. Pass them only to the steps that run the collector (the crawl and the `--meal-results` call, which needs no `CF_AI_TOKEN`), never to build or OCR steps, and never print them.
- The dormitory menu (Hiyabye/Rhythm#318): the crawl writes menu images with `--meal-images "$RUNNER_TEMP/meals"`; only when that folder holds `*.json` jobs do the OCR steps run (`meal_ocr.py` and its hashed lockfile sparse-checked-out from `RHYTHM_REF`, models cached under `$RUNNER_TEMP/paddlex`), then `rhythm-collector` with `--meal-results` on a fresh state path. The OCR's rules live in Hiyabye/Rhythm `crates/collector/AGENTS.md` "Dormitory menu". Run logs are public.
- `keepalive.yml` re-enables every workflow monthly; keep it while any workflow is scheduled.
- Every run of `collect.yml` changes production state (R2 state and the public feed). Dispatching it, changing its inputs or moving `RHYTHM_REF` needs approval. Live checks of collector changes run from Hiyabye/Rhythm against test prefixes, never through this workflow.
- README.md holds the maintainer's operations (first run, upgrade, canary reset, pause, rollback); keep it in step with `collect.yml`.
