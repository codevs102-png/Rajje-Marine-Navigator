# Raajje Marine Navigator

**Powered by Pabotik Investment**

A responsive Maldives-focused navigation app served by a Cloudflare Worker and backed by Supabase Auth and PostgreSQL. The Worker validates Supabase user sessions and the database enforces per-account row-level security. The browser keeps a local copy and synchronizes an account snapshot with optimistic concurrency checks. The root [`index.html`](./index.html) is a convenience redirect to the app in [`public/`](./public/index.html).

## Backend setup

You need a Cloudflare account, a Supabase project, and a current Node.js/npm installation on the deployment computer (this workspace does not include Node.js or Wrangler).

### 1. Set up Supabase

1. Create a Supabase project and open its **SQL Editor**.
2. Run [`supabase/migrations/20261003000000_user_snapshots.sql`](./supabase/migrations/20261003000000_user_snapshots.sql).
3. In Supabase **Authentication → Providers**, enable Email/Password. Configure email confirmation and password policy to suit your tester rollout.
4. Keep the project URL and the **publishable/anon key** available for Worker configuration. Never use a Supabase `service_role` key in this app or Worker.

The migration creates one account-owned JSON snapshot table, enables RLS, and adds a compare-and-swap save function. Users cannot read or overwrite another account's data.

### 2. Run the Worker locally

From this project directory in PowerShell:

```powershell
npm install
Copy-Item .dev.vars.example .dev.vars
```

Edit `.dev.vars` with your real HTTPS Supabase URL and publishable/anon key. `.dev.vars` is gitignored; do not share it or commit it. Then run:

```powershell
npm run dev
```

Open the local URL Wrangler prints (usually `http://localhost:8787`). The Worker serves the app from [`public/`](./public/index.html) and handles `/api/*`. In Supabase Auth URL configuration, add your local Worker URL to the allowed redirect/site URLs if required by your project settings.

### 3. Deploy for testers

Set production secrets without putting credentials in source control:

```powershell
npx wrangler secret put SUPABASE_URL
npx wrangler secret put SUPABASE_ANON_KEY
```

Enter each value when prompted. The Worker automatically permits requests from the same origin that serves the app. If you later host a separate frontend origin, set `ALLOWED_ORIGINS` in [`wrangler.toml`](./wrangler.toml) to its exact HTTPS origin. Then deploy:

```powershell
npm run deploy
```

Share the deployed HTTPS URL with testers. They can choose **Sign in / create account** from the top bar or **More → Account & cloud sync**. Email confirmation may need to be completed before first sign-in, depending on Supabase Auth settings.

## API

- `GET /api/health` — reports whether the Worker has Supabase configuration; no credentials are returned.
- `POST /api/auth/signup`, `/api/auth/login`, `/api/auth/refresh`, `/api/auth/logout` — constrained Supabase Auth proxy routes.
- `GET /api/sync` — reads the signed-in account's snapshot.
- `PUT /api/sync` — saves a validated snapshot using an optimistic revision. Concurrent changes return a conflict; the browser fetches the latest snapshot, merges records by ID, and retries once.

The Worker is in [`worker/index.js`](./worker/index.js), Worker settings are in [`wrangler.toml`](./wrangler.toml), and browser sync/auth flows are in [`public/app.js`](./public/app.js). Email/password credentials and bearer tokens are sent over HTTPS after deployment. The publishable/anon key is not a secret; a privileged `service_role` key is intentionally not used.

## Tester data and limitations

Routes, waypoints, tracks, vessel profile, chart-layer/night-mode preferences, and MOB position are synchronized. Guest records are merged by ID on the first sign-in; local vessel and display preferences take precedence when configured. Device-local data is kept separate for each signed-in account, so switching accounts does not upload the previous account's saved data. Large account snapshots are capped at 1 MiB and each collection at 5,000 records. Browser-local data remains if the user signs out.

This is still an early testing build, not a production marine navigation service. It does not provide licensed Maldives nautical charts, bathymetry, verified hazards, weather forecasts, verified emergency contacts, or offline marine chart downloads. Do not use it for real-world navigation decisions. Add appropriate monitoring, backups, abuse/rate controls, account recovery, privacy/retention policies, and security review before a public launch.

## App-only local preview

If Python is installed, the static interface can be previewed without the Worker API:

```powershell
python -m http.server 8000 --directory public
```

Open `http://localhost:8000`. Sign-in and cloud sync require the Cloudflare Worker; the preview server does not implement `/api/*`.
