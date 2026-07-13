# easy-GTFS-RT

CI/CD-only repository that automates daily recording of Łódź's GTFS-RT `VehiclePositions`
feed and the reconstruction of a realized ("what actually happened") GTFS from it, for
[`GISBoost/easy-OTP`](https://github.com/GISBoost/easy-OTP)'s **Family A** tool
(`tools/family_a_reconstruction/`).

This repo contains **no application code** — only GitHub Actions workflows. Every workflow
here does an additional read-only `actions/checkout` of `GISBoost/easy-OTP` and runs the
`family_a` CLI from that checkout. The map-matching/interpolation logic itself is developed
and documented entirely in `easy-OTP`; nothing is duplicated here.

## Why this is a separate repo from `easy-OTP`

- `easy-OTP`'s GitHub Releases are the plugin's own version history (`easy_otp-0.3.5.zip`
  etc.). Dumping a GitHub Release into that list every single day would bury real plugin
  releases under daily data snapshots.
- This repo is **public**, which gets GitHub Actions unlimited minutes and artifact storage
  outside the default 500MB/repo cap that applies to private repos — meaningful for a job that
  records for up to 16 hours/day and produces a new GTFS build every evening.
- Secrets and repo variables here (`CALLMEBOT_*`, `LODZ_*`) are scoped to this repo only and
  are never needed by, or exposed to, `easy-OTP`.

## Pipeline, end to end

```
06:00 ───────────────────────────────────────────────────────────────── 22:00  (Europe/Warsaw)
 chunk1 [05:00–10:30]
    chunk2 [10:00–14:30]
                chunk3 [13:30–18:30]
                                chunk4 [18:00–22:00]
                                                      ↓ (workflow_run, right after chunk4 finishes)
                                                 build_and_notify
                                                      → GitHub Release (this repo)
                                                      → WhatsApp message (CallMeBot)
```

1. **Four recording workflows** (`family_a_record_chunk{1..4}.yml`) each run `family_a record`
   against the live Łódź `VehiclePositions` feed for a chunk of the day and upload the raw
   `.pb` snapshots as an artifact named `positions-<YYYY-MM-DD>-chunk{N}`. Chunk windows and
   durations are deliberately uneven and overlapping — they're built around the morning
   (7:00–10:00) and afternoon (14:00–18:00) transit peaks, with extra buffer where GitHub
   Actions' own schedule jitter could otherwise eat into a peak window. **Don't rebalance these
   into equal thirds/quarters** — the uneven split is intentional, see the comments in each
   chunk workflow.

   **GitHub's `schedule:` trigger is best-effort, not guaranteed**, and is documented to be
   most congested at the top of the hour — every `cron:` here deliberately avoids `:00`/`:30`
   for that reason. Even so, delays happen and aren't bounded: on 2026-07-12, chunk1's trigger
   (then still at `:00`) fired 2h31m late, confirmed via the GitHub API to be 100% scheduler
   dispatch delay (the job itself started ~3s after the run was created — zero runner-queue
   delay), and left a real gap in that morning's peak recording. chunk1's buffer was widened
   from 1h to 2h lead-in afterward (05:00 start instead of 06:00) to better absorb a repeat.
   This mitigates the risk, it does not eliminate it — a large enough delay can still leave a
   gap, and there is no configuration that guarantees otherwise on this trigger type. **A gap
   like that one is no longer silent**, though — see the "Actual recorded coverage" point below;
   it would now show up directly in the Release/WhatsApp output instead of requiring an Actions
   log check.
2. **`family_a_build_and_notify.yml`** ("Łódź — build") runs once a day, after the last chunk
   finishes:
   - Discovers **all** of that day's recording artifacts by **name prefix**
     (`positions-<date>-`), via the GitHub REST API — never a hardcoded chunk count. This is
     what lets the pipeline degrade gracefully (fewer chunks that day) or, later, pick up an
     on-demand custom recording under the same prefix, with zero workflow changes.
   - Reads each chunk's `recording.json` manifest (written by `family_a record` itself) to
     compute the **actual recorded coverage** — real, merged start/stop times (e.g.
     `08:32-11:52, 12:16-16:16`), not the nominal cron windows — so a schedule-trigger delay
     shows up as a visible gap in the Release notes and WhatsApp message instead of being
     assumed away.
   - Downloads a fresh static GTFS and runs `family_a match` (merging every discovered
     directory) then `family_a build`, producing P50 (median) and P85 (85th percentile)
     corrected GTFS zips.
   - Publishes the P50/P85 zips **and that day's static GTFS** as a GitHub Release in **this**
     repo (tag `lodz-realized-<date>`), and sends one WhatsApp message via
     [CallMeBot](https://www.callmebot.com/) with the direct P50/P85 download links and the
     recorded-coverage summary — or a failure message if anything broke, so a bad day doesn't
     just show up as silence.

### Why the build trigger isn't a simple fixed-time schedule

`build_and_notify` triggers primarily via `workflow_run` on chunk4's completion — it fires the
moment chunk4 *actually* finishes, whatever real delay that involved that day, instead of
guessing a fixed time buffer after its nominal end (a fixed buffer was tried first and found to
be too tight against GitHub's observed schedule jitter). A few things layered on top of that,
worth knowing before touching this workflow:

- The `workflow_run` trigger only fires a build for chunk4's **scheduled** runs (gated by
  `github.event.workflow_run.event == 'schedule'`), not manual/test runs — otherwise testing
  chunk4 by hand would trigger a premature build with an incomplete day.
- `build_and_notify` has **no time-based fallback of its own** anymore. A `schedule:` 23:45 CEST
  end-of-day cron used to fill that role, but it was removed on 2026-07-13 after GitHub's
  schedule-queue delay pushed one such run past local midnight: it computed "today" as the
  *next* day (whose chunks hadn't recorded yet) and failed with a spurious "no artifacts found"
  right after `workflow_run` had already published that day's release successfully. The cron
  also routinely fired too late to be useful even when it didn't fail outright. If `workflow_run`
  doesn't fire for a given day, run **`family_a_build_and_notify_on_demand.yml`** by hand
  instead — same steps, `workflow_dispatch`-only, same `date` override.
- An idempotency guard (`gh release view` before doing any work) makes a same-day
  double-trigger a clean no-op instead of a duplicate-release-tag failure, and
  `concurrency: group: family-a-build-and-notify` (queue, don't cancel) — shared with the
  on-demand workflow too — closes the small check-then-act race window between two
  near-simultaneous triggers.

## Repository configuration

Set under **Settings → Secrets and variables → Actions**, in this repo (not `easy-OTP`'s):

**Variables**
| Name | Value |
|---|---|
| `LODZ_STATIC_GTFS_URL` | `https://otwarte.miasto.lodz.pl/wp-content/uploads/2025/06/GTFS.zip` |
| `LODZ_VEHICLE_POSITIONS_URL` | `https://otwarte.miasto.lodz.pl/wp-content/uploads/2025/06/vehicle_positions.bin` |

**Secrets**
| Name | Value |
|---|---|
| `CALLMEBOT_PHONE` | Your WhatsApp number, with country code (e.g. `+48...`) |
| `CALLMEBOT_APIKEY` | Issued by CallMeBot after the one-time opt-in below |

**CallMeBot one-time opt-in** (per phone number, done once):
1. Add the CallMeBot contact in WhatsApp: `+34 644 51 71 92`.
2. Send it the message: `I allow callmebot to send me messages`.
3. It replies with your `apikey` — put that in the `CALLMEBOT_APIKEY` secret above.

## Manual testing

Every workflow here also has a `workflow_dispatch` trigger, so nothing requires waiting for a
cron to fire:

- Run any `family_a_record_chunk{1..4}.yml` by hand from the Actions tab to produce a test
  artifact for today.
- Run `family_a_build_and_notify.yml` or `family_a_build_and_notify_on_demand.yml` by hand; both
  accept an optional `date` input (`YYYY-MM-DD`) to target a specific day's recording instead of
  today — useful when running shortly after local midnight, when "today" would otherwise no
  longer match last night's chunk artifacts. The on-demand version is also the way to trigger a
  build at all if `workflow_run` never fired for that day (see above).

A manually-triggered build does **not** get skipped by the idempotency guard unless a Release
for that date already exists.

## Known, accepted trade-offs

- **CallMeBot is an unofficial, volunteer-run API** (no SLA/guarantee). If it's down, the
  WhatsApp notification silently fails to send — but the GitHub Release is still published
  either way. The Release is the source of truth; WhatsApp is a convenience notification on
  top of it, not the delivery mechanism itself.
- **DST is not auto-compensated.** All `cron:` schedules in this repo are UTC and hand-tuned
  for the currently active offset (see the comment above each `cron:` line for the CET/CEST
  equivalent) — they need a manual one-hour edit after each DST switch (last Sunday of
  March/October).
- **The static GTFS is downloaded fresh every build**, never cached long-term — Łódź's open
  data feed has been observed to republish with a shifted `trip_id` generation between
  recording sessions, and reusing a stale static feed against newer recordings silently
  produces a suspiciously high `unknown_shape` reject count in `match`'s output instead of an
  error. If that ever shows up, it's this, not a pipeline bug. Since 2026-07-13, the exact
  static GTFS used for a given day's `match`/`build` is archived as a third asset on that day's
  Release (`lodz_static_gtfs_<date>.zip`) — precisely so a future comparison between two days'
  realized GTFS isn't invalidated by this same drift.
- **Recorded-coverage times come from each chunk's own `recording.json` manifest**
  (`started_at`/`stopped_at`, written by `family_a record`), not from a live poll log — so a
  chunk that ran but produced zero/`failed` snapshots throughout would still report as
  "covered" for that time range. This matches what the coverage summary is meant to answer
  ("was a recording job actually running then"), not "how much clean data came back" — the
  latter is what `match`'s own reject-count output is for.

## Where the actual logic lives

This repo only orchestrates. The `family_a` CLI (`record` / `match` / `build`), its algorithm,
and its own documentation live in `GISBoost/easy-OTP`, under
[`tools/family_a_reconstruction/`](https://github.com/GISBoost/easy-OTP/tree/main/tools/family_a_reconstruction)
(see that folder's own `README.md` for the CLI contract this repo's workflows call into, and
`docs/prd/PR_easy-OTP_v07.md` for the full design rationale behind this pipeline, milestones
FA-7/FA-8/FA-9).
