# Farm Dinner Check-in + Seating App

iPad-first web app for the parking-lot server. Three views built in one HTML:
1. **Check-in roster** — arriving / checked-in / all tabs, with allergens, returning-guest flags, quick notes.
2. **Seating layout** — visual A / B / C table diagram, tap a guest → tap a seat → party auto-fills adjacent seats.
3. **Sync layer** — defaults to `localStorage` (single iPad), upgrades to **Firebase Firestore** the moment a config is dropped in (multi-device realtime).

Lives at `Codename: Chakotay/Farm Dinners/checkin-app/`.

---

## Files

```
checkin-app/
├── index.html                  — the whole app (CSS + JS embedded)
├── build-checkin-data.py       — emits per-dinner JSONs from xlsx + guest-master
└── data/
    ├── index.json              — list of all known dinner dates
    ├── 2026-06-06.json         — one per dinner (regenerated on each xlsx sweep)
    ├── 2026-06-07.json
    └── ...
```

---

## How it's used

**On the iPad (parking lot, day of service):**
1. Open `https://samfermo.github.io/farm-checkin/` in Safari (add to home screen for full-screen mode).
2. The app auto-detects today's date and loads the matching dinner JSON. (Override with `?date=2026-06-07` if needed.)
3. **Check-in tab:** server taps a guest → modal shows allergens + returning history → optional quick-note ("waiting on +1") → tap **Check in**. Guest moves to the Checked-in tab.
4. **Seating tab:** when ready to seat, switch tabs. Tap a guest in the left picker, then tap an open seat. The system auto-fills the rest of their party in adjacent seats on the same side. Tap an occupied seat to clear it. End-cap checkboxes at the top enable A-left, A-right, C-left, or C-right when covers go over 26.

**On Sam's laptop / FOH iPad (if Firebase configured):** same URL, same view, realtime sync. Anyone with the URL sees who's arrived and where everyone's sitting.

---

## Regenerating the per-dinner data

The data JSONs are produced by `build-checkin-data.py`. Run it whenever the xlsx changes:

```bash
cd "Codename: Chakotay/Farm Dinners/checkin-app"
python3 build-checkin-data.py
```

This reads:
- `Codename: Chakotay/2026 Farm Dinners current.xlsx` (live source)
- `Codename: Chakotay/guest-master/guest-master.json` (for returning-guest cross-reference)

And writes per-dinner JSONs under `data/`. The Thursday xlsx sweep should call this script automatically once we wire it in.

---

## Enabling Firebase realtime sync

The app falls back to `localStorage` (per-device, no sync) when no Firebase config is present. To enable multi-device sync:

1. Go to https://console.firebase.google.com → **Add project** → name it something like `swan-house-checkin`. Free tier (Spark plan) covers your volume forever.
2. Inside the project: **Build → Firestore Database → Create database** → start in **production mode**, choose `us-west1` (Oregon) or whatever's closest.
3. **Build → Authentication → Get started → Sign-in method → Anonymous → Enable.** This lets the iPad sign in without a login.
4. **Project Settings → General → Your apps → Web (the `</>` icon)** → register a new web app (name it `farm-checkin`). Copy the `firebaseConfig` snippet.
5. Open `index.html` and find the `window.FIREBASE_CONFIG` block (around line 480). Replace it with your snippet:

   ```html
   <script>
     window.FIREBASE_CONFIG = {
       apiKey: "...",
       authDomain: "swan-house-checkin.firebaseapp.com",
       projectId: "swan-house-checkin",
       appId: "..."
     };
   </script>
   ```

6. **Firestore Rules** (in the Firebase console → Firestore → Rules tab): paste these to allow the anonymous auth user to read/write dinner docs:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /dinners/{dinner} {
         allow read, write: if request.auth != null;
       }
     }
   }
   ```

   Publish. Done.

Once configured, the footer status flips from `localStorage only` to `synced via Firebase`. Multi-device sync is automatic.

---

## Data model

Each dinner stores one document in Firestore at `/dinners/{YYYY-MM-DD}`:

```json
{
  "checkins": {
    "g-2026-06-06-001": {
      "checked_in_at": "2026-06-06T15:32:00.000Z",
      "note": "arrived early, parked by barn"
    }
  },
  "seats": {
    "B-T6": "g-2026-06-06-001",
    "B-T7": "g-2026-06-06-001"
  },
  "endcaps": {
    "A-LEFT": true
  }
}
```

Same shape on `localStorage` under key `farm-checkin-2026-06-06`.

---

## Table layout

Three tables top-to-bottom on the canvas (matches the pergola from Sam's photo):

| Table | Seats | End-caps |
|---|---|---|
| **C** (top of room, closest to garden wall) | 8 (T1-T4 top, B1-B4 bottom) | Yes — `C-LEFT`, `C-RIGHT` |
| **B** (middle, longest) | 10 (T1-T5 top, B1-B5 bottom) | No |
| **A** (closest to service entrance) | 8 | Yes — `A-LEFT`, `A-RIGHT` |

End-caps only appear visually when their checkbox is toggled on. Base capacity 26, max 30 with all four end-caps enabled.

Seat IDs:
- Regular: `{table}-{T|B}{n}` — e.g. `B-T3`, `A-B4`
- End-cap: `{table}-{LEFT|RIGHT}` — e.g. `A-LEFT`, `C-RIGHT`

---

## Future enhancements (parked)

- **Drag-and-drop** seat reassignment. Currently it's tap-select + tap-place; drag would feel more natural on iPad. Library candidate: SortableJS.
- **Floor plan overlay** — image of the actual pergola behind the table layout to make orientation obvious.
- **Pre-arrival walk-in handling** — guests who didn't pre-buy. Right now they have to be added to the xlsx first.
- **Kitchen view** — a read-only URL with just the allergen-flagged seats highlighted, for Chris to glance at from the line.
- **Print export** — when all 26 are seated, generate a printed table-by-table card the runners can carry.
- **Per-night photo capture** — server snaps a photo of the actual setup after seating, archived alongside the dinner JSON for the post-service debrief.

---

## Deploying to samfermo.github.io

The app deploys as part of the existing `samfermo.github.io` GitHub Pages repo. Put `index.html` and `data/*.json` under `farm-checkin/` in that repo, push, and the URL goes live at `https://samfermo.github.io/farm-checkin/`. The Thursday xlsx sweep can be extended to also rebuild and push the per-dinner JSONs.

(Wiring this into the existing GitHub Contents API helper is the last open piece. See cross-project-todos.)
