# Deploying Kitsunescan

Run it locally, push it to GitHub, ship it on Vercel.

## Contents

1. [Run it locally](#1-run-it-locally)
2. [Push to GitHub](#2-push-to-github)
3. [Deploy on Vercel](#3-deploy-on-vercel)
4. [Verify the deployment](#4-verify-the-deployment)
5. [Troubleshooting](#5-troubleshooting)
6. [MangaDex API compliance](#6-mangadex-api-compliance)

---

## 1. Run it locally

Requires **Node 18+** (the server uses built-in `fetch`). Check with `node -v`.

```bash
npm run install:all     # installs root + server + client
npm run dev             # Express on :3001, Vite on :5173
```

Open <http://localhost:5173>. Vite proxies `/api/*` through to Express, so the whole app works in dev.

Quick health check:

```bash
curl http://localhost:3001/api/health
# {"ok":true,"cache":{...},"staleCache":{...},"proxied":false,"upstream":{...}}
```

To test the production path (Express serving the built client itself):

```bash
npm start               # builds the client, then serves everything on :3001
```

Open <http://localhost:3001>.

---

## 2. Push to GitHub

Create an **empty** repo on GitHub (no README, no .gitignore, no license). Then, from the project folder:

```bash
git init                # skip if the folder is already a git repo
git add -A
git commit -m "Initial commit"

git branch -M main
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

If `git remote add` says the remote already exists, update it instead:

```bash
git remote set-url origin https://github.com/<your-username>/<your-repo>.git
```

Before pushing, sanity-check what's going up:

```bash
git status --short
git ls-files | head -30
```

`node_modules/`, `dist/`, `.env` and `*.log` are in `.gitignore` and stay local. That is intended: Vercel builds `client/dist` itself, so a committed build would just go stale.

The README screenshots live in `docs/screenshots/` and **are** committed, so they render on GitHub.

---

## 3. Deploy on Vercel

### Import the repo

1. Go to <https://vercel.com/new>.
2. **Import Git Repository** → pick your repo (authorize the Vercel GitHub app if prompted).
3. **Root Directory**: leave it at `./`. The monorepo scripts are wired from the root.
4. **Framework Preset**: **Other**. Do *not* pick Vite; `vercel.json` already specifies the build and a preset can fight with it.
5. Leave the Build / Output / Install command fields blank. `vercel.json` supplies all three.
6. **Deploy**.

### What `vercel.json` does

```json
{
  "installCommand": "npm run install:all",
  "buildCommand": "npm run build",
  "outputDirectory": "client/dist",
  "redirects": [
    { "source": "/manga/:id/chapter/:chapterId", "destination": "/read.html?manga=:id&chapter=:chapterId", "permanent": true },
    { "source": "/manga/:id", "destination": "/manga.html?id=:id", "permanent": true }
  ],
  "rewrites": [
    { "source": "/api/(.*)", "destination": "/api" },
    { "source": "/((?!api/).*)", "destination": "/index.html" }
  ]
}
```

- The React app is built to `client/dist` and served from Vercel's CDN (multi-page build: `index.html`, `manga.html`, `read.html`).
- `api/index.js` is auto-detected as a serverless function. It re-exports the same Express app as `server/app.js`, so your local server and deployed API can't drift apart.
- The two redirects send clean URLs (`/manga/<id>`, `/manga/<id>/chapter/<id>`) to the matching page.
- Rewrite 1 funnels every `/api/*` path into that one function. **Order matters:** it must come before rewrite 2.
- Rewrite 2 is the fallback to `index.html` for any other path.
- `server/app.js` checks `process.env.VERCEL` and skips static-file serving there, since the CDN handles that.
- `app.set('trust proxy', 1)` is enabled so `express-rate-limit` sees each visitor's real IP from `X-Forwarded-For` instead of treating everyone as one client.

### Environment variables

None are required, but set the first one. Add them under **Project → Settings → Environment Variables**, then redeploy.

| Variable | Purpose |
|---|---|
| `MANGADEX_CONTACT` | Goes into the `User-Agent` sent to MangaDex. Their docs require a User-Agent that isn't spoofed, and a real contact URL lets them reach you instead of banning you if this deployment misbehaves. Use your repo URL or an email. |
| `MANGADEX_BLOCKED_GROUPS` | Comma-separated scanlation-group UUIDs. Chapters from these groups are hidden from lists and return HTTP 451 when opened. Use this to honour a group's removal request. |
| `UPSTREAM_PROXY_URL` | Routes all MangaDex calls through a proxy you control, e.g. `http://user:pass@host:port`. Vercel functions egress from a shared, rotating IP pool and MangaDex rate-limits and bans per IP, so requests can fail intermittently through no fault of your own. A static-IP proxy (cheap VPS, QuotaGuard, Fixie) gives them one stable IP to see. `HTTPS_PROXY` works as a fallback name. **The proxy must be transparent:** MangaDex rejects any request carrying a `Via` header. |
| `MANGADEX_REPORT` | Set to `false` to disable MangaDex@Home network reports. Leave it unset: their docs ask every client to report, and turning it off makes the volunteer CDN worse for everyone. |

---

## 4. Verify the deployment

```bash
curl https://<your-project>.vercel.app/api/health
```

A JSON body with `"ok": true` means the function is live and routing correctly. If that works but the gallery is empty, the problem is upstream (MangaDex / Cloudflare), not your deploy. Try `/api/manga?limit=1`.

Then open the site and check that the gallery loads covers, search returns results, a chapter opens and pages render, and refreshing on a chapter URL doesn't 404.

Every push to `main` triggers a production deploy from here on. Pushes to other branches get preview URLs.

---

## 5. Troubleshooting

**Build fails: "Could not parse File as JSON"**: a BOM or trailing comma crept into `vercel.json`. Check with:

```bash
head -c 3 vercel.json | od -An -tx1     # want "7b 0a 20", not "ef bb bf"
node -e "JSON.parse(require('fs').readFileSync('vercel.json','utf8'))"
```

VS Code: bottom-right encoding indicator → **Save with Encoding** → **UTF-8** (not "UTF-8 with BOM").

**Site loads, no manga, `/api/*` returns 404**: the `/api/(.*)` rewrite is missing or sits after the fallback rewrite.

**`/api/*` returns 500**: open **Vercel → Deployments → your deployment → Functions** and read the logs. Usually an upstream timeout or a Cloudflare block on the shared IP pool; see `UPSTREAM_PROXY_URL` above.

**Deep links 404 on refresh**: confirm `outputDirectory` is `client/dist` and that `index.html`, `manga.html` and `read.html` exist in the build output.

**`npm run dev` fails with `concurrently: not found`**: re-run `npm run install:all`.

**Blank page, console errors about MIME types**: Framework Preset got set to Vite and is overriding the output directory. Set it to **Other** and redeploy.

**Intermittent "The manga source is temporarily unreachable"**: this is the app's own message after its retries are exhausted. Expected occasionally on Vercel's shared IPs; a static-IP proxy is the real fix.

**Screenshots don't show on GitHub**: make sure `docs/screenshots/` was committed (`git ls-files docs`) and that paths in `README.md` are relative, not absolute.

---

## 6. MangaDex API compliance

The app follows the rules in [api.mangadex.org/docs](https://api.mangadex.org/docs/). Breaking these gets your server's IP banned rather than just erroring. See [MANGADEX-API-AUDIT.md](MANGADEX-API-AUDIT.md) for the full audit.

| Rule (from the docs) | Where it's handled |
|---|---|
| ~5 req/s per IP, globally | Outbound token bucket in `server/app.js`, set to 4/s for headroom |
| `GET /at-home/server/{id}`: 40/min | Second bucket, applied only to that route |
| Don't keep sending while being 429'd | A 429 sets a server-wide cooldown; all upstream calls stop until it clears, and stale cache is served meanwhile |
| `User-Agent` required, must not be spoofed | Honest UA built from `MANGADEX_CONTACT` |
| No `Via` header (no non-transparent proxies) | Only relevant if you set `UPSTREAM_PROXY_URL`; use a transparent proxy |
| Must proxy images, never hotlink | `/api/image` streams them server-side |
| No auth headers to image hosts | None are sent. An @Home node is a third party's machine |
| `offset + limit > 10,000` is rejected; `limit` ≤ 100 | Clamped in both `client/src/api.js` and `server/app.js`; the gallery's page buttons stop at the last reachable page |
| Report every @Home image load, success or failure | `POST https://api.mangadex.network/report` after each image from a non-`mangadex.org` host |
| Base URLs are guaranteed for only 15 minutes | A failed page image triggers a fresh `/at-home/server/:id` call, capped at 3 per chapter |
| Don't hardcode base URLs | Always taken from the at-home response |
| Credit scanlation groups; honour removal requests | Group name shown on chapters; `MANGADEX_BLOCKED_GROUPS` |

`GET /api/health` reports the live state of all of this:

```json
{ "upstream": { "cooldownMs": 0, "globalTokens": 4, "atHomeTokens": 40, "reports": true, "blockedGroups": 0 } }
```

A `cooldownMs` that stays high means the IP you're egressing from is being throttled. That's the signal to set up `UPSTREAM_PROXY_URL`.

### The serverless caveat

The rate limiters live in process memory. On Vercel each function invocation may run in its own isolate, so they only coordinate requests that share a warm instance, and under real traffic your true outbound rate can exceed the bucket. A single long-lived Node server (`npm start` on a VPS, Railway, Fly, Render) enforces them properly. If you stay on Vercel, `UPSTREAM_PROXY_URL` pointed at one machine you control is the next best thing.
