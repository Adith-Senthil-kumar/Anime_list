# Anime Watchlist — installable web app

A static site. Host it free on GitHub Pages and install it as an app.

## Put it on GitHub Pages
1. Create a new GitHub repository (e.g. `anime-watchlist`).
2. Upload EVERYTHING in this folder to the repo (index.html, the .js files, data.json, manifest.webmanifest, sw.js, the icons, and the `images/` folder).
3. Repo → Settings → Pages → Source: `main` branch, `/ (root)` → Save.
4. After a minute your app is live at `https://<your-username>.github.io/anime-watchlist/`.

## Install it as an app
- **iPhone (Safari):** open the link → Share → Add to Home Screen.
- **Android/Chrome:** open the link → menu → Install app / Add to Home screen.
- **Desktop Chrome/Edge:** click the install icon in the address bar.

Once opened online the first time, the service worker caches every poster, so it works offline afterwards. Posters are full resolution.

## Editing
Tap **Edit** in the app to change entries; edits save on that device. Use **Backup (.json)** / **Import** to move them between devices, or edit `data.json` in the repo directly.
