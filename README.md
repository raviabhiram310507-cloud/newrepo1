# Drone Delivery Mission Planner

**Intelligent, Priority-Aware Drone Delivery Optimization**
Built for APSH 2026 — Algorithmic Problem-Solving Hackathon

A full-stack mission planner for a delivery drone. It models the delivery network as a weighted graph, computes real geographic distances with the Haversine formula, and plans a route with a **priority-aware greedy algorithm** that respects a hard **battery-range constraint**. A **naive nearest-neighbor algorithm** runs alongside it so the greedy strategy's behavior can be measured against a real baseline — every number shown anywhere in the app comes from an actual algorithm run, never a fake/hard-coded value.

## Features

- Graph-based delivery network (base + delivery locations as vertices, Haversine distance as edge weights)
- Greedy priority/distance algorithm with a full, inspectable decision log (every candidate considered at every step, why it was feasible or not, and why one was selected)
- Naive nearest-neighbor baseline, executed for real on every optimization — not simulated
- 3D mission visualization (React Three Fiber): a from-scratch low-poly drone model, animated propellers, hover idle animation, and a flight animation that follows the real computed route
- Interactive 2D map (Leaflet + OpenStreetMap, no API key required) kept in sync with the 3D simulation via one shared progress value
- Mission Planner with validated delivery entry, a "Load Demo Mission" shortcut, and live feedback
- Algorithm page: step-by-step explainer, the formulas in plain language, and the live decision log
- Comparison dashboard: metric cards, table, and bar chart comparing greedy vs. naive on the same mission
- Mission history persisted to PostgreSQL (Supabase-ready), with load/delete
- Dark, glass-panel "logistics command center" visual language; responsive from desktop to mobile

## Architecture

```
React + Vite (Tailwind, Leaflet, Three.js/R3F, Recharts, Framer Motion)
            │  REST API (JSON over HTTPS)
            ▼
Node.js + Express + TypeScript
  ├─ algorithms/   (graph.ts, greedy.ts, naive.ts — zero Express/DB dependency)
  ├─ services/     (validation + orchestration)
  ├─ repositories/ (Postgres repo OR local JSON-file repo — same interface)
  └─ controllers/routes/middleware
            │
            ▼
PostgreSQL (Supabase) — missions, delivery_locations, mission_results, route_steps
```

The algorithm layer (`server/src/algorithms/`) imports nothing from Express or the database layer — it is pure, testable TypeScript that takes a mission description in and returns a route + decision log out. The repository layer is swapped automatically: if `DATABASE_URL` is set, the app uses real Postgres; if not, it falls back to a local JSON file so the app is still fully usable offline during development.

## Technology Stack

**Frontend:** React, TypeScript, Vite, Tailwind CSS, Lucide React, React Router, React-Leaflet/Leaflet, Recharts, Three.js, React Three Fiber, @react-three/drei, Framer Motion, Zustand
**Backend:** Node.js, Express, TypeScript, node-postgres (`pg`)
**Database:** PostgreSQL via Supabase
**Deployment:** Vercel (frontend), Render (backend), Supabase (database), GitHub (source control)

## Algorithm

See [PROJECT_DOCUMENTATION.md](./PROJECT_DOCUMENTATION.md) for the full write-up, pseudocode, and complexity analysis. In short:

- **Greedy (priority-aware):** at each step, among all unvisited deliveries that are feasible (drone can reach them *and* still return to base), pick the one maximizing `priority / distance`.
- **Naive (nearest-neighbor):** at each step, pick the nearest feasible unvisited delivery — priority is ignored.
- Both are real **heuristics**. Greedy is not guaranteed to find the globally optimal route (that would require solving an NP-hard routing problem) — it is fast, explainable, and always battery-safe.

## Database Schema

Four tables — `missions`, `delivery_locations`, `mission_results`, `route_steps` — using UUID primary keys, foreign keys with cascading deletes, and indexes on foreign-key columns. Full DDL: [`server/src/database/schema.sql`](./server/src/database/schema.sql).

## API

| Method | Path | Description |
|---|---|---|
| GET | `/api/health` | Health + database connectivity check |
| POST | `/api/missions` | Create a mission (base + deliveries) |
| GET | `/api/missions` | List mission summaries |
| GET | `/api/missions/:id` | Get a mission |
| DELETE | `/api/missions/:id` | Delete a mission |
| POST | `/api/missions/:id/locations` | Add a delivery location |
| DELETE | `/api/locations/:id` | Delete a delivery location |
| POST | `/api/missions/:id/optimize` | Run greedy + naive and persist both results |
| GET | `/api/missions/:id/results` | Get saved results for a mission |
| POST | `/api/mission/validate` | Validate a mission payload without saving |

## Installation & Running Locally

Requires Node.js 18+ and npm.

```bash
# 1. Install dependencies
npm run install:all

# 2. Configure environment
cp server/.env.example server/.env
cp client/.env.example client/.env
# Leave server/.env's DATABASE_URL empty to use the local JSON-file fallback,
# or paste your Supabase connection string to use real Postgres.

# 3. (Optional, only with DATABASE_URL set) apply the schema
npm run --prefix server migrate

# 4. Seed a demo mission
npm run seed

# 5. Run both apps (two terminals)
npm run dev:server   # http://localhost:4000
npm run dev:client   # http://localhost:5173
```

You can also see the algorithm run with **zero installed dependencies beyond TypeScript tooling**:

```bash
cd server
npm run demo   # prints the real greedy vs. naive routes for the demo mission
```

### Database setup (Supabase)

1. Create a free project at [supabase.com](https://supabase.com).
2. Copy the connection string (Project Settings → Database → Connection string → URI) into `server/.env` as `DATABASE_URL`.
3. Run `npm run --prefix server migrate` to apply `schema.sql`.
4. Run `npm run seed` to load the demo mission.

### Testing

```bash
cd server
npm test   # vitest — Haversine correctness + all 13 required algorithm scenarios
```

## Deployment

See [DEPLOYMENT.md](./DEPLOYMENT.md) for the full GitHub → Supabase → Render → Vercel walkthrough.

## Complexity

See [PROJECT_DOCUMENTATION.md](./PROJECT_DOCUMENTATION.md#complexity-analysis) for the full analysis of the actual implementation (graph construction, Haversine, and both algorithms' best/average/worst case).

## Limitations

- The greedy algorithm is a heuristic, not an exact optimizer — see above.
- Distances are straight-line (Haversine), not real flight-restricted or road-network paths.
- Battery consumption is modeled as linear in distance; no wind, altitude, or payload-weight effects.
- Single-drone, single-base missions only.

## Future Scope

Real flight-path distance APIs, weather/wind-aware routing, dynamic battery consumption modeling, multiple drones and fleet optimization, charging stations, no-fly zones, delivery time windows, and dynamic re-routing. See `PRESENTATION_NOTES.md` for the full discussion — these are intentionally **not implemented** to keep the delivered project focused and fully real (per the project's own "no unnecessary features" requirement).

## References

- Haversine formula: [Wikipedia — Great-circle distance](https://en.wikipedia.org/wiki/Great-circle_distance)
- Greedy algorithms: Cormen, Leiserson, Rivest, Stein — *Introduction to Algorithms*
- OpenStreetMap / Leaflet documentation
- React Three Fiber documentation
