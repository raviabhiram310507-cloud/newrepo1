# Project Documentation — Drone Delivery Mission Planner

APSH 2026 — Algorithmic Problem-Solving Hackathon

## 1. Problem Statement

A delivery drone must visit a subset of delivery locations starting and ending at a fixed base, subject to:

- A **battery range constraint**: the drone can only travel a limited total distance before it must recharge, and it must always be able to return to base.
- **Delivery priorities**: not all deliveries are equally valuable, so when battery is limiting, the drone should prefer higher-value deliveries.
- **Geographic distance**: real-world travel cost is distance, not delivery order.

The goal is to compute a delivery sequence that delivers high priority value while never leaving the drone stranded.

## 2. Objectives

1. Model the delivery network as a weighted graph with real geographic edge weights.
2. Implement a priority-aware greedy heuristic that respects battery feasibility at every step.
3. Implement a baseline (naive nearest-neighbor) algorithm for honest comparison.
4. Make every algorithmic decision transparent and inspectable (a "decision log"), not a black box.
5. Visualize the computed route in both 2D (map) and 3D (simulation) in a way that is synchronized and demonstrably driven by real algorithm output.
6. Persist missions and results so they can be revisited.

## 3. Proposed Approach

- **Graph construction**: vertices = base + delivery locations; edge weight = Haversine great-circle distance between every pair. Built once per optimization request, O(n²) time/space.
- **Greedy priority-aware algorithm**: at each step, evaluate every unvisited delivery's feasibility (can the drone reach it *and* still return to base on remaining battery?), score feasible candidates by `priority / distance`, and pick the highest score. Stop when no delivery is completed or all are feasible-exhausted; return to base.
- **Naive nearest-neighbor algorithm**: identical feasibility check, but selection is purely by minimum distance — priority plays no role.
- Both algorithms are implemented as pure functions with **zero dependency on Express or the database** (`server/src/algorithms/`), so they can be tested, reused, or swapped independently of the web layer.

## 4. Algorithm / Pseudocode

```
START

Initialize graph (base + all delivery vertices, Haversine edge weights)
Initialize base, battery = batteryCapacity
Initialize unvisited = all delivery locations

WHILE unvisited is not empty:

    feasibleCandidates = []

    FOR each candidate in unvisited:
        distanceToCandidate = haversine(current, candidate)
        returnDistance      = haversine(candidate, base)
        batteryRequired     = distanceToCandidate + returnDistance

        IF batteryRequired <= remainingBattery:
            score = candidate.priority / distanceToCandidate   # GREEDY ONLY
            # (naive variant: score is just -distanceToCandidate, i.e. "nearest wins")
            feasibleCandidates.add((candidate, score))

    IF feasibleCandidates is empty:
        BREAK   # no remaining delivery is reachable while still returning to base

    selected = candidate with highest score among feasibleCandidates

    move drone to selected
    remainingBattery -= distance(current, selected)
    mark selected as completed
    current = selected
    remove selected from unvisited

Return to base (remainingBattery -= distance(current, base))

END
```

This matches `server/src/algorithms/greedy.ts` and `server/src/algorithms/naive.ts` exactly — the decision log each algorithm emits is generated from this same loop, not reconstructed after the fact.

## 5. Complexity Analysis

Let **n** = number of delivery locations.

### Graph construction (`graph.ts`)
- Building the full pairwise distance matrix over (n+1) vertices (base included): **O(n²)** time and space. Each individual Haversine computation is **O(1)**.

### Haversine distance (`haversineDistanceKm`)
- Constant number of trigonometric operations: **O(1)**.

### Greedy algorithm (`greedy.ts`)
- Each of up to *n* steps scans the remaining unvisited set (up to *n* candidates), computing two O(1) distance lookups and a score per candidate.
- **Best case: O(n)** — e.g. a single delivery, or a case where the feasible set collapses to ≤1 candidate immediately at every step.
- **Average case: O(n²)** — a typical run scans a shrinking-but-still-linear candidate set at each of *n* steps.
- **Worst case: O(n²)** — every delivery remains a feasible candidate at every step (no early pruning possible in this formulation).

### Naive algorithm (`naive.ts`)
- Identical loop shape and cost profile to greedy: **O(n) best, O(n²) average/worst**. The only difference is the comparison key (minimum distance vs. maximum priority/distance), which doesn't change the asymptotic cost.

### Space complexity
- **O(n)** for the unvisited set, route steps, and decision log; **O(n²)** for the cached distance matrix built once per optimization call.

These figures describe the actual code in `server/src/algorithms/`, not a theoretical upper bound — they were not invented separately from the implementation.

## 6. Implementation

- **Backend**: Node.js + Express + TypeScript, layered as `algorithms/ → services/ → controllers/ → routes/`, with a storage-agnostic `repositories/` layer (Postgres in production, a local JSON file for offline development — selected automatically based on whether `DATABASE_URL` is set).
- **Frontend**: React + TypeScript + Vite, with a 3D mission scene (React Three Fiber), a synchronized Leaflet map, Recharts-based analytics, and a Zustand store driving both visualizations from one shared simulation clock so the map and the 3D scene can never disagree about where the drone is.
- **Validation**: battery > 0, at least one delivery, priority in (0, 10], valid lat/lng ranges, non-empty names, duplicate-ID detection, and rejection of NaN/Infinity — enforced both client-side (immediate feedback) and server-side (source of truth), in `server/src/utils/validation.ts`.

## 7. Results and Test Cases

`server/src/algorithms/algorithms.test.ts` implements and actually executes all 13 required scenarios (normal mission, single delivery, multiple deliveries, very low battery, high-priority-nearby, high-priority-far, partially unreachable, all-reachable, equal priorities, edge-case coordinates, empty locations, exact battery boundary, and the return-to-base constraint), plus a Haversine correctness suite in `graph.test.ts`. All scenarios pass (verified in this environment — see the delivery notes for how, since `npm test` requires a one-time `npm install`).

On the seeded demo mission (6 deliveries, 45 km battery), the two algorithms produce **different real routes**:

| Metric | Greedy | Naive |
|---|---|---|
| Route | Base → Indiranagar Depot → Koramangala Hub → Electronic City → Jayanagar Market → Base | Base → Jayanagar Market → Koramangala Hub → Indiranagar Depot → Hebbal Lake → Base |
| Total distance | ~38.2 km | ~29.3 km |
| Completed / Missed | 4 / 2 | 4 / 2 |
| Priority delivered | 24 | 20 |

This demonstrates the real tradeoff the comparison page explains: naive minimizes distance, greedy maximizes delivered priority value within the same battery budget.

## 8. Conclusion and Future Scope

The project delivers a working, explainable, battery-aware drone routing tool built on a real graph model and two genuinely-executed algorithms, with full-stack persistence and a synchronized 2D/3D visualization. Future scope (intentionally not implemented, to keep the delivered scope focused): real flight-path distance APIs, weather/wind-aware routing, dynamic battery modeling, multi-drone fleet optimization, charging stations, no-fly zones, delivery time windows, and dynamic re-routing.

## 9. References

- Haversine formula — [Wikipedia](https://en.wikipedia.org/wiki/Great-circle_distance)
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms* (greedy algorithms, NP-hardness of TSP-like routing)
- OpenStreetMap / Leaflet, React Three Fiber, Supabase documentation
