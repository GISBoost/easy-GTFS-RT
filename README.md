# easy-GTFS-RT

CI/CD-only repository that automates daily recording of GTFS-RT `VehiclePositions` feeds and
the reconstruction of a realized ("what actually happened") GTFS from them, for
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
- Secrets here (`CALLMEBOT_*`) are scoped to this repo only and are never needed by, or exposed
  to, `easy-OTP`.

## Multi-city support (TX-8)

The phone-build workflow (`family_a_build_and_notify_from_phone.yml`) and the healthcheck
(`family_a_phone_healthcheck.yml`) support recording/building more than one city from the same
phone in parallel. **`config/cities.json`** (versioned in this repo) is the single source of
truth for which cities are configured — each key is a city id (e.g. `"lodz"`) with a
`display_name` and a `static_gtfs_url`. Release tags and filenames are all prefixed with the city
id (`<city>-realized-<date>-phone`, `positions-raw-<city>-<date>`, etc.) instead of the old
hardcoded `lodz-`/`Łódź` naming. Adding a city is a `config/cities.json` entry here plus a
matching per-city config on the phone (`easy-OTP`'s `scripts/termux/README.md` has the runbook)
— no other workflow changes needed.

## Current approach: recording via a phone (Termux, TX-*)

**As of 2026-07-14, this is the only actively producing pipeline.** Recording moved off
GitHub Actions runners entirely and onto Michał's Android phone, running continuously via
Termux instead of in scheduled chunks — see `GISBoost/easy-OTP`'s
[`scripts/termux/README.md`](https://github.com/GISBoost/easy-OTP/blob/main/scripts/termux/README.md)
for the phone-side half of this pipeline (recording, self-healing, daily upload, one process per
city). This repo only holds the GitHub-side half:

```
Phone (Termux, easy-OTP)                          This repo (easy-GTFS-RT)
-------------------------                          -------------------------
06:00-22:00  continuous recording (self-healing),
             one process per configured city
22:10        sweep_and_upload.sh loops every city:
             uploads today's raw recordings as a
             pre-release (positions-raw-<city>-
             <date>), then fires a
             repository_dispatch event per city
             that had new data          ----->
                                                    (seconds later, per city) family_a_build_and
                                                    _notify_from_phone.yml downloads that city's
                                                    raw release, builds P50/P85 corrected GTFS,
                                                    publishes "<city>-realized-<date>-phone",
                                                    WhatsApp notify
                                                    22:15  schedule fallback - expands to every
                                                           city in config/cities.json; each
                                                           city's idempotency guard makes an
                                                           already-built one a fast no-op
                                                    21:00  family_a_phone_healthcheck.yml alerts
                                                           via WhatsApp, listing any city whose
                                                           raw release is still missing
```

- **`family_a_build_and_notify_from_phone.yml`** — builds and publishes the corrected GTFS, one
  city per matrix leg. Triggered primarily by `repository_dispatch` (fired by the phone right
  after upload — starts within seconds), with the `schedule:` cron kept only as a fallback for
  when the phone's dispatch call itself fails, plus `workflow_dispatch` for manual runs/date/city
  overrides. Same idempotency-guard and actual-recorded-coverage logic as the retired chunk-based
  build below.
- **`family_a_phone_healthcheck.yml`** — a separate scheduled check (21:00 Europe/Warsaw) that
  alerts via WhatsApp if any configured city's raw release from the phone hasn't shown up yet, so
  a silent phone-side failure (recording or upload) is caught from the reliable GitHub Actions
  side.

## History: recording via GitHub Actions (FA-7/FA-8/FA-9, retired 2026-07-14)

Before the phone, recording ran directly on GitHub Actions runners in four overlapping daily
"chunk" windows (`family_a_record_chunk1..4.yml`, built around the morning/afternoon transit
peaks), feeding **`family_a_build_and_notify.yml`** (triggered via `workflow_run` on chunk4's
completion) and its manual companion **`family_a_build_and_notify_on_demand.yml`**. Together
these produced the non-phone-suffixed `lodz-realized-<date>` release.

**This approach is retired, not maintained.** The four chunk workflows have been deleted -
GitHub Actions' `schedule:` trigger proved to have unbounded delay (a 2h31m-late run was
observed and only partially mitigated by widening buffers), and continuous phone recording
sidesteps that whole class of problem instead of working around it further. `family_a_build_
and_notify.yml` and `family_a_build_and_notify_on_demand.yml` are left in the repo as a
reference for the artifact-discovery/coverage/idempotency-guard patterns they established (the
phone-build workflow above reuses the same ideas against a release instead of artifacts), but
**neither has a working data source anymore** - both discover input by `positions-<date>-*`
artifact name prefix, which only the now-deleted chunk workflows ever produced. Running either
by hand today will fail at "no artifacts found" unless something else starts uploading
matching-prefixed artifacts again. Full original design rationale, if ever needed:
`GISBoost/easy-OTP`'s `docs/prd/PR_easy-OTP_v07.md` (FA-7/FA-8/FA-9) and
`docs/prd/PR_easy-OTP_termux-migration.md` (the migration rationale itself).

## Repository configuration

Set under **Settings → Secrets and variables → Actions**, in this repo (not `easy-OTP`'s):

Since TX-8, per-city static GTFS URLs live in `config/cities.json` (this repo, versioned) instead
of Settings variables — see "Multi-city support" above. Each city's `VEHICLE_POSITIONS_URL`
equivalent lives entirely on the phone, in `~/easy-gtfs-rt-termux/cities/<city>.env` (`easy-OTP`'s
`scripts/termux/README.md`), never in this repo's settings.

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

- `family_a_build_and_notify_from_phone.yml` accepts optional `date` and `city` inputs
  (`YYYY-MM-DD`, a `config/cities.json` key) to target a specific day/city instead of building
  every configured city for today - useful when testing, or recovering a day the
  `repository_dispatch`/schedule triggers both missed. See `easy-OTP`'s
  `docs/handoffs/termux-ssh_cheatsheet-for-michal.md` (section 12) for how to manually re-fire
  the `repository_dispatch` event itself from the phone.
- `family_a_phone_healthcheck.yml` also has a bare `workflow_dispatch` for an on-demand check.

A manually-triggered build does **not** get skipped by the idempotency guard unless a Release
for that date/city already exists.

## Known, accepted trade-offs

- **CallMeBot is an unofficial, volunteer-run API** (no SLA/guarantee). If it's down, the
  WhatsApp notification silently fails to send — but the GitHub Release is still published
  either way. The Release is the source of truth; WhatsApp is a convenience notification on
  top of it, not the delivery mechanism itself.
- **DST is not auto-compensated.** `cron:` schedules in this repo are UTC and hand-tuned for
  the currently active offset (see the comment above each `cron:` line for the CET/CEST
  equivalent) — they need a manual one-hour edit after each DST switch (last Sunday of
  March/October). Since the phone-build workflow's primary trigger is now `repository_dispatch`
  (not schedule-based), this only matters for its fallback path and for the healthcheck.
- **The static GTFS is downloaded fresh every build**, never cached long-term — an open data
  feed has been observed (Łódź) to republish with a shifted `trip_id` generation between
  recording sessions, and reusing a stale static feed against newer recordings silently
  produces a suspiciously high `unknown_shape` reject count in `match`'s output instead of an
  error. If that ever shows up, it's this, not a pipeline bug. The exact static GTFS used for a
  given day's `match`/`build` is archived as a third asset on that day's Release
  (`<city>_static_gtfs_<date>.zip`) — precisely so a future comparison between two days'
  realized GTFS isn't invalidated by this same drift.
- **Recorded-coverage times come from each recording directory's own `recording.json` manifest**
  (`started_at`/`stopped_at`, written by `family_a record`), not from a live poll log — so a
  recording session that ran but produced zero/`failed` snapshots throughout would still report
  as "covered" for that time range. This matches what the coverage summary is meant to answer
  ("was a recording job actually running then"), not "how much clean data came back" — the
  latter is what `match`'s own reject-count output is for.

## Where the actual logic lives

This repo only orchestrates. The `family_a` CLI (`record` / `match` / `build`), its algorithm,
and its own documentation live in `GISBoost/easy-OTP`, under
[`tools/family_a_reconstruction/`](https://github.com/GISBoost/easy-OTP/tree/main/tools/family_a_reconstruction)
(see that folder's own `README.md` for the CLI contract this repo's workflows call into,
`docs/prd/PR_easy-OTP_termux-migration.md` for the current phone-based pipeline's design
rationale, and `docs/prd/PR_easy-OTP_v07.md` for the retired GitHub-Actions-recording approach).
