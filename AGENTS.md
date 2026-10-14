# Working on this project

## What this is
A static portfolio site: `index.html`, `style.css`, `script.js` (currently empty) and
local images in `image/`. There is **no** build step, package manager, backend, or
database, and no external credentials are required.

## Running it
```
docker compose -f docker-compose.base44.yml up -d
```
The site is served at http://localhost:3000 by nginx (`nginx:alpine`).

## How the setup is wired (things that are easy to get wrong)
- The repository root is bind-mounted **read-only** into nginx at
  `/usr/share/nginx/html`, so nginx always serves the files as they are on disk.
  Adding a new `.html`/`.css`/`.js`/image anywhere in the repo makes it servable
  immediately — no compose change, no rebuild, no restart.
- `.base44/nginx.conf` is mounted over `/etc/nginx/nginx.conf` (the whole main
  config, not just a server block). It ships `user root;` instead of nginx's
  default `user nginx;`: the sandbox checkout root is mode `0700 root:root`, so a
  uid-101 worker cannot traverse the mount and every request would be 403. If the
  repo root ever becomes readable by other users, `user nginx;` can be restored.
- That config also denies dotfiles/dot-directories (`.git`, `.base44`) and the
  Base44 setup files, so repo internals are never published by the preview, and it
  sends `Cache-Control: no-store` so a refresh always picks up edits. Keep the deny
  list in mind when adding repo-level files that must not be served.
- Nothing here uses `BASE44_PREVIEW_MODE`; the static server needs no host/origin
  allowlisting because nginx answers any `Host` header.

## Verifying it works
```
curl -fsS http://localhost:3000/ | grep -i '<title>'
curl -fsS -o /dev/null -w '%{http_code}\n' http://localhost:3000/image/icon.png
docker compose -f docker-compose.base44.yml ps
```
`image/My photo.jpg` and `image/Nairobi city.jpeg` have spaces in their names and are
requested URL-encoded (`%20`) — that is expected and served correctly.

Edits need no rebuild or restart: write the file, then refresh (nginx reads straight
from the mounted checkout). There is no live-reload watcher, so a browser refresh —
or the preview's reload action — is what picks up a change.

## Design system (keep new work on it)
`style.css` defines one dark theme: page `#181f2b`, panels `rgba(255,255,255,0.04)`,
cards `rgba(255,255,255,0.05)`, body text `#cdd6e4`/`#e8ecf3`, accent `#ffd600`.
The shared `section, .achievements-section, .references-section` rule frames every
section (max-width, padding, radius, border, shadow), so a new section only needs the
content. Poppins is loaded from Google Fonts in `index.html`; the whole stylesheet
uses that one stack. Sections carry `scroll-margin-top` so anchor links clear the
sticky navbar.

## Notes
- `index.html` loads the Poppins webfont from Google Fonts and icons from
  `icons8.com`, and the intro overlay uses an Unsplash background; those need
  network access from the **visitor's browser**, not from a container.
- `image/icon.png` is the mobile menu button: a 64x64 PNG of white bars on a
  transparent background. It must stay transparent — the navbar is dark.
- All referenced imagery was resized for the web (longest edge 400-900px, JPEG
  quality 84-85); the served page now pulls ~640KB of images instead of ~4.3MB.
  `Card.jpeg`, `back.jpeg`, `sunday.jpeg`, `thank.jpeg` and `PRAYER.jpeg` are not
  referenced by any page.
- The hero background is `image/Nairobi city.jpeg` at only 275x183, stretched
  full-bleed, so it renders soft — only a larger photo will sharpen it.
- The intro overlay is dismissed by any menu tap, and skipped entirely when the URL
  already carries a section hash (except `#home`).
