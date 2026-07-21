# Poster image persistence on GitHub Pages

**Date:** 2026-07-21
**Status:** Approved

## Problem

Posters dragged into an `<image-slot>` do not persist when the app is served
from GitHub Pages, and the failure is invisible to the user.

`image-slot.js` persists through a host-injected API:

- **Read:** `fetch('.image-slots.state.json')`
- **Write:** `window.omelette.writeFile(...)` (`image-slot.js:218`, `:224`)
- **Edit gate:** `!!(window.omelette && window.omelette.writeFile)`
  (`image-slot.js:1066`)

`window.omelette` is undefined on GitHub Pages (verified against the deployed
site). Three consequences:

1. `save()` hits `if (!w) return;` and abandons the write with no error.
2. The drop/reframe UI is hidden, because editability is gated on the same
   check — deliberate, so shared links stay read-only.
3. `load()`'s fetch 404s, so nothing is restored even if a write had landed.

Text edits are unaffected: they persist to `localStorage` under `aw_data_v1`
(`index.html:196`) and work correctly.

## Goals

- Dragged posters survive reload, app restart, and reboot.
- A failed save is never mistaken for a successful one.
- Backups round-trip posters, not just text.
- No regression to the authoring host, where `omelette` is present.

## Non-goals

- Cross-device sync. Transfer stays manual, via Backup/Import.
- Reconciling `data.json` updates with local edits. Deferred by decision;
  `localStorage` continues to win outright once any edit exists.
- A build step or test framework. The project stays a plain static site.

## Design

### Storage backend abstraction

Both coupling points move behind a backend with `read()` and `write(slots)`:

```js
backend = (window.omelette && window.omelette.writeFile)
  ? omeletteBackend
  : idbBackend
```

Resolved per call rather than at module init, because the host may inject
`omelette` after first render (`image-slot.js:712`). When present it wins — it
is the authoring host and authoritative.

Both backends serialize the identical `slots` object, so the existing
`{u, s, x, y}` shape, the hydration merge, and the tombstone logic are
unchanged.

**IndexedDB, not localStorage.** A poster is a 150–300KB WebP, so roughly 20 of
them exceed localStorage's ~5MB quota. localStorage is also synchronous, so
large writes block the main thread.

- Database `anime-watchlist`, object store `kv`, key `image-slots`.
- `navigator.storage.persist()` requested once, opportunistically, to reduce
  the chance of eviction under storage pressure.

### Load path

`load()` keeps its sidecar `fetch` as a base layer and overlays IndexedDB on
top, IndexedDB winning per key. A committed `.image-slots.state.json` therefore
acts as repo-provided defaults that a local drop can override. The existing
merge-and-tombstone behaviour is preserved as-is.

### Error surfacing

The current code swallows every failure: `if (!w) return;` and
`.catch(() => {})`. Write failures — quota exceeded, IndexedDB unavailable in
private browsing — surface inline on the affected slot and via `console.warn`.

This is the defect class the whole change exists to fix, so silence is not an
acceptable outcome for any failure path.

The message cannot live in `.empty`: that block is `display:none` the moment a
slot holds an image, which is precisely the failing case — a poster visible on
screen that was never committed. It renders instead in a `.save-error` band
across the bottom of the frame, driven by a `data-save-error` host attribute in
the same idiom as the existing `.attr-error` overlay. A band rather than a full
cover, so the image stays visible; one clipped line with the full sentence in
`title`, because these slots render ~60px wide and a wrapping message grows
until it hides the poster it is describing.

The band shows only on the slot whose write failed, tracked by recording the id
passed to `setSlot`. The underlying condition is usually global (a full quota),
and banding all 228 posters at once would bury the one the user just touched.

### Edit gate

The gate becomes "a writable backend exists **and** the app is in edit mode."

`toggleEdit()` in `index.html` sets `data-aw-edit` on `document.documentElement`;
`image-slot.js` watches that attribute with a `MutationObserver` and re-renders
through the existing `subs` broadcast. One line of coupling on each side, no
module imports.

Reposition writes need no debounce: `_commitView()` fires once when reframe
exits, not per pointer event, so writes are already infrequent.

They do need a second commit point. `pagehide` alone is not enough once the
backend is asynchronous — an unloading document never runs the callbacks, and
iOS routinely kills a backgrounded home-screen app without firing `pagehide` at
all. Committing on `visibilitychange` → hidden runs while the document is still
alive, so the write has time to land.

### Backup format v2

```json
{ "version": 2, "data": [ ... ], "images": { "poster-xyz": { "u": "data:image/webp;base64,..." } } }
```

`importBackup` accepts both shapes. Existing backups are bare arrays, so
`Array.isArray(d)` distinguishes them; a v2 object restores text and images
together. Only user-dropped posters are embedded — the 271 repo posters remain
path references, keeping backups proportional to what the user actually added.

Export requires reading slot state from `image-slot.js`, so it exposes a narrow
public API: `window.imageSlots.export()`, `.import(obj)`, and `.ready()`.

That API must be installed under the same `!customElements.get('image-slot')`
guard as the element definition. The file is evaluated more than once, and only
the first evaluation's class is registered; installing the API unconditionally
publishes a later closure's `slots`, which no live element ever writes to. The
symptom is quiet and confusing — drops persist correctly through the real
closure while `export()` reads empty, so backups come out missing every poster.

### WebP fallback

`toDataURL('image/webp')` silently returns PNG on browsers without WebP encode
support (older iOS Safari), roughly 10× larger and quota-hostile. The encoder
verifies the returned data URL's MIME type and falls back to JPEG when WebP was
not honoured.

## Files

| File | Change |
| --- | --- |
| `image-slot.js` | Backend abstraction, IndexedDB, error surfacing, edit gate, export/import API, WebP check |
| `index.html` | Set `data-aw-edit`; backup/import v2 |
| `sw.js` | `CACHE` → `anime-watchlist-v2` |
| `README.md` | Document persistence behaviour and limits |

The `sw.js` bump is required, not cosmetic: the service worker is cache-first,
so returning visitors would otherwise keep the old `index.html` and
`image-slot.js` indefinitely.

## Verification

No test harness exists and none is added — there is no `package.json`, and a
runner would be disproportionate to a static site of this size. Verification is
browser-driven against the real page, with observed output reported rather than
asserted:

1. Drop an image, hard-reload, confirm it survives.
2. Confirm the slot is inert outside edit mode and active inside it.
3. Force a quota failure, confirm it is visible rather than silent.
4. Round-trip a v2 backup through export and import.
5. Import a legacy bare-array backup, confirm it still loads.
6. Confirm the authoring host path still writes through `omelette` when present.

Two cautions learned while running these. The service worker is cache-first, so
it will serve a stale `image-slot.js` mid-session and make a fixed bug look
unfixed; unregister it and clear caches before trusting a result. And assert on
storage, not just on screen — the double-evaluation bug rendered a correct
image from one closure while another reported an empty store, so a screenshot
alone would have passed it.
