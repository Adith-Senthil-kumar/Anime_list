# Anime Watchlist — installable web app

A static site, live on GitHub Pages and installable as an app.

**→ https://adith-senthil-kumar.github.io/Anime_list/**

## Deployment
Served by GitHub Pages from the `main` branch, `/ (root)`. Every push to `main`
redeploys automatically — no build step, since it's plain HTML/JS.

## Install it as an app
- **iPhone (Safari):** open the link → Share → Add to Home Screen.
- **Android/Chrome:** open the link → menu → Install app / Add to Home screen.
- **Desktop Chrome/Edge:** click the install icon in the address bar.

Once opened online the first time, the service worker caches every poster, so it works offline afterwards. Posters are full resolution.

## Editing
Tap **Edit** in the app to change entries; edits save on that device. Posters
you drag onto a slot save too — they're downscaled to WebP and kept in
IndexedDB, so they survive reloads and app restarts. Drop targets only appear
while **Edit** is on, so a shared link stays read-only.

Use **Backup (.json)** / **Import** to move everything between devices; the
backup carries your posters as well as the list. Or edit `data.json` in the
repo directly.

Storage is per-device and per-context — nothing syncs. On iPhone in particular,
a Home Screen app has its own storage separate from Safari, so install it first
and then edit, or your Safari edits won't be there. If storage fills up, the
slot says so instead of failing quietly.

If you edit `data.json` (or any file) in the repo, also bump `CACHE` in `sw.js`
— `anime-watchlist-v1` → `-v2`, etc. The service worker is cache-first, so
anyone who has already opened the app keeps seeing the old copy until that
version string changes. Bumping it purges the old cache and re-fetches
everything on the next visit.
