# TrigScript Strategic AI — Architecture & Handoff

**Status:** design lock for direction (Nov 2026). Not an implementation checklist.  
**Scope:** a generalizable TrigScript AI module for UMS maps (proof-of-concept first), later adapted into SCR_RussianCivilWar / Bolshevik Revolution.  
**Related:** [design-vision.md](design-vision.md) (map fantasy, factions, shatter/P12); agent rules in `.cursor/rules/agent-instructions.mdc`.

This document captures the architecture discussion so work can resume on another machine or after a break without relying on Cursor chat sync (chats are **local to one computer** and do not follow the account).

---

## 1. Goal and constraints

### What we are building

A **map-local strategic AI** written entirely in TrigScript (`program()` / modules), intended to be **importable into other UMS maps** with a thin adapter—not a Melee build-order AI and not a shipped game mod.

Responsibilities (high level):

1. Hierarchical decisions via a coordinator (e.g. economy/affordability gates before combat allocation).
2. Little or no base construction / building management — buildings are mostly **preplaced, invincible, capturable** (`give()` / ownership transfer).
3. Classify regions as self / ally / enemy and prioritize **defend self → assist ally → attack**, with graded priorities.
4. Assemble **assault groups**, track strength, issue group orders via unit loops.
5. Evaluate relative force (global and local); local eval should not be superhuman (throttled / sampled).
6. A **commitment-aware** strategic state machine so groups do not thrash between distant objectives.

Unit micro (kite, focus fire) is **out of scope for v1**.

### Map assumptions that make this tractable

- No expand/build/worker AI.
- Economy is mostly “resources available vs action cost.”
- Geography is region/settlement-centric (RCW: cities/towns), not continuous base placement.
- Computer players stay in **players 1–8**. P9–P12 are not normal combatants; **P12 is shatter/rout ownership** (free kills unless rallied)—see design vision. Black Army / fighting NPCs must not use P12.

### Feasibility verdict

**Doable in TrigScript.** Hard parts are commitment design, world-tally honesty under capture/`give()`/shatter, and keeping a clean map adapter—not the language. Remastered `program()` gives variables, arrays, records, unit objects, orders, and events (`on.died` / `on.killed`). Prefer `stats` / `memory.*` over raw addresses; group tagging may need a static per-unit field other than `rank` (see §5).

---

## 2. Architecture (locked shape)

Three rings. Subsystems must not all mutate one shared “brain” variable.

```text
  ┌─────────────────────────────────────────┐
  │  World model (facts, read-mostly)       │
  │  regions, owners, powers, resources     │
  └─────────────────┬───────────────────────┘
                    │ snapshot
  ┌─────────────────▼───────────────────────┐
  │  Strategy (intent / missions)           │
  │  commit, hysteresis, patch intake       │
  └─────────────────┬───────────────────────┘
                    │ mission intents
  ┌─────────────────▼───────────────────────┐
  │  Execution (groups + orders)            │
  │  membership, strength, order fan-out    │
  └─────────────────────────────────────────┘

  Event path: on.died / capture / spawn
       → update facts
       → (optional) PatchRequest → strategy only
```

| Layer | Owns | Must not |
| --- | --- | --- |
| **World model** | Cached facts: ownership, global/local power, threats, resources | Decide attacks or issue orders |
| **Strategy** | Missions, commitment, scoring among legal candidates | Loop units or spam orders |
| **Execution** | Group membership, strength arrays, `order()` fan-out | Pick a new city / rewrite strategy |
| **Map adapter** | Regions, adjacency, alliances, costs, spawn hooks | Core FSM logic |
| **Coordinator** | Tick order and clocks | Become a junk drawer of combat rules |

**Coordinator order (strategic tick):** sense → affordability gates → defense priority → attack missions → group maintenance → issue orders (execution).

**Tactical sampler** (death-triggered, probabilistic): may only **patch** an active mission (reinforce / hold / request retreat)—never open a full strategic rescore on every death.

Mental model: *a slow strategic planner that buys missions, a group executor, and a throttled sensor that may amend a purchase.*

---

## 3. Clocks and performance budget

Do **not** run one omniscient loop every frame. Stagger work:

| Clock | Cadence (starting point) | Work |
| --- | --- | --- |
| Strategic | ~0.5–2 Hz | Rescore threats, create/abort missions under commitment rules |
| Maintenance | ~0.5–1 Hz | Refresh group strength, refill from idle/overflow, repair orders if needed |
| Event path | sparse | Capture, spawn, death sample → facts and optional `PatchRequest` |

**Cost reality:** arithmetic over tens of regions is cheap. **Unit scans dominate.** TrigScript guidance: scans a few times per second are fine; every-frame full-map scans are a deliberate cost. Avoid `on.order` for bookkeeping (it scans every frame).

**Healthy profile:** a few unit scans per second per AI player; cached region powers; orders on **edges** (mission change, refill, periodic repair)—not every tick. Multiple AI players roughly multiply maintenance unless they share one sense pass.

**Danger zone:** every-frame `allUnits`, per-frame re-order of hundreds of units, recounting every region every tick with overlapping `unitsAt` scans.

**Budget rule:** fix a scan allowance (e.g. ≤1 full friendly scan + ≤1 local/partial scan per AI per second). Keep strategy as pure functions over cached numbers so intelligence tunes without buying CPU.

---

## 4. Spatial model: graph + coordinates

Use **both**; different jobs.

### Adjacency / region graph (strategic truth)

- Hand-authored (or offline-generated) **neighbor list** / weighted edges between regions.
- Answers: legal attack axes, retreat candidates, contiguous fronts, “don’t teleport the army across the map.”
- Edge `cost` can encode travel time, choke danger, rail priority—designer numbers beat live pathfinding in the AI tick.

While a group is committed, **legal retargets** should be graph-constrained (neighbors of current objective or of owned regions), not “any enemy region with a better raw score.”

### X/Y embedding (tactical / tie-break)

- Each region stores a center `(x, y)` (location anchor or building).
- Euclidean (or tile) distance soft-penalizes among **already graph-legal** candidates; sorts reinforce urgency, etc.
- TrigScript: `u.x` / `u.y`, `distanceTo`; do not treat pixel distance as connectivity.

**PoC default:** hand-authored adjacency for a few regions + Euclidean only as a score tie-breaker.

---

## 5. Assault groups and unit tagging

### Group model

- Integer **group id** per unit (single membership for v1; bitwise multi-group packing deferred).
- Convention sketch: `0` = idle / unassigned; `1..G` = assault groups; high sentinel (e.g. `255`) = overflow waiting for a depleted group.
- Maintain `groupStrength[]` / power by **periodic** scan, not every frame.
- Strategy requests power on a region; execution allocates/refills groups and issues orders.

Also plan **garrison floors**: not every unit joins field groups; cities need sticky defenders strategy cannot strip below a minimum without panic rules.

### Tagging field — important constraint

**Do not use `rank` for group IDs.** `rank` may display helpful text (e.g. a red **"Shattered!"** string when morale breaks and the unit retreats). Group membership needs **another** static per-unit field (exact field TBD; validate on Remastered). Bitwise packing of an unused byte is optional later; integer id is enough for v1.

Order fan-out: loop friendlies with that group id and `u.order("attack"|"move"|"patrol", location)` on mission edges / repair ticks.

---

## 6. Missions and commitment (hardest strategic piece)

Strategy state is a **small set of missions**, not one global mood enum continuously re-argmaxed.

Suggested mission fields:

| Field | Role |
| --- | --- |
| `type` | Idle / Attack / Defend / Reinforce / Retreat |
| `region` | Target region id |
| `groupId` | Bound assault group |
| `commitUntilFrame` | Earliest full re-evaluate |
| `reinforceBudget` | Still allowed while committed |
| `abortStreak` | Samples failing abort predicates before exit |

**Enter vs exit use different thresholds** (hysteresis) so the AI does not flap.

While committed:

- Legal responses: wait, reinforce from idle pool, hold.
- Retreat / retarget only if abort conditions fire (e.g. power ratio below floor for N samples, target flipped, group wiped) or commitment expired.

Defense of **self** regions outranks ally assist. Critical self-threat may preempt an attack via an explicit panic tier or after commit expiry—not via continuous re-optimization.

---

## 7. Force evaluation

One shared **combat power** function everywhere (global and local):

- Start simple: headcount × type weight, optionally × current HP fraction.
- Consistency > fidelity for FSM thresholds.

**Global:** cached on strategic/maintenance ticks.  
**Local:** probe location after sampled death (or defend tick): `dying.locate(probe)` then count/score in location.

Death sampling:

- `on.died` / `on.killed` → with probability `p` and **per-group/region cooldown**, run local eval → emit `PatchRequest`.
- Sampling rate alone does not prevent superhuman behavior; **reflexes must not bypass strategy**. Patches are advisory under commitment rules.

UMS default: AI may use full unit knowledge (cheat vision). Fog-limited AI is a later, harder mode.

---

## 8. Data types to instantiate

Prefer indexes, arrays, and records over a type zoo.

### Adapter / static config (map-owned)

- `RegionId` — integer index  
- `Region` — `{ id, location, centerX, centerY, baseWeight }`  
- `Edge` — `{ to, cost }` ; `neighbors: Edge[][]` (adjacency list preferred over dense matrix unless N is tiny)  
- Alliance table / fn: Self | Ally | Enemy | Neutral  
- `unitWeight` by type  
- `actionCosts` for spawn/ability gates  

### World model (facts)

- `regionOwner[]`  
- `regionPowerFriendly[]` / `regionPowerEnemy[]` (cached)  
- `globalPowerSelf` / `globalPowerEnemy`  
- `threatLevel[]`  
- Resource snapshot (minerals/gas or abstract currencies)  

### Execution

- `GroupState` — `{ id, missionId, strength, power, lastOrderFrame, targetRegion }`  
- Per-unit group tag (non-`rank` field)  
- Optional `garrisonFloor[]`  

### Strategy

- `Mission` — see §6  
- Fixed cap on active missions (e.g. 2–4), not an unbounded queue  
- `PatchRequest` — `{ groupId or region, kind: Reinforce | Hold | RequestRetreat, localRatio }`  

### Coordinator

- `lastStrategicFrame`, `lastMaintainFrame`  
- Tunables: intervals, commit duration, enter/exit ratios, sample probability, eval cooldowns  

---

## 9. Module boundary (generalization)

| In the reusable AI package | In the map adapter / map scripts |
| --- | --- |
| Coordinator, mission FSM, groups, force math, patch protocol, debug dumps | Region list, adjacency, weights, alliances, costs, spawn/create hooks, retreat graph extras |
| | Capture notifications: e.g. `ai.reportCapture(region, newOwner)` |
| | City names, siege narrative, RCW-specific events |

If something needs “Kiev” or a special siege trigger, it does **not** belong in the core.

Suggested module sketch (names flexible): `ai/coordinator`, `ai/world`, `ai/missions`, `ai/groups`, `ai/sense`, `ai/patches`, plus `mapAdapter` per map.

---

## 10. Implementation order

**Vertical spine first, then thicken**—not “perfect one layer in isolation,” and not a fully dressed empty skeleton forever.

1. **Spine slice** — world snapshot, one mission, one group, coordinator: sense → assign → orders. Hard-coded regions/adjacency. No economy, death sampler, or ally assist.
2. **Execution truth** — membership, strength refresh, order fan-out; wiped group clears mission.
3. **Commitment** — `commitUntil` + enter/exit hysteresis (no B↔C thrash).
4. **Graph scoring** — adjacency-limited candidates + simple power/value score (+ optional XY tie-break).
5. **Tactical patches** — sampled death local eval → reinforce/hold/request retreat only.
6. **Garrison floors, economy gates, ally priority** — after the spine is reliable.
7. **Extract package + adapter** — then RCW integration (content volume, not new algorithms).

### Proof-of-concept map (before RCW)

Minimal: ~3 regions, human vs one computer, preplaced invincible bases, stub resource/action costs. Prove tagging, orders, commitment, and adapter boundary. Only then port the adapter to RCW city/siege geography.

Debug: mission/group dumps (print or equivalent) are mandatory on the PoC; FSM tuning is blind without them.

---

## 11. Gaps and non-goals

**Plan for (often under-specified at first):**

- Garrison vs field army split  
- Unit value table  
- Order refresh policy  
- Retreat destinations (nearest owned safe region via graph)  
- Mission invalidation on capture  
- `on.created` / spawn → idle pool or group refill  
- Multi-computer: `runAi(player, config)` or per-player programs sharing sense  
- Interaction with shatter → P12 and general rally (map rules own shatter; AI must not assume P12 is a friendly army)

**Explicit non-goals for v1:** Melee-style build AI, strong micro, fog-limited sensing, bitwise multi-group membership, scmJS Mod Editor / shipped `units.dat` packs for AI data.

---

## 12. Difficulty (expectations)

| Scope | Difficulty |
| --- | --- |
| PoC: few regions, 1 AI, groups + attack/defend + commitment | Medium |
| Generalizable module + adapters | Medium–Hard |
| RCW integration (many cities, sieges, allies, events) | Hard (content/edge cases) |
| Fog-limited AI / strong micro | Very hard (different project) |

---

## 13. References

- TrigScript language (units, events, memory): `/home/michael/maps/trigscript-reference/trigscript-language.md`
- Engine quirks: `/home/michael/maps/idiosyncrasies.md`
- Map design vision: [design-vision.md](design-vision.md)
- Legacy triggers (migration source only): `Triggers/`

---

## 14. Handoff checklist (resume here)

When continuing this workstream:

1. Read this doc + [design-vision.md](design-vision.md) § factions / P12 shatter.  
2. Do **not** assume prior Cursor chats exist on the new machine.  
3. Start with PoC map + spine slice (§10), not full RCW wiring.  
4. Pick/validate a **non-`rank`** per-unit group tag on Remastered before baking APIs.  
5. Keep adapter boundary sacred—RCW events call in; core stays map-agnostic.  
6. Prefer engine-native behaviour and staggered scans; measure before every-frame work.  
7. Record durable Remastered discoveries in `idiosyncrasies.md`.

**Open decisions (safe to defer until PoC):** exact unit struct field for group id; exact group cap `G`; numeric commit durations and enter/exit ratios; whether adjacency is fully hand-authored for RCW or semi-generated from city layout.
