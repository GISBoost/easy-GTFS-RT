# How it works — the method behind these releases, and what it does to the data

*[Wersja polska](HOW-IT-WORKS.pl.md)*

Every day this pipeline publishes, per city, a **"realized" GTFS**: the city's official timetable
with the arrival and departure times rewritten to reflect what vehicles actually did. This
document explains how that is produced, why it is built the way it is, and — most importantly —
**what those design decisions do to the numbers you download**.

It is written for someone who has a realized GTFS (or a diff chart) in front of them and wants to
know how much to trust it and how to read it. It is not a CLI reference; for the flags and their
defaults see
[`tools/family_a_reconstruction/README.md`](https://github.com/GISBoost/easy-OTP/tree/main/tools/family_a_reconstruction)
in `easy-OTP`. For how the automation runs (schedules, workflows, notifications) see this repo's
[`README.md`](README.md).

**Contents**

1. [The single constraint everything follows from](#1-the-single-constraint-everything-follows-from)
2. [The chain, step by step](#2-the-chain-step-by-step)
3. [Design decisions and what each one costs](#3-design-decisions-and-what-each-one-costs)
4. [How to read the published numbers](#4-how-to-read-the-published-numbers)
5. [What is known to be wrong right now](#5-what-is-known-to-be-wrong-right-now)
6. [Checking any of this yourself](#6-checking-any-of-this-yourself)

---

## 1. The single constraint everything follows from

GTFS-Realtime has two relevant feed types:

- **`TripUpdates`** — the agency tells you directly: *trip X is 4 minutes late at stop Y*.
- **`VehiclePositions`** — the agency tells you only: *vehicle V is at this lat/lon right now,
  running trip X*.

Many cities publish only the second one. **Delay is therefore never measured here. It is
reconstructed by inference** — from where vehicles were, when, and where the schedule says they
should have been.

That one fact is the root of everything below. Because the delay is inferred rather than reported:

- every number depends on a chain of intermediate decisions (which trip? where on the route? when
  did it cross that stop?), and **each link can fail quietly**, producing a plausible number
  rather than an error;
- the output can only ever describe **segments that were actually observed**; everywhere else it
  falls back to the published schedule;
- "no correction" and "ran exactly on time" end up looking identical in the output file. This
  ambiguity is unavoidable given the input, and §4 explains how to work around it.

The method is the one described in Wessel, Allen & Farber (2017), *"Constructing a Routable
Retrospective Transit Timetable from a Real-time Vehicle Location Feed and GTFS"* — position-based
map-matching plus interpolation — adapted to run without an external routing engine.

---

## 2. The chain, step by step

Five steps turn GPS pings into a rewritten timetable. Each one has to *decide* something the raw
data does not state outright, and each decision has a price.

```
  VehiclePositions          static GTFS
   (every 60 s)             (that day's)
        |                        |
        v                        v
  1. record  ->  2. match  ->  3. anchor stops  ->  4. interpolate  ->  5. aggregate
                                                                              |
                                                                              v
                                                                        6. rebuild
                                                                     (realized GTFS)
```

### 2.1 Record — one poll per minute

A phone runs continuously from ~06:00 to ~22:00 and saves the raw feed once every 60 seconds.

**What that costs:** 60 seconds is the finest time resolution that can ever exist in the output. A
vehicle passing a stop between two polls is never *seen* doing it — its crossing time is
interpolated (step 4). Denser polling would not create more observations; it would only make each
one more precise (see §2.5 for why).

### 2.2 Match — put each position somewhere on a route

A lat/lon on its own is useless. To be meaningful it must become *"this far along route R"*. That
needs two things the position does not carry:

- **Which trip is this?** — taken from the RT feed's own `trip_id`. If a feed does not populate
  it, the observation cannot be used at all. This is not hypothetical: on 2026-07-20 **335,257 of
  Turin's 337,023 observations carried no `trip_id`**, and on 2026-07-22, *all* 344,449 did.
- **What does that trip's route look like?** — taken from the static GTFS (`shapes.txt`). This is
  why the static feed used must be **the same publication the RT feed is currently serving**. Some
  agencies renumber their entire `trip_id` namespace on every republication (Łódź does, every 1–3
  days), so a static feed that is merely *recent* is not good enough — it must be *the matching
  one*. Each day's release therefore archives the exact static GTFS used, as a third asset.

The position is then projected onto the route's polyline to get a distance-along-route.

**What that costs:** projection alone is ambiguous wherever a route passes close to itself — a
loop, a turnback, two parallel carriageways. A few metres of GPS noise can flip a match between
two points that are metres apart in space but kilometres apart *along the route*. Steps 2.3 and
the windowing described in §3 exist to contain this.

### 2.3 Anchor the stops onto the same route line

Before you can say *when* a vehicle passed a stop, you must know *where along the route that stop
sits*. This sounds trivial and is the single largest source of error in the whole method.

Projecting each stop independently onto the polyline breaks on any route that revisits its own
path: a stop near the end of a trip can anchor to the *early* pass through the same place. The
result is arithmetic performed correctly on real GPS data, anchored to the wrong point in space —
producing, in one reproduced case, an "observed" 35-minute travel time for a scheduled 60-second
hop.

Two strategies are used, in order of preference:

1. **Trust the feed's own measurements.** Some feeds publish `shape_dist_traveled` — the agency's
   own distance-along-route for every stop and shape point. Where present, consistently filled,
   and unit-consistent, this removes the ambiguity entirely. Of the monitored cities only Prague
   actually provides it, and in **kilometres** rather than metres — the spec does not fix the
   unit, so it has to be detected per feed rather than assumed. Łódź and Vilnius have the column
   in their files but leave **every value blank**, which is why the check is on the data and not
   on the column's existence: *present* and *usable* are not the same thing.
2. **Resolve the whole trip's stop pattern at once.** Where the feed does not provide it, stops
   are resolved *in sequence order*, each one searching only forward of the previous one (with a
   small backward tolerance for depots and interchanges where real feeds are legitimately
   slightly out of order).

**What that costs:** strategy 2 is a large risk reduction, not a proof. Very densely overlapping
loops can still resolve wrongly, and the strategy chosen depends on what each feed happens to
publish — so two cities' outputs are not produced by identical code paths.

### 2.4 Interpolate — when did it cross that stop?

With the stop's position on the route known, the vehicle's own series of (time, distance) pings is
scanned for the consecutive pair that brackets it, and the crossing time is interpolated linearly
between them.

**What that costs, and this is important:**

- If the two bracketing pings are far apart in time, the interpolated crossing describes **how
  sparsely we were recording**, not how fast the vehicle moved. Pairs more than 300 seconds apart
  are therefore rejected outright.
- If a stop was crossed while the vehicle was not being tracked at all, there is no bracketing
  pair and **no observation is produced**. That segment becomes a *gap* and keeps its scheduled
  time.
- A single bad GPS reading contaminates more than one segment. Because every scheduled stop whose
  distance falls inside the same bad pair gets the same bogus bracket, one wrong ping "fans out".
  Measured in Bucharest: **1,414 anomalous raw pairs explained 3,613 rejected segment
  observations — a 2.56x amplification**, because its stops sit close together.

### 2.5 Aggregate — pool observations into segment statistics

Observations are not applied trip-by-trip. They are pooled into a **segment key**:

```
(route_id, direction_id, from_stop, to_stop, day_type, time_bucket)
```

Each dimension is there for a reason, learned the hard way:

- **route + direction + stop pair** — the natural unit of "how long does this hop take".
- **day_type** (weekday / Saturday / Sunday) and **time_bucket** (2-hour blocks) — added after a
  single afternoon recording was found to have corrected **74% of a six-month feed**, including
  trips departing at 3:48 AM that the recording could not possibly have observed. Without these
  two dimensions, corrections leak across times and days they have no evidence for.

**What that costs — the most under-appreciated property of this data:** one trip run contributes
**at most one observation** to a given key, because a trip visits any consecutive stop pair once.
Minute-level polling densifies the pings *within* a trip; it does not create more samples of it.
So a key's sample size is governed by **how often that route actually runs inside that 2-hour
window** — an hourly bus can contribute at most ~2 observations no matter how good the recording
is. Measured across cities, the mean is **1.8–6.2 observations per key**, and in Poznań and
Prague **28–47% of keys rest on a single observation**. For those, the reported median *is* that
one reading.

Each key is then reduced to a **P50** (median — the typical case) and a **P85** (85th percentile —
the pessimistic case). Two files are published per day for exactly this reason: P50 answers "how
long does this usually take", P85 answers "how long should I allow".

### 2.6 Rebuild — write the corrected timetable

Finally the schedule is rewritten. For every trip: start from its **scheduled first departure**,
then walk the stops in order, using the observed segment time where one exists and the scheduled
one where it does not, accumulating as you go.

**What that costs — read this before interpreting any delay number.** The reported delay is a
**running total**, not an independent per-stop measurement:

- it is anchored to the *schedule* at the first stop, so a vehicle that departed late starts at
  zero delay by construction;
- error accumulates along the trip, and a segment with **no** observation carries whatever
  accumulated so far forward unchanged rather than letting it decay.

This is directly visible in the published data. Median delay by position within a trip, 2026-07-24:

| City | early in the trip | late in the trip |
|---|---|---|
| Łódź | 8 s | 48 s |
| Vilnius | 17 s | 67 s |
| Gdańsk | 98 s | 233 s |
| Poznań | 86 s | 147 s |
| Prague | 158 s | 244 s |

The rise is monotone in **every** city measured. Some of that is real — buses genuinely fall
further behind — but the structure of the metric guarantees a rise even where reality would not.
Prague's case is instructive: its delay is roughly mode-independent, and **even its metro shows
+158 s**, which is not credible for a closed right-of-way system. Treat "delay grows along the
trip" as partly an artifact of the method, not purely a property of the city.

---

## 3. Design decisions and what each one costs

Every one of these is a deliberate trade-off. The right-hand column is what it means for you.

| Decision | Why it is this way | What it costs in the data |
|---|---|---|
| Poll every 60 s | Balances feed politeness against precision; agencies rate-limit | Crossing times are interpolated, never observed directly |
| One recording day per build | The phone records one continuous session per city per day | Thin samples per segment key (§2.5). Pooling several days is possible but not currently the default |
| Static GTFS downloaded fresh each build, and archived with the release | Feeds republish with renumbered `trip_id`s; a stale static silently matches nothing | If the agency publishes the *next* period's feed early, that day's build degrades — see §5 |
| Segment keyed by day_type + 2-hour bucket | Stops a single afternoon from "correcting" a whole feed | 12 buckets/day × 3 day types fragments the sample; more resolution means fewer observations per key |
| Minimum 2 observations per segment | A single reading is not evidence | Segments seen once are dropped entirely and keep scheduled times — they become indistinguishable from unobserved ones |
| Gap = keep the scheduled time | The only honest fallback; inventing a number would be worse | **A delay of exactly 0 is ambiguous**: either genuinely on time, or never observed |
| Reject implied speeds over 100 km/h | Catches interpolation artifacts; 100 km/h is generous for urban operating speed | Legitimately fast services are rejected too. In Prague this removes **2,187 of 123,833 keys**, almost all regional rail |
| Reject bracketing pairs more than 300 s apart | Such a pair measures recording sparsity, not speed | Genuinely slow, sparsely-tracked segments lose data along with the bad ones |
| Correction applies to every trip sharing a segment key | One observation per key is often all there is; without sharing, almost nothing would be corrected | One wrong observation propagates to every trip on that route/direction/stop-pair/bucket |
| P50 and P85 published separately | Median hides tail risk; the 85th percentile is the planning number | Neither is "the" answer; pick per use case |
| No external routing engine | Keeps the tool dependency-free and portable to a phone | Map-matching is geometric only, with no notion of trajectory continuity — hence the loop ambiguity in §2.3 |

---

## 4. How to read the published numbers

### What "delay" means here

`delay = realized_departure_time - scheduled_departure_time`, per `stop_times.txt` row. The
realized feed is byte-identical to the static one except for corrected times, so every row has a
counterpart and the join is exact.

### A zero is ambiguous — and common

A `0` means *either* "ran exactly on schedule" *or* "never observed, so the scheduled time was
kept". Nothing in the GTFS files distinguishes them. This is why the published charts **exclude
zero-delay rows**: including them would drag every average toward zero without meaning anything.

### "% of rows changed" is not a quality score

This one trips up almost everyone, so it is worth being precise.

A recording covers **one day**. The static feed is typically valid for **weeks**. Corrections are
only accepted for trips whose service actually runs on the recorded day type — so a large share of
rows was never eligible for correction in the first place. The honest denominator is *what could
have been corrected*, not *all rows*.

Measured for 2026-07-24 (a Friday):

| City | rows eligible that day | rows actually changed | share of what was achievable |
|---|---|---|---|
| Lisbon | 61.7% | 50.3% | **82%** |
| Vilnius | 83.5% | 65.7% | **79%** |
| Gdańsk | 79.7% | 62.4% | **78%** |
| Prague | 68.3% | 48.0% | **70%** |
| Łódź | 35.4% | 23.8% | **67%** |

Łódź is the clearest illustration: only **35.4%** of its rows were correctable at all, so "23.8%
changed" is two thirds of the maximum — not a two-thirds failure. **A healthy day lands at roughly
66–82% of what was achievable.** Anything far below that is worth investigating; a raw percentage
on its own tells you almost nothing.

### Delay grows along a trip

See §2.6. When comparing cities or days, compare like-for-like positions within trips, or accept
that a route with long trips will report larger delays than one with short trips regardless of
actual punctuality.

### P50 vs P85

P50 is the median observed segment time; P85 is the 85th percentile, clamped so it is never below
P50. For a key with a single observation both equal that observation — the P85 file does *not*
represent extra evidence in that case.

---

## 5. What is known to be wrong right now

This is a living list; it is not exhaustive. Issues are tracked in
[`GISBoost/easy-OTP`'s issues](https://github.com/GISBoost/easy-OTP/issues) and
[`KNOWN_ISSUES.md`](https://github.com/GISBoost/easy-OTP/blob/main/KNOWN_ISSUES.md).

### Affecting specific cities

| City | What is wrong | Effect on the data |
|---|---|---|
| **Boston** | Builds before 2026-07-28 dropped most trips (a type-inference defect on purely numeric `trip_id`s) | Pre-fix releases cover only **25 of 126 observed routes** — an arbitrary, clustered slice. **Not comparable with later builds and not rescalable.** Fixed going forward; historical releases are deliberately not recomputed |
| **Turin** | Its `VehiclePositions` feed frequently omits `trip_id` entirely | Days where this happens produce almost nothing. 2026-07-20 published a build with **217 corrected rows out of 1,416,230**; 2026-07-22 produced none at all |
| **Poznań** | The agency publishes the next period's static feed several days early, so a build can use a feed not yet valid for the recorded day | Roughly **1 day in 3** is badly degraded. Two of six sampled days had a static feed starting *after* the recording date |
| **Łódź** | **97 `shape_id`s referenced by `trips.txt` have no geometry at all** in `shapes.txt` (18 of them on route `603` alone) — a defect in the agency's own export | Affected routes are invisible: route `603` produced **7,478 observations, all unusable**, and route `R9` is affected on **every archived day**. Their times are pure schedule |
| **Prague** | The flat 100 km/h plausibility filter rejects legitimate regional rail | **2,187 of 123,833 segment keys** dropped, almost all `route_type=2`. Prague's rail is under-corrected relative to its trams and buses |
| **Bucharest** | Its raw feed has a ~6–8x higher rate of isolated bad GPS readings than Poznań or Łódź (0.267% vs 0.035–0.041% of consecutive pairs) | More segments rejected as implausible. The filter is working correctly; the input is noisier |
| **Vilnius** | It is the only city with no `current_stop_sequence`, so live-position matching relies on `stop_id` alone | A known regression concentrated on route `A62`, whose geometry interacts badly with the matching window. Reproducible across every day tested |

### Affecting every city

- **Delay is a running total, not a per-stop measurement** (§2.6). Partly structural, not purely
  real.
- **Thin samples.** Up to ~47% of segment keys rest on a single observation (§2.5).
- **A zero delay is ambiguous** (§4).
- **Day type is the local calendar day, not GTFS's service day.** An overnight trip observed just
  after midnight is attributed to the next day's type.
- **Public holidays are not modelled.** A holiday running Sunday-style service is still treated as
  whatever weekday it falls on.
- **Two unexplained cases remain open**, both investigated and both still without a confirmed
  cause: a Poznań stop pair at Rondo Rataje that stays inflated after four separate fixes, and the
  precise origin of Prague's constant baseline offset (the accumulator in §2.6 explains the *rise*
  along a trip, not the offset already present at the second stop).

---

## 6. Checking any of this yourself

Everything here is reproducible from public artifacts. Each daily release contains:

| Asset | What it is |
|---|---|
| `<city>_static_gtfs_<date>.zip` | The exact static feed that build used |
| `<city>_realized_<date>_p50.zip` | Median-corrected timetable |
| `<city>_realized_<date>_p85.zip` | 85th-percentile-corrected timetable |
| `<city>_diff_<date>_p50_summary.csv` | Per-route delay statistics for that day |
| `<city>_diff_<date>_p50_chart.png` | Mean delay by time of day |

The summary CSV gives, per `route_id`: row count, how many changed, `pct_changed`, and mean /
mean-absolute / stdev / min / max delay in seconds, plus an `ALL` row. Read `pct_changed` against
§4's ceiling caveat.

To go deeper, the reconstruction tool itself is open and runnable:
[`tools/family_a_reconstruction/`](https://github.com/GISBoost/easy-OTP/tree/main/tools/family_a_reconstruction)
in `easy-OTP` — `record`, `match`, and `build` are separate commands, and both `match` and `build`
print per-route diagnostics and warn when a run looks unhealthy (a route with observations but no
usable ones, an implausible rejection rate, or a matched table paired with the wrong static feed).

Browse the releases at **[gisboost.github.io/gtfs-dashboard](https://gisboost.github.io/gtfs-dashboard/)**.
