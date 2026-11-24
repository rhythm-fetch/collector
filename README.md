# collector

Scheduled collector for the [Rhythm](https://github.com/Hiyabye/Rhythm) v2 notice feed. Owned by a machine account so GitHub Actions use stays separate from the app repository.

## Workflows

| Workflow | What it does |
| --- | --- |
| `collect.yml` | Every 30 minutes: crawls every public Sogang board into the state, embeds new notices through Workers AI and publishes the signed feed (`rhythm-collector --r2 state/ --publish feed/ --embed --articles 150 --history-minutes 15`). |
| `keepalive.yml` | Monthly: re-enables every workflow so GitHub's 60-day inactivity rule never stops the schedule. |

Each collect run downloads the newest state version from the private `rhythm-state` bucket (`state/<UTC time>.sqlite3.zst`, newest 48 kept), crawls, publishes the new feed files to `rhythm-feed` under `feed/`, uploads the state as a new version, and only then publishes the manifest. A run that fails or is killed before the upload changes nothing; files it uploaded are deleted by the next run. A failed board is listed in the run log and in the manifest's board health, and does not fail the run.

Secrets live in the `collector` environment (main only) and reach only the collect step. Run logs are public: the collector prints board names, counts and errors only.

## Operations

**First run.** Actions → Collect → Run workflow with `init` checked. It refuses when a state version already exists. Each run fetches every board's new notices first, then spends the rest of its first 15 minutes on history, in turns of at most 150 articles per board, so the full history (about 17,300 notices) and its embeddings (at most 1,000 per run) arrive over about a day of runs.

**Upgrade.** Production is built from the Rhythm commit in `RHYTHM_REF`. To upgrade, open a PR here that sets it to a newer `main` commit of Hiyabye/Rhythm. The first run after the merge builds and caches the binary (a few minutes).

**Canary stop.** When the run log says embedding stopped on canary drift, Workers AI vectors changed. After deciding the new vectors are acceptable (Hiyabye/Rhythm#166 D27), run once with `reset_canary` checked.

**Pause.** `gh workflow disable collect.yml -R rhythm-fetch/collector`, and `enable` to resume. A disabled workflow stays disabled until enabled again, or until the next keep-alive run re-enables it.

**Rollback.** Returns the state to an older version and restarts the feed from it (Hiyabye/Rhythm `docs/FEED.md`, "Restart"); clients follow on their next sync:

1. Pause the collector and wait for a running collect run to finish.
2. Delete every state version newer than the one to return to, with the R2 collector credentials in `CF_ACCOUNT_ID`, `R2_ACCESS_KEY_ID` and `R2_SECRET_ACCESS_KEY`:
   ```sh
   r2() { curl -fsS --aws-sigv4 "aws:amz:auto:s3" --user "$R2_ACCESS_KEY_ID:$R2_SECRET_ACCESS_KEY" "$@"; }
   base="https://$CF_ACCOUNT_ID.r2.cloudflarestorage.com/rhythm-state"
   r2 "$base?list-type=2&prefix=state/" | grep -o '<Key>[^<]*</Key>'
   r2 -X DELETE "$base/state/<version>.sqlite3.zst"
   ```
3. Enable the workflow again and run Collect with `reset_feed` checked. The feed continues after the published seqs with only a snapshot of the older state, which every client applies on its next sync. The old feed files are deleted a day later. A scheduled run that starts first only refuses to publish, without uploading anything. If the reset run fails, run it with `reset_feed` again.
