# easy-GTFS-RT

CI/CD-only repository that automates daily recording of GTFS-RT `VehiclePositions` feeds and
the reconstruction of a realized ("what actually happened") GTFS from them, for
[`GISBoost/easy-OTP`](https://github.com/GISBoost/easy-OTP)'s **Family A** tool
(`tools/family_a_reconstruction/`).

This repo contains **no application code** — only GitHub Actions workflows. Every workflow
here does an additional read-only `actions/checkout` of `GISBoost/easy-OTP` and runs the
`family_a` CLI from that checkout. The map-matching/interpolation logic itself is developed
and documented entirely in `easy-OTP`; nothing is duplicated here.

## Understanding the data

**→ [`HOW-IT-WORKS.md`](HOW-IT-WORKS.md) — how a realized GTFS is reconstructed, why the method is
built the way it is, and what those design decisions do to the numbers you download.**
**Po polsku: [`HOW-IT-WORKS.pl.md`](HOW-IT-WORKS.pl.md)** (same content, kept in sync).

Start there if you have a realized GTFS or a diff chart in front of you and want to know how much
to trust it. It walks the causal chain from a GPS ping to a rewritten timetable, gives the
trade-off behind each design decision, explains how to read the published figures (including why a
`0` delay is ambiguous and why "% of rows changed" is not a quality score), and lists the per-city
defects currently known to affect the output. This README covers the *orchestration*; that
document covers the *method and its consequences*.

## Browsing the data

[`GISBoost/gtfs-dashboard`](https://github.com/GISBoost/gtfs-dashboard) is a new, separate
static site for easily browsing the Releases this repo produces
(**[gisboost.github.io/gtfs-dashboard](https://gisboost.github.io/gtfs-dashboard/)**) — a
city → month → day drill-down instead of scrolling the raw Releases list, which is no longer
practical once you're past a handful of cities recording daily. It only reads this repo's public
Releases API and `config/cities.json`, on its own daily schedule (04:00 UTC — late enough to have
caught Boston, which publishes at 02:21–02:26 UTC); nothing here needs to change to support it.

Between 2026-07-20 and 2026-07-29 this repo also pushed a `repository_dispatch` to that one after
each publish, so the dashboard updated within seconds instead of waiting for its cron. That step
is gone: releases appear at most once per city per day, which made near-instant refresh not worth
maintaining a cross-repo PAT for.

## Why this is a separate repo from `easy-OTP`

- `easy-OTP`'s GitHub Releases are the plugin's own version history (`easy_otp-0.3.5.zip`
  etc.). Dumping a GitHub Release into that list every single day would bury real plugin
  releases under daily data snapshots.
- This repo is **public**, which gets GitHub Actions unlimited minutes and artifact storage
  outside the default 500MB/repo cap that applies to private repos — meaningful for a job that
  records for up to 16 hours/day and produces a new GTFS build every evening.
- Anything secret this pipeline ever needs is scoped to this repo only and is never needed by, or
  exposed to, `easy-OTP`. (As of 2026-07-29 it needs none — see "Repository configuration".)

## Multi-city support (TX-8)

The phone-build workflow (`family_a_build_and_notify_from_phone.yml`) supports
recording/building more than one city from the same phone in parallel. **`config/cities.json`**
(versioned in this repo) is the single source of
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
                                                    publishes "<city>-realized-<date>-phone"
```

The workflow filenames still say `_notify_`, which is now historical — they no longer notify
anything (see "Failure notification" below). Renaming them would break every existing bookmark,
`workflow_dispatch` shortcut and cross-document reference for no functional gain, so they keep
their names.

- **`family_a_build_and_notify_from_phone.yml`** — builds and publishes the corrected GTFS, one
  city per matrix leg. Triggered by `repository_dispatch` (fired by the phone right after upload
  — starts within seconds), plus `workflow_dispatch` for manual runs/date/city overrides (no
  `schedule:` fallback — see its header comment for why that was dropped). Same idempotency-guard
  and actual-recorded-coverage logic as the retired chunk-based build below.

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

**No repository secrets are required.** As of 2026-07-29 this repo's workflows use nothing beyond
the automatic `GITHUB_TOKEN`.

Since TX-8, per-city static GTFS URLs live in `config/cities.json` (this repo, versioned) instead
of Settings variables — see "Multi-city support" above. Each city's `VEHICLE_POSITIONS_URL`
equivalent lives entirely on the phone, in `~/easy-gtfs-rt-termux/cities/<city>.env` (`easy-OTP`'s
`scripts/termux/README.md`), never in this repo's settings.

Three secrets used to live here and are now **unused and safe to delete** under
**Settings → Secrets and variables → Actions** — nothing reads them any more:

| Removed secret | What it was for |
|---|---|
| `CALLMEBOT_PHONE` | WhatsApp notification target (removed with the notify steps) |
| `CALLMEBOT_APIKEY` | CallMeBot API key for the same |
| `GTFS_DASHBOARD_DISPATCH_TOKEN` | Cross-repo PAT to `repository_dispatch` the dashboard refresh |

The phone still authenticates to *this* repo with its own token to upload raw recordings and fire
its `repository_dispatch`; that lives on the phone, not in these settings, and is unaffected.

### Failure notification

There is none of our own, deliberately. GitHub already emails the repository owner when a
scheduled or dispatched workflow run fails, which is the same signal the WhatsApp "build FAILED"
message carried — so that message was pure duplication, and it depended on a free third-party API
with a request quota that eventually ran out. Watch the Actions tab, or GitHub's own notification
settings, instead.

## Manual testing

- `family_a_build_and_notify_from_phone.yml` accepts optional `date` and `city` inputs
  (`YYYY-MM-DD`, a `config/cities.json` key) to target a specific day/city instead of building
  every configured city for today - useful when testing, or recovering a day the
  `repository_dispatch` trigger missed. See `easy-OTP`'s
  `docs/handoffs/termux-ssh_cheatsheet-for-michal.md` (section 12) for how to manually re-fire
  the `repository_dispatch` event itself from the phone.

A manually-triggered build does **not** get skipped by the idempotency guard unless a Release
for that date/city already exists.

## Known, accepted trade-offs

- **Success is not announced anywhere.** Until 2026-07-29 a WhatsApp message went out per
  published release, via CallMeBot — an unofficial, volunteer-run API with a request quota, which
  ran out. It was removed rather than replaced: the Release itself was always the source of
  truth and the message only ever restated it, while failures are already covered by GitHub's own
  emails. The cost is that noticing a *silently missing* day is now entirely manual (see the last
  bullet below).
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
- **No automated detection of a fully dead phone.** `family_a_phone_healthcheck.yml` used to poll
  for this (deleted 2026-07-17 — its per-city local-time gate had a false-positive/timing problem
  that made it unreliable in practice: it could fire "raw recording missing" for a city shortly
  after that same city's build had already published successfully). Until a replacement exists,
  the only signal that a day silently produced nothing is the *absence* of that city's usual
  Release — nothing alerts on that absence itself. Note this got weaker on 2026-07-29: a missing
  WhatsApp message used to be a passive cue that arrived by itself, whereas a missing Release has
  to be looked for. GitHub's failure emails do not cover this case, because a phone that never
  uploads never triggers a run that can fail.

## Data and attribution

**The licence on this repository covers this repository's own contents — not the data in its
Releases.** That distinction matters here more than it does in most repos, because every Release
contains material this project does not own:

- **The archived static GTFS** (`<city>_static_gtfs_<date>.zip`) is a *verbatim copy* of the
  transit agency's own feed, redistributed unchanged. Its terms are entirely the agency's.
- **The realized GTFS** (`<city>-realized-<date>-*`) is a *derivative* of that feed — it reuses the
  agency's `stop_id`s, `route_id`s, `trip_id`s, shapes and stop coordinates, and rewrites only the
  times. Most open-data licences carry attribution, and some carry share-alike, obligations
  through into derivatives like this one.
- **The recorded positions** (`positions-raw-<city>-<date>`) come from each agency's public
  GTFS-RT `VehiclePositions` endpoint.

The authoritative list of sources is [`config/cities.json`](config/cities.json) — each entry's
`static_gtfs_url` points at the operator or open-data portal the feed came from. **Check the terms
at that source before redistributing or publishing anything derived from a Release.** They are not
uniform: the 13 cities currently recorded span nine countries and a correspondingly wide range of
open-data regimes, from explicit Creative Commons grants to bespoke portal terms of use. This
project cannot and does not relicense any of it.

If you use this data in published work, attribute both the originating agency (per its own terms)
and, if the reconstruction itself is relevant to your result, this project — the delay figures are
*inferred*, not measured, and [`HOW-IT-WORKS.md`](HOW-IT-WORKS.md) explains what that costs.

## Licensing

| What | Licence |
|---|---|
| Code in this repo (workflows, `config/cities.json`, `README.md`) | MIT — [`LICENSE`](LICENSE) |
| `HOW-IT-WORKS.md` / `HOW-IT-WORKS.pl.md` | CC BY 4.0 — [`LICENSE-docs`](LICENSE-docs) |
| Data in the GitHub Releases | Not ours to license — see "Data and attribution" above |

The workflows here are MIT precisely so they are easy to copy: pointing this pipeline at a new
city is a `config/cities.json` entry and a phone-side config, and nothing in this repo should
stand in the way of someone doing that for their own city.

`easy-OTP`, which holds the actual reconstruction logic these workflows invoke, is
**GPL-3.0-or-later** (a QGIS plugin repository requirement). There is no conflict: these workflows
`checkout` and run its CLI as a separate process, which is use, not derivation — but note that a
copy of the *`family_a` code itself* stays GPL wherever it goes.

## Where the actual logic lives

This repo only orchestrates. The `family_a` CLI (`record` / `match` / `build`), its algorithm,
and its own documentation live in `GISBoost/easy-OTP`, under
[`tools/family_a_reconstruction/`](https://github.com/GISBoost/easy-OTP/tree/main/tools/family_a_reconstruction)
(see that folder's own `README.md` for the CLI contract this repo's workflows call into,
`docs/prd/PR_easy-OTP_termux-migration.md` for the current phone-based pipeline's design
rationale, and `docs/prd/PR_easy-OTP_v07.md` for the retired GitHub-Actions-recording approach).
