# Strokes Gained — On-Course Tracker

A single-file web app for tracking **strokes gained / lost** on every shot, GPS-assisted, at your own course — comparable against a **PGA Tour** or **scratch** baseline, with a **game profile** that reads out a handicap for each part of your game.

This README is the project handoff: what it is, how it works, where the data comes from (and where it's approximate), and what to build next.

---

## Status

Working web app, one file (`index.html`), no backend. Runs in any modern mobile browser. Persists data locally on the device. Ready to host on HTTPS and use on-course.

---

## How it works

### Strokes gained engine
For any ball position — defined by **distance to the hole** and **lie** (tee, fairway, rough, sand, recovery, green) — there is an *expected number of strokes to hole out*. Strokes gained on a single shot is:

```
SG = expected(start) − expected(end) − 1 − penalties
```

The last shot of a hole holes out, so its `end` expected value is 0. Summed over a round and split into four categories, this shows not just your score but *where* you gain or lose strokes.

- `expected(benchmark, lie, dist)` — linear interpolation over the baseline tables. Distances are **yards** for tee/fairway/rough/sand/recovery and **feet** for `green` (putting).
- `holeStrokesGained(hole, benchmark)` — per-shot SG + category for one hole.
- `roundTotals(round, benchmark)` — totals and per-category sums for a round.
- `classify(shot, par)` — category rules: `green` → putting; within 30 yds of the hole (off the green) → around-the-green; tee shot on a par 4/5 → off-the-tee; everything else (incl. par-3 tee shots) → approach.

### Benchmarks (the `BENCHMARKS` block at the top of the script)
Two baselines, each a set of `[distance, expectedStrokes]` anchors per lie:

- **`tour`** — approximation of Mark Broadie's PGA Tour benchmark (≈ *Every Shot Counts*, Table 5.2). The tee column was corrected so a 400-yard tee shot reads ~3.99.
- **`scratch`** — derived from the tour table plus research-implied position gaps (small near the green, larger on long shots; the rough-vs-fairway penalty is slightly smaller than tour's). Sanity-checked: ~3.11 from 150 fairway, ~4.11 off a 400-yard tee, ~1.77 from 14 feet.

**These tables are approximate and intentionally swappable.** To use exact published values, replace the arrays in `BENCHMARKS` — the cleanest source is Broadie's published baseline (his book, or the Broadie-derived datasets that circulate as CSVs). Because SG is a *difference* of expected values, small absolute errors in a baseline's level largely cancel; what matters is the shape (how expected strokes change with distance and lie).

### Game profile (category handicaps)
Aggregates every logged round, computes average SG/round **vs scratch** per category, and maps each to the handicap that typically performs at that level.

- `HCP_COEF = { ott: 0.22, app: 0.30, arg: 0.15, putt: 0.13 }` — strokes/round lost per handicap point vs scratch, per category. Calibrated to published per-category benchmarks (approach is the widest gap, putting the narrowest).
- `HCP_TOTAL_COEF = 0.80` — for the overall figure.
- `handicap = clamp(round(−sgPerRound / coef), −8, 36)` — negatives render as plus-handicaps (e.g. `+2`).

**This is the approximate part of the app.** The *spread* between your categories (e.g. "driving much better than putting") is the trustworthy signal; treat the exact handicap digit as ±a few. It measures shot quality, not course-adjusted handicap, so it will not match your official number exactly. Note a real, counterintuitive effect: two-putting from range is slightly *below* scratch (scratch makes more of those), so putting often reads as a higher handicap than people expect — that's correct, not a bug.

### GPS & course profiles
- A **course profile** stores each hole's **green-centre** coordinate and par, set once and reused for every round. Green centres are stable (pins move daily); the centre is captured by tapping a satellite map (Esri World Imagery via Leaflet, no API key) or by pasting coordinates from Google Maps.
- During a round, **Measure** reads the phone's GPS and computes the haversine distance from your ball to the saved green centre. GPS requires an HTTPS page + location permission.
- **Putting is always entered manually in feet.** GPS (±3–5 m) is far too coarse for the steep putting curve, and the on-green guard blocks Measure on the green so you can't accidentally log a distance-to-centre as a putt length.
- Auto-detecting greens from imagery was considered and rejected: unreliable, and no more accurate than a tap once GPS error is accounted for. The realistic future automation is pulling green polygons from OpenStreetMap and confirming them — not image recognition.

---

## Data model

Stored via a small shim that prefers `window.storage` (when embedded) and falls back to `localStorage` (when self-hosted). Keys:

- `sg_rounds_v1` — `[ Round ]`
- `sg_courses_v1` — `[ Course ]`
- `sg_settings_v1` — `{ benchmark: "tour" | "scratch" }`

```
Round   = { id, course, date, holes: [ Hole ] }
Hole    = { par, shots: [ Shot ], pin: {lat,lng} | null }
Shot    = { lie, dist, pen }            // dist in yards, or feet when lie === "green"
Course  = { id, name, holes: [ { par, green: {lat,lng} | null } ] }
```

---

## Known limitations

- **Data is local to one browser/device.** No backup or sync yet — clearing browser data wipes your rounds. (See roadmap: export/import is the first fix.)
- **GPS and the satellite map need HTTPS.** They will not work over `file://` or in a non-secure preview.
- Baseline tables and the handicap mapping are approximations (see above).
- Course rating/difficulty is ignored, so SG-vs-scratch won't line up exactly with your handicap.

---

## Deployment roadmap

1. **Host on HTTPS and use it (no code).** Drop `index.html` on Netlify Drop, GitHub Pages, or Cloudflare Pages; open on your phone; allow location; Add to Home Screen.
2. **Make it an installable PWA.** Add a web-app manifest, an icon set, and a service worker. Gains: installable + offline. (Shot tracking works offline since GPS needs no data; only course setup needs signal.)
3. **Stop losing data.** Add export/import (rounds → JSON file → re-import). Optional cloud sync via Supabase/Firebase if multi-device matters.
4. **Native / app store (optional).** Wrap the existing web code with Capacitor for real iOS/Android binaries — no rewrite. Publishing needs Xcode + Android Studio and the Apple ($99/yr) and Google Play ($25 one-off) accounts.

---

## Backlog / next features

- Export / import rounds (data safety) — do this first.
- PWA manifest + service worker + icons.
- Edit / delete saved rounds and shots.
- Additional baselines (5- and 10-handicap) as extra toggle options, or as a category-offset readout.
- OpenStreetMap green-centre auto-suggest (Overpass), user-confirmed.
- Charts: category handicaps and SG trends over time.
- Replace approximate baselines with exact published Broadie values.

---

## File map

```
index.html    The entire app (HTML + CSS + JS, no build step)
README.md     This file
.gitignore
```
