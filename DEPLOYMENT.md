# Deployment Guide

Target stack: **GitHub** (source) → **Supabase** (PostgreSQL) → **Render** (backend API) → **Vercel** (frontend).

> This guide assumes you're deploying from your own machine or CI, where `npm install` and the GitHub/Supabase/Render/Vercel dashboards are reachable. (This project was built and verified in a sandboxed environment without outbound access to npm/GitHub — see the delivery notes for what was and wasn't run there.)

## 1. GitHub

```bash
cd drone-delivery-planner
git init
git add .
git commit -m "Initial commit: Drone Delivery Mission Planner"
git branch -M main
git remote add origin https://github.com/<your-username>/drone-delivery-planner.git
git push -u origin main
```

`.gitignore` already excludes `node_modules/`, `dist/`, `.env`, and the local JSON fallback database — never commit real secrets.

## 2. Supabase (PostgreSQL)

1. Go to [supabase.com](https://supabase.com) → **New Project**. Note the database password you set.
2. Once provisioned: **Project Settings → Database → Connection string → URI**. Copy it — this is your `DATABASE_URL`.
3. Locally (or from any machine with the repo and `DATABASE_URL` set in `server/.env`):
   ```bash
   cd server
   npm install
   npm run migrate   # applies schema.sql
   npm run seed      # loads the demo mission
   ```
4. Also grab **Project Settings → API → Project URL** and **anon public key** if you want to set `SUPABASE_URL` / `SUPABASE_ANON_KEY` (not required by the current backend, which talks to Postgres directly via `pg`, but included in `.env.example` for future use with the Supabase client libraries).

## 3. Render (backend API)

1. [render.com](https://render.com) → **New → Blueprint**, point it at your GitHub repo. Render will read `render.yaml` at the repo root.
2. Alternatively, configure manually: **New → Web Service**
   - Root directory: `server`
   - Build command: `npm install && npm run build`
   - Start command: `npm start`
   - Health check path: `/api/health`
3. Set environment variables in the Render dashboard:
   - `DATABASE_URL` — your Supabase connection string
   - `CLIENT_URL` — your Vercel frontend URL (set after step 4, then redeploy)
   - `NODE_ENV=production`
   - `PORT` — Render provides this automatically; the app reads `process.env.PORT` and binds to it, as required.
4. Deploy. Confirm `https://<your-render-service>.onrender.com/api/health` returns `{"status":"ok","database":"connected",...}`.

## 4. Vercel (frontend)

1. [vercel.com](https://vercel.com) → **New Project** → import the same GitHub repo.
2. Set the project root to `client/`.
3. Framework preset: **Vite**. Build command `npm run build`, output directory `dist` (already configured in `client/vercel.json`).
4. Environment variable:
   - `VITE_API_URL` = `https://<your-render-service>.onrender.com/api`
5. Deploy. Once live, copy the Vercel URL back into Render's `CLIENT_URL` env var and redeploy the backend so CORS allows it.

## 5. Post-deploy checklist

- [ ] `GET /api/health` returns `status: ok` and `database: connected`
- [ ] Creating a mission from the deployed frontend succeeds (check the Network tab for CORS errors)
- [ ] "Load Demo Mission" works end-to-end (create → optimize → simulate)
- [ ] Mission History loads and a mission can be deleted
- [ ] No `localhost` URLs appear anywhere in the deployed frontend's network requests

## Environment variables reference

**`server/.env`**
```
DATABASE_URL=postgresql://postgres:<password>@<host>:5432/postgres
SUPABASE_URL=
SUPABASE_ANON_KEY=
PORT=4000
CLIENT_URL=https://your-app.vercel.app
NODE_ENV=production
```

**`client/.env`**
```
VITE_API_URL=https://your-api.onrender.com/api
```

Never commit either `.env` file — both are covered by `.gitignore`, and `.env.example` files are provided as templates with no real secrets.
