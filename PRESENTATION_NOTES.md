# Presentation Notes — Drone Delivery Mission Planner

## 30-second explanation

"We built a drone delivery planner that models delivery locations as a graph and routes the drone using a greedy algorithm that balances delivery priority against battery range — picking the highest priority-per-kilometer delivery it can still reach *and return from*. We compare it live against a naive nearest-neighbor baseline so you can see the actual tradeoff, visualized in a synced 3D simulation and real map."

## 1-minute explanation

"The core problem is: given a drone with limited battery and a set of deliveries with different priorities, in what order should it visit them? We represent the base and every delivery as a vertex in a graph, with real geographic distance (Haversine formula) as edge weights. Our greedy algorithm, at every step, only considers deliveries the drone can reach *and still return to base from* — then among those feasible options, picks the one with the best priority-to-distance ratio. We also run a naive nearest-neighbor algorithm on the same mission as a baseline, so the comparison page shows real, computed numbers — not a canned example. Every decision the greedy algorithm makes is logged and shown to the user: which candidates were considered, their scores, and why one was chosen. The whole thing is visualized with a synchronized 3D drone simulation and a real Leaflet map, backed by a Node/Express API and PostgreSQL persistence."

## 3-minute presentation outline

1. **Problem** — drones have finite battery; not all deliveries are equally valuable; naive routing (just "go to the nearest place") can waste battery on low-value stops while missing high-value ones.
2. **Why a graph?** — distance is a pairwise relationship; a graph is the natural structure, and it lets us compute exact feasibility (round-trip distance) before ever committing to a delivery.
3. **Why greedy?** — the real optimal version of this problem is NP-hard (related to the Traveling Salesman Problem with capacity constraints) — infeasible to solve exactly at interactive speed for anything but tiny inputs. Greedy gives a fast, deterministic, *explainable* answer: O(n²) worst case, and every choice can be shown to a user or auditor.
4. **Why priority/distance?** — it's the simplest meaningful way to compare "is this delivery worth the detour?" — a high-priority delivery that's far away can still beat a low-priority delivery that's close, and the formula makes that tradeoff explicit and tunable.
5. **Why the battery constraint?** — a drone that can't get home is a lost drone. We enforce round-trip feasibility (out-and-back distance ≤ remaining battery) before a delivery is ever considered, not after.
6. **How Haversine works** — treats the Earth as a sphere and computes the great-circle (shortest surface-path) distance between two lat/lng points from their angular difference — accurate to within ~0.5% for most practical distances, and O(1) to compute.
7. **Why compare with naive?** — without a baseline, "our algorithm works" is just an assertion. Running a real nearest-neighbor algorithm on the *same* mission data proves the priority-aware approach actually changes outcomes (more priority delivered), not just routes differently for no reason.
8. **Complexity** — O(n²) graph construction, O(1) per-pair distance, O(n²) worst-case for both algorithms — all measured against the actual code, documented in PROJECT_DOCUMENTATION.md.
9. **Real-world applications** — medical supply drones, disaster relief logistics, last-mile delivery in low-infrastructure areas, agricultural monitoring routes.
10. **Limitations** — heuristic (not globally optimal), straight-line distance (not flight-restricted airspace or wind), single drone/single base.
11. **Future scope** — multi-drone fleets, no-fly zones, weather-aware routing, delivery time windows, dynamic re-routing, charging stations.

## Likely judge questions & viva answers

**Q: Is the greedy algorithm always optimal?**
A: No — it's a heuristic. It makes the locally best choice at each step without backtracking, so it can miss a global optimum that a full search (NP-hard for this class of problem) would find. We say this explicitly in the UI and docs rather than overselling it.

**Q: Why not just use Dijkstra's or A\*?**
A: Those solve shortest-path-between-two-points problems. Our problem is closer to an orienteering / prize-collecting TSP variant — choosing *which subset and order* of multiple destinations to visit under a budget constraint — which those algorithms don't directly address.

**Q: What happens if two candidates have the same score?**
A: JavaScript's comparison is strict (`>`), so the first one encountered in iteration order keeps the lead; ties aren't specially broken. This is a known, documented simplification.

**Q: How do you know the naive and greedy results aren't faked?**
A: Both algorithms are implemented as pure, independently testable functions (`server/src/algorithms/greedy.ts`, `naive.ts`) with their own automated test suite (13 required scenarios, all passing), and the API actually executes both on every `/optimize` call — there's no shortcut or cached "expected" answer.

**Q: What's the space/time complexity in the worst case?**
A: O(n²) time and O(n²) space for the full pipeline (graph construction dominates), where n is the number of delivery locations.

**Q: Why PostgreSQL / Supabase?**
A: Relational structure fits the mission → deliveries → results → route-steps hierarchy naturally, with foreign keys enforcing referential integrity, and Supabase gives us a free, managed Postgres instance with zero infrastructure to run ourselves.

**Q: Could this scale to hundreds of deliveries?**
A: The O(n²) algorithms would still run (sub-second for n in the low hundreds), but the real-world modeling (straight-line distance, single drone) would need the "future scope" improvements — multi-drone assignment, more accurate distance data — to stay realistic at that scale.
