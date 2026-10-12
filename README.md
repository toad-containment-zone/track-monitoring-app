# Track Survey Time-to-Detection — TCZ

A field app for the Toad Containment Zone (TCZ) program. It records multi-species track surveys:
start a walk, tap each species the first time you see its tracks or other sign, and the app
records how many seconds into the walk you found it. When the walk ends you add a few survey
details and the record syncs to our ODK Central form. The app logs a GPS track of the walk
automatically. It works fully offline once installed, so you don't need signal in the field.

**App URL:** [https://toad-containment-zone.github.io/track-monitoring-app/](https://toad-containment-zone.github.io/track-monitoring-app/)

## Setup (do this once)

You'll need an app user QR code or text code from whoever manages the TCZ program's ODK Central
project.

1. **Install the app on your phone.**
   - **iOS:** open the app URL in **Safari** (other browsers don't support installing), tap
     **Share**, then **Add to Home Screen**.
   - **Android:** open the app URL in **Chrome**, tap **⋮**, then **Add to Home screen**
     (or **Install app**).

   Open the app once while you have signal so it can finish setting up for offline use.
2. **Add your app user code.** Tap ⚙, then either scan the QR code with **📷 Scan QR code
   instead** or paste the text code. Tap **Save settings**.
3. When your phone asks for location permission, allow it. The app uses your location only to
   record where each walk went.

## Doing a survey

1. Tap **Start track survey** as you start walking. The clock starts and the GPS track begins.
2. The moment you see tracks of a species you haven't recorded on this walk yet, tap its
   **group** (e.g. *Goanna*). The time is recorded at that tap. Then tap the species
   (e.g. *Sand goanna*).
   - Pressed the wrong group? Tap **‹ Back**. Nothing is recorded.
   - Picked the wrong species? Tap **Undo** on the message that pops up.
   - A species you've already recorded is greyed out and shows the time you found it.
   - Not sure what it is yet (e.g. a scat you want to show the others)? Record your best guess,
     or an "unsure" option. Later, tap it under **Seen this walk** (or on the end-of-walk
     screen) to change it. The time you first saw it stays the same.
   - For something not on the list, use **Other (describe)** (in the Lizard, Native mammal and
     Other groups) and type a short description. You can record these more than once.
3. When the walk is over, tap **End walk**, then **Yes, end walk**.
4. Fill in the number of people searching (half-people are OK, e.g. 1.5 if someone joined
   partway), whether there's been recent rain, the dirt colour (red is the default), and
   whether it was a road walk or a bush walk. Tap a species name to change it, or ✕ to remove
   a mistake, then tap **Save survey**. Once saved, a survey can't be changed.
   - Ended the walk by accident? Tap 🗑, then **Resume walk**.

Records are always saved on the phone first and sync to Central automatically whenever you have
signal. The header shows how many are still queued. If the app gets closed partway through a
walk, reopen it and the walk continues where it left off.

## Files

- `track-monitoring-app.html`: the whole app.
- `track_ttd_survey.xlsx`: the XLSForm (the source of truth for the form).
- `track_ttd_survey.xml`: the same form compiled with `xls2xform`. Upload this (or the `.xlsx`)
  to ODK Central.
- `manifest.json`, `sw.js`, `index.html`: the offline and installable app shell.
