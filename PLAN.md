# Sherpa — feasibility and plan of attack

## Verdict

**Feasible.** The two functions you describe — record a route, follow a route — fit AllSpeak well, and the legacy repos already contain the data model and most of the product language. The work is not in the app logic; it is in the **browser APIs AllSpeak does not yet expose**. Today the runtime has `localStorage` and little else: no geolocation, no camera, no IndexedDB. Those have to arrive as plugins.

There are two constraints that shape the whole design:

1. **Background location in a PWA is unreliable.** On both iOS and Android the browser suspends `watchPosition` when the app is backgrounded or the screen locks. Recording a walk with the phone in your pocket will have gaps. The app has to be designed for screen-on recording (with a wake lock), autosave, and graceful gap handling — or later be wrapped natively (Capacitor) if true background tracking becomes essential.
2. **Google Maps needs a connection.** Map tiles are not available offline, and caching them is not permitted. Route data and photos can be cached; the basemap cannot. If routes are walked in areas with no signal, the map stack has to be MapLibre/Leaflet with offline tiles instead. I recommend keeping the map layer behind a thin interface so this can be swapped.

Everything else in your outline is straightforward: photos, editing, an offline outbox, accounts, and a spatial database.

---

## What the outline gets right

- Two primary functions, with editing as a third screen. That is the right product shape.
- Capture by walking, then edit afterwards. Capture should stay fast and dumb; editing carries the richness.
- Steps as the unit of information, with photos, descriptions and links hanging off them.
- Local-first, upload-when-connected. That is the correct default for a walking app.

## Where it needs sharpening

| Your wording | What it should become | Why |
|---|---|---|
| Routes held **in memory** until a connection | Held in **IndexedDB** until upload | Memory is lost on refresh, tab close or OS kill. A two-hour walk cannot live in RAM. |
| Save a **record** at a step | Record the **full GPS track continuously**, and save **steps as annotations** on it | Without a track, the follower has no line to draw between steps and cannot snap to the path. A step is a point of interest; the track is the route. |
| Steps added to **follow bends** | Steps are ordered by **distance along the track**, not by array index | Inserting a bend changes the geometry. Distance-from-start is stable across simplification and insertion; an index is not. |
| **Database finds the closest saved location** | PostGIS **KNN** query over step/place points, plus route proximity | The query is easy once the data is spatial. But it only matters after sync exists; it is not MVP. |
| Map **follows as the user moves** | A **follow mode** with an auto-recentre toggle | Users need to pan and inspect without the map yanking back. Follow mode should pause on gesture and resume on a button. |
| Auto-record when moved **more than a distance** | Distance **plus** accuracy and speed filters, with hysteresis | Raw GPS jitter will otherwise create a step every time the user stands still. |
| Photos added to the step | Photos stored **separately, compressed, EXIF-stripped**, referenced by ID | Inline base64 bloats the route file and makes sync fragile. The legacy Sherpa did base64; we should not repeat it. |
| Download a route, then run it | Download **route + photos** into local storage, verify, then run offline | "Downloaded" must mean "complete and self-contained", not "metadata fetched". |

## The runtime gap (verified)

| Need | Available today | Work required |
|---|---|---|
| Map + markers | `gmap` plugin: `gmap`, `marker`, centre/zoom/bounds, click | Extend for **polyline**, user-position marker, fit-bounds, travelled/remaining styling |
| Current position | Nothing (`navigator.geolocation` not exposed) | New `geo` plugin: permission, `watchPosition`, accuracy/heading/speed, wake lock |
| Photo capture | Nothing | New `photo` plugin: camera/file input, resize, JPEG encode, EXIF strip, thumbnail |
| Offline route/photo storage | `localStorage` only | New `store` plugin over **IndexedDB**: routes, photo blobs, draft, upload outbox |
| Geometry / map matching | Nothing | New `routegeo` plugin: haversine, nearest point on segment, along-track progress, simplify |
| PWA install / offline shell | Not an AllSpeak concern | Plain `manifest.webmanifest` + `sw.js` registered from the launcher HTML |
| Accounts / sync | `rest` is available | Backend + a small sync module in AllSpeak |

The plugin split should stay narrow: one concern per plugin, roughly 150–400 lines each. AllSpeak then owns screens, state and workflow, which is exactly the mixed-stack principle the language is designed around.

---

## Architecture

```
sherpa.html                  launcher; loads runtime, plugins, manifest, service worker
sherpa.json                  Webson layout: map pane, step list, sheets/popups
sherpa-main.allspeak         app state machine: Home → Record / Follow / Edit
sherpa-route.allspeak        route model, validation, import/export
sherpa-record.allspeak       recorder screen
sherpa-edit.allspeak         editor screen
sherpa-follow.allspeak       follower screen
sherpa-sync.allspeak         outbox flush, auth, download

plugins/
  sherpa-geo.js              geolocation watch, permission, wake lock
  sherpa-photo.js            camera/gallery, resize, EXIF strip
  sherpa-store.js            IndexedDB: routes, photos, drafts, outbox
  sherpa-routegeo.js         geometry + map matching
  sherpa-map.js              map adapter (Google Maps first; MapLibre possible)
```

`gmap` can be extended rather than replaced for the first version; `sherpa-map.js` exists so the follower/editor do not depend directly on Google.

### Backend (later phase)

Postgres + **PostGIS**, object storage for photos, and auth. Supabase is the fastest route to all three (Postgres/PostGIS, Auth, Storage, Row Level Security, REST), and `rest` in AllSpeak talks to it directly. A self-hosted Postgres + API is equally valid if you prefer to own it. `server.allspeak` is the development server; it is not a production host.

Core spatial queries:

```sql
-- nearest steps to a position
select s.*, st_distance(s.position, p) as metres
from steps s, (select st_point(:lon, :lat)::geography as p) q
order by s.position <-> q.p
limit 20;

-- routes passing within a radius
select r.*, st_distance(r.track, p) as metres
from routes r, (select st_point(:lon, :lat)::geography as p) q
where st_dwithin(r.track, q.p, :radius)
order by r.track <-> q.p
limit 10;
```

GiST indexes on `steps.position` and `routes.track` make both queries cheap. This is what lets routes "feed each other": a nearby query returns steps and places from other routes, filtered by visibility.

---

## Data model

Separate **track** (dense geometry) from **steps** (sparse annotations). This is the most important modelling decision.

```json
{
  "schema": "sherpa.route/1",
  "id": "uuid",
  "name": "Coventry historic cars",
  "description": "Markdown",
  "mode": "walk",
  "createdAt": "2026-10-23T10:00:00Z",
  "updatedAt": "2026-10-23T12:00:00Z",
  "revision": 1,
  "visibility": "private",
  "track": {
    "encoding": "array",
    "points": [[52.4081, -1.5106], [52.4084, -1.5102]]
  },
  "distanceMetres": 1840,
  "steps": [
    {
      "id": "uuid",
      "name": "Hillman Minx",
      "description": "Markdown",
      "position": { "lat": 52.4082, "lon": -1.5104 },
      "distanceFromStartMetres": 420,
      "triggerRadiusMetres": 25,
      "photos": [{ "id": "uuid", "caption": "Front grille" }],
      "links": [{ "title": "Coventry Transport Museum", "url": "https://..." }]
    }
  ],
  "photos": [
    { "id": "uuid", "file": "photos/<hash>.jpg", "width": 1600, "height": 1200 }
  ]
}
```

- `distanceFromStartMetres` is canonical ordering. The editor recomputes it whenever the track changes.
- `position` duplicates the track point at that distance, so the database can index it without parsing the geometry.
- Photo blobs live outside the JSON, referenced by content hash. Export is a zip (`.sherpa`) containing `route.json` and `photos/`.
- GPX import/export is a cheap addition and gives interoperability with every other walking app.

### Sync model

- Client-generated UUIDs; no server round trip needed to start a route.
- Outbox operations: `route.upsert`, `photo.upload`, `route.delete`, each with `opId`, `baseRevision`, retry count and next-attempt time.
- Flush on app start, on `online`, and on a manual "sync now".
- Photos upload first (content-addressed, idempotent), then the route references them.
- Conflicts: server returns `409` with its revision; the user chooses keep-mine / keep-theirs / duplicate. Last-write-wins silently is not acceptable for a route someone walked.
- Delete is a tombstone until acknowledged.

---

## Recording behaviour

- `watchPosition` with high accuracy; discard fixes with accuracy worse than ~30 m.
- Reject impossible jumps (speed above a mode-specific ceiling) unless the next fix confirms them.
- Write a track point every ~5 s or ~10 m, whichever comes first; simplify on save.
- **Manual step** is always available and always wins.
- **Auto step**: record when distance from the last step exceeds a configurable threshold (default 50 m), accuracy is good, and the user is actually moving. Add a minimum time between auto-steps.
- Draft autosave to IndexedDB every few points; recover on reload. This is not optional given background suspension.
- Wake lock while recording; release on pause/finish.
- Finish writes the route locally and queues it for upload.

## Following behaviour

- Snap the current position to the track: nearest segment, cross-track distance, along-track distance.
- Progress = along-track distance / total. The next step is the first one ahead of the current position (with a small lead).
- Arrival fires once inside the trigger radius; departure fires once outside radius + hysteresis. No repeat spam while standing at a step.
- Off-route warning after sustained cross-track distance (e.g. 50 m for 10 s), showing direction and distance back to the route.
- Map shows travelled vs remaining track, numbered step markers, and the user marker with heading.
- Sidebar/sheet shows the next step and the full list; tapping a step opens its description, photos and links.
- A **simulation mode** that replays a recorded track at adjustable speed is essential. It is the only sane way to test arrival logic without repeatedly walking the route.

## Editing behaviour

- Route fields: name, description, tags, visibility.
- Step fields: name, description (Markdown), photos, links, trigger radius.
- Map editing: tap a marker to edit, drag to move, tap the line to insert a step at the projected point, delete, reorder, reverse.
- Inserting a bend is a track edit: project the tap onto the nearest segment, insert the point, recompute distances, shift downstream steps.
- Undo/redo over a command stack; autosave; explicit publish/sync.
- Drag-to-move and insert-on-line are the expensive parts. The first version can use a list plus "add step at map centre" and still be useful.

---

## Plan of attack

### Phase 0 — walking skeleton and de-risking (1–2 sessions)

Build the smallest thing that proves the hard parts:

- PWA shell with manifest and service worker.
- Google Map centred on the current position.
- `geo` plugin: permission, watch, accuracy, wake lock.
- Record a 10-point track and 3 steps, save to IndexedDB, reload, draw the line.
- Test on a real Android phone and a real iPhone, screen on and screen off.

**Exit criteria:** you know how the app behaves when the screen locks, how accurate the fixes are, and whether the map is usable where you walk. If screen-off recording is a dealbreaker, decide now whether to accept gaps or plan a Capacitor wrapper.

### Phase 1 — foundation (3–5 sessions)

- Initialise the AllSpeak GUI project (`sherpa.html`, `sherpa-main.allspeak`, `sherpa.json`, `.allspeak-init`).
- Define `sherpa.route/1` and the zip export/import.
- `store` plugin: routes, photo blobs, drafts, outbox.
- `routegeo` plugin with unit tests: distance, bearing, snap-to-track, progress, simplify, reverse.
- Extend the map layer: polyline, user marker, fit-bounds.

**Exit criteria:** a route file can be created, saved, reloaded, exported, imported and drawn — with no network.

### Phase 2 — recorder (1–2 weeks)

- Map follow mode with auto-recentre toggle and zoom control.
- Start/pause/finish; live track; manual step with optional name; auto-step.
- Photo capture and thumbnail.
- Draft autosave and crash recovery; step list with edit/delete.
- Finish → route saved locally, queued for upload.

**Exit criteria:** walk a real route, kill the app mid-walk, reopen, recover, finish, export. The exported file reloads and matches what you walked.

### Phase 3 — editor (1–2 weeks)

- Route and step metadata; photos, captions, description, links.
- Reorder, delete, insert at bend, reverse.
- Validation (a route needs at least two track points; steps need positions).
- Undo/redo and autosave.

**Exit criteria:** a route recorded in Phase 2 can be tidied into something a stranger could follow, entirely on the phone.

### Phase 4 — follower (1 week)

- Load a local route; draw travelled/remaining; current position.
- Next-step card, full step list, step detail with photos/links.
- Arrival/departure triggers with hysteresis; off-route warning.
- Simulation mode.

**Exit criteria:** replay a recorded track and get the correct step sequence; then follow it outdoors.

**This is the MVP: record → edit → follow, offline, on one device, with file export.**

### Phase 5 — accounts, sync, nearby (1–2 weeks)

- Auth and route ownership.
- Outbox flush with retry/backoff; photo upload; conflict handling.
- Download a route and its photos for offline use.
- PostGIS schema, GiST indexes, nearby query.
- Sharing: unlisted link / QR; visibility rules.
- "Nearby" panel: steps and places from other routes within a radius.

**Exit criteria:** record on the phone offline, sync, download on another device, follow it offline. A nearby query returns sensible results.

### Phase 6 — hardening

- Battery profiling, GPS gap handling, permission-denied UX.
- Storage persistence (`navigator.storage.persist()`), export reminders.
- Accessibility, large-text and one-handed use.
- Optional: spoken guidance (the legacy `On Arrival` / `On Departure` messages). Speech synthesis is a small plugin and was core to the old product; it is worth adding once the visual flow works.
- Optional: Capacitor wrapper for true background tracking.

---

## Testing

- `node --test` for the geometry plugin: known-distance pairs, snap-to-segment, progress through a route, simplify preserving endpoints, reverse symmetry.
- Playwright with mocked geolocation: replay a GPX, assert arrival order, off-route detection, and that steps fire once.
- Offline simulation in DevTools: record, reload, follow with the network disabled.
- Real-device field tests at each phase; battery and accuracy notes kept alongside the route.
- AllSpeak `check` / `test` for the state machine and route validation.

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Background tracking stops | Gaps in recorded routes | Screen-on recording, wake lock, autosave, gap markers; native wrapper if unacceptable |
| Offline basemap unavailable | Blank map where there is no signal | Online-only MVP with Google; design the map adapter so MapLibre + offline tiles can replace it |
| GPS drift | False steps, false arrivals, wandering track | Accuracy filter, speed gate, hysteresis, manual confirmation, track simplification |
| Storage eviction | Lost routes and photos | Persistent storage request, autosave, export, sync early |
| Photo size | Slow upload, huge exports | Resize to 1600 px, JPEG ~0.75, content-addressed uploads, separate blobs |
| Google API key/cost | Abuse or surprise bill | Referrer-restricted key, quotas, billing alerts |
| Editing complexity | Schedule creep | Ship list-based editing first; add drag/insert-on-line second |
| Location privacy | Sensitive data exposure | TLS, RLS, unlisted share tokens, EXIF stripping, explicit visibility |

## Reuse from the old repos

**Reuse:** the My Guide product model (guide → ordered steps → arrival/departure), the Sherpa field names and help copy, the revision/sync ideas from `checkRevisions`, the exported CTM Historic Cars content as a demo route, and the Scanna Pro keyword list as a vocabulary checklist.

**Do not reuse:** the PHP CMS, `edit_area`, the Android/Java code, or base64-in-JSON photo storage.

## Decisions needed before Phase 1

1. **Map stack.** Google Maps online (matches your outline, simplest) or MapLibre/OSM with offline tiles (works without signal, more work). Recommendation: Google first, behind an adapter.
2. **Backend.** Supabase (fastest: Postgres/PostGIS, auth, storage) or self-hosted Postgres + API. Recommendation: Supabase.
3. **Offline expectation.** Is "record anywhere, map visible" required, or is "record anywhere, map needs signal" acceptable?
4. **Spoken guidance.** Keep it in scope from the start, or defer?
5. **Photos in the route file.** Zip bundle (recommended) or loose files plus a JSON manifest?
6. **Target devices.** Which phones will you actually test on? This determines how much iOS-specific work is needed.

Once these are settled, Phase 0 is a short, concrete build: the shell, the geo plugin, a map, and a saved track.
