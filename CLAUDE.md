# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A no-build, offline-first field data collection tool for the TCZ program's **multi-species track
surveys**. A crew member starts a walk, which starts a timer and a GPS track. They walk a
haphazard path, and the first time they see tracks or other sign of each species they tap it. The
app records the seconds since the walk started (time to detection, TTD). At the end of the walk
they add survey metadata, and the record syncs to an ODK Central form over OpenRosa. Species not
seen on a walk are censored at `duration_seconds`.

It is the third app in the family, after `../goanna-hunting-app` and `../toad-monitoring-app`.
**Read `../goanna-hunting-app/CLAUDE.md` first.** The settings/sync engine, app-user QR decoding,
GPS sampler, track logging (`prepareTrack()` and the geotrace serialisation), wake lock, PWA
shell and jsQR vendoring were ported from it largely unchanged. This file documents what's
different. There are no Entities (unlike the toad app), because walks aren't tied to persistent
sites.

- `track-monitoring-app.html`: the whole application (HTML, CSS and vanilla JS in one file, no
  dependencies, no bundler).
- `track_ttd_survey.xlsx`: the XLSForm (`survey`, `choices`, `settings`). `form_id` is
  `track_ttd_survey`, with no `version` column, following the goanna form's precedent.
- `track_ttd_survey.xml`: the compiled XForm, which the user uploads to Central. Regenerate it
  whenever the `.xlsx` changes (see below) and commit both.
- `manifest.json` / `sw.js` / `index.html`: the PWA shell, the same mechanism as goanna.
  `CACHE_NAME` is `tcz-track-shell-v1`; bump it when the cached files change. The stale-while-revalidate
  service worker means the *first* reload after an edit still serves the old HTML, which is easy
  to mistake for a failed edit while testing.

There's no build step or test suite. Serve the directory (`python3 -m http.server`) and open
`track-monitoring-app.html`. To re-validate the form (the only automated check):
```
pip install pyxform   # throwaway venv, not the system Python
xls2xform --odk_validate --pretty_print track_ttd_survey.xlsx track_ttd_survey.xml
```

## What differs from the goanna app

1. **Walk state machine.** `walkState` moves `idle → running → stopped → idle`.
   - `running` mirrors `walkState === 'running'` for the ported helpers.
   - The timer card has two views: `#idleView` (ring + Start) and `#walkView` (compact clock +
     End walk). The species picker (`#pickerCard`) is visible only while walking.
   - **End walk asks first** in a small sheet (`#endConfirmOverlay`: "Yes, end walk" / "No,
     keep walking"). It isn't a blocking `confirm()`, so the clock and GPS track keep running
     while it's open. It replaced a tap-twice button, because field crews didn't realise it
     needed the second tap (issue #2).
   - The end-of-walk sheet (`#overlay`) has **no backdrop-tap dismiss**, because it holds the
     whole walk. The ways out are Save, or 🗑 → Resume walk (continues the same clock) /
     Discard walk.
   - There's no per-minute beep: walks are meant to run long.
2. **In-progress walk persistence** (`tcz_track_active_walk_v1`). `saveActiveWalk()` writes
   `{ state, startTs, lastElapsedMs, trackPoints, detections, startFix }` on every change, and
   `restoreActiveWalk()` resumes on load. The goanna app has no equivalent; a killed tab there
   loses one short search, but here it would lose a whole walk.
   - The clock is wall-clock based, so time spent killed still counts. That's correct, because
     the walk carried on.
   - Answers half-entered in the end-of-walk sheet are not persisted.
3. **Species picker, two levels.**
   - `SPECIES_TREE` is the only definition in the app. It has 9 categories with ≤12 leaves each,
     so every level fits one phone screen as a 3-column grid. **Array order is display order**,
     taken from how often each species turned up in 20 pilot surveys.
   - **TTD is captured at the category (level-1) tap** (`pendingTtdMs`, snapped to whole
     seconds), not the leaf tap, so the drill-down adds no lag. It's committed by the leaf tap
     and discarded by ‹ Back.
   - A recorded leaf is disabled and shows its time. The exception is leaves with `other: true`
     (`other_lizard`, `other_mammal`, `other`): these open `#otherOverlay` for a description,
     stay enabled, and warn once on a case-insensitive duplicate description.
   - **Changing a detection** (issue #2): tapping a "Seen this walk" chip, or a name on the
     end-of-walk sheet, opens `#editOverlay`. This is a second copy of the two-level picker,
     driven by `editingId`/`editGroup`. Changing a detection relabels it, but `ttd_seconds` and
     the detection location stay, since they belong to the moment the sign was first seen.
     `other: true` leaves go through `#otherOverlay`, which has a higher z-index so it can sit
     on top of either sheet. If the chosen species is already recorded, `applyEdit()` merges
     the two, keeping the earlier TTD. Undo is offered only mid-walk; on the end sheet the
     toast would cover Save. Saved records are final.
   - Each tap shows a 5 s toast with **Undo**. `showToast()` is the toad app's toast plus an
     optional undo callback, and it's dismissed when the walk ends so it doesn't cover the
     sheet's Save row.
4. **GPS.**
   - The start fix comes from `sampleLocation()`, but with `GPS_SAMPLE_MAX_MS` cut to **20 s**,
     because the crew is already walking away while it samples.
   - GPS problems are shown live on the walk card (`#gpsLine`), not as save-time `confirm()`s
     like goanna's `geoWarned`/`gpsGateWarned`. By the time the walk ends, a retry can't fix the
     start point.
   - The track logs every **30 s** (`TRACK_INTERVAL_MS`), plus a final vertex at the stop time
     (`logFinalTrackPoint()`).
   - Each detection gets a one-shot `getLocation()` fix (`maximumAge` 30 s). If that fails, save
     borrows the nearest track vertex within 60 s (`DETECTION_TRACK_FALLBACK_MS`), and otherwise
     leaves the fields blank.
   - `location_lat`/`lon` hold the start fix, falling back to the first track vertex.
5. **Record/field naming follows the toad app's convention.** Record fields are named exactly as
   in the XLSForm `survey` sheet, with no `FIELD_MAP`. `date` is the **local** date. (Goanna uses
   `toISOString()`, which gives the UTC date, i.e. the previous day for WA walks before 08:00.)
   `start_time` is local `HH:MM:SS`.
   - `walk_type` (`road` | `bush`) is required and has no default, like `recent_rain`. Records
     queued before it existed send it empty.
   - `observer_names` was dropped from the settings and the form in v2 (issue #1). The old
     form versions on Central still have the column.
6. **Submission XML with a repeat.** `buildSubmissionXml()` emits:
   - `<data id="track_ttd_survey">` with the flat fields in survey order,
   - then one `<detection>` (`species`, `species_other`, `ttd_seconds`, `detection_lat`,
     `detection_lon`) per species seen,
   - then `<meta><instanceID>`.

   The shape was copied from pyxform's compiled instance, leaving out its `jr:template` copy. A
   walk with nothing seen emits no `<detection>`, which is valid. Central exports the repeat as
   a second table (`…-detection.csv`, keyed by `PARENT_KEY`). No warehouse ETL exists for this
   form yet.

## Keeping species in sync

The species codes are in two places: `SPECIES_TREE` in the HTML, and `species_choices` in the
XLSForm. The XLSForm's extra `group` column holds the category code, so the category mapping is
also documented in the form and available to the ETL. Both were generated from one Python list
when the app was built. When adding or reordering species, edit both by hand and keep the codes
identical.
- **Never rename or reuse an existing code.** Codes are the stored data. Labels can change
  freely.
  For example, the `macropod` and `feral_herbivore` categories are labelled "Kangaroo / wallaby"
  and "Camel, cow & other feral" for rangers (issue #2), but their codes didn't change. The
  XLSForm leaf labels carry the category label as a prefix, so relabel both places.
- Adding an `other: true` leaf also means adding it to the `species_other` `relevant` expression
  in the XLSForm.
- Adding any choice means re-compiling the `.xml` and re-publishing the form on Central.

## Styling

The same CSS-custom-property shell as the sibling apps, re-themed pindan/ochre/spinifex
(`--pindan`, `--ochre`, `--spinifex`, `--rock`, `--cream`) so crews with all three apps
installed can tell them apart. The header logo and jsQR block were copied verbatim from the
goanna app. The app icon (two pairs of kangaroo hind-foot prints with tail drags, on pindan, embedded as a data URI in `manifest.json` and
the `apple-touch-icon` link) was generated with Pillow. Regenerate it rather than hand-editing the
base64.

## Notes for editing

- Keep the application in the single HTML file. `manifest.json`/`sw.js` are the only sanctioned
  exception, as in the sibling apps.
- `crypto.randomUUID()` generates the OpenRosa `instanceID` (Central's dedupe key) and the
  in-walk detection ids. Detection ids are local only and aren't submitted.
