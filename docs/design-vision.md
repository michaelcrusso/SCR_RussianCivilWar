# Russian Civil War / Bolshevik Revolution — Design Vision

Working title may later become **The Bolshevik Revolution**. Scope: SC:R UMS, ~1917–1923, TrigScript-first. This document locks the preferred design direction from design discussion (Nov 2026); it is not an implementation checklist.

Legacy TrigEdit under `Triggers/` remains migration source only.

---

## 1. High-level fantasy

A multi-front civil war that is both a **battle sim** and a rough **economy / political sim**. Players race to defeat opposing armies **before** unrest, desertion, and the Black Army devour the home front. Food, morale, mobilization quality, and political choices matter as much as winning firefights.

---

## 2. Factions and slots

Exact slot map is TBD, but the structure is:

| Role | Intent |
|------|--------|
| Soviet Army A / B | Two Bolshevik players (split fronts; arbitrary doctrinal split) |
| White Army A / B | e.g. Ukrainian theater vs Caucasus/Urals theater |
| Entente / Poland | Optional evolving slots |
| Black Army | Hostile NPC in **players 1–8** (not P12); hard to pin down; gains settlements from unrest |
| Open human slots | Unfilled → competent TrigScript AI (owned by separate AI workstream) |

### Extended players 9–12

P9–P12 lack normal vision and cannot attack without exotic tricks. **Never** assign Black Army or any fighting faction there.

**Player 12** is reserved as the **shatter / rout** owner: shattered units are given to P12 and are **free kills** unless rallied (e.g. by generals). That vulnerability is intentional pressure against letting armies break wholesale.

---

## 3. Settlements, food, unrest

- **Cities** and **towns/rural settlements** are capturable; each grants bonuses.
- Soviets lean industrial; Whites lean countryside — but **both** want both terrain types.
- **Food** (prefer **minerals**) is consumed continuously by field forces and cities.
- **Gas** can represent munitions / industrial throughput / rail capacity (Soviet edge candidate).
- Players can raise extraction (War Communism–style levers, requisition). Higher extract → more food, faster **unrest / revolt risk**.
- Both factions are ultimately antagonistic to villages (Whites less so, balanced by worse city politics).

### Allied resources (**locked lean:** sync income, local spend)

Two Red (or two White) players are one front, two theaters:

- **Income / manpower tick globally for the faction** — success in either theater (cities, extract) raises **both** players’ pools the same way; losses cut both.
- **Spending is independent** — each player chooses how to train, summon, and burn their own copy of that income. No shared wallet griefing.
- **Balance caution:** taking one city buffs two friendly incomes and nerfs two enemy incomes (2× leverage). Tune city values, soft caps, or diminishing returns so one breakthrough doesn’t snowball the whole war instantly.

Engine note: SC resources are per-player; sync is TrigScript bookkeeping (mirror income events onto both wallets).

Full tile-to-tile supply logistics are optional later. Prefer virtual food economy first; vulnerable courier/rail “supply” units are a stretch goal once FOBs exist.

### Military doctrines (candidate layer)

Per-player (or per-theater) **doctrine picks** that retune army composition, shatter/morale behaviour, and reinforce mix — see design discussion. Prefer a small mutually exclusive or dual-slot doctrine set, not a huge tech tree. Faction politics remain the dual-consent layer; doctrine is how *this* army fights.

---

## 4. Forward operating bases (FOBs)

- Each player has an FOB represented by a **Terran Siege Tank** (siege mode = intentional immobility).
- Disarm weapons; custom controls for reinforce / rally / politics hooks (hotkeys or hijacked buttons first; full buttonsets later).
- Preferred reinforce flow: troops originate from controlled settlements (or training yards) and **march toward the FOB** (geography and ambush matter). Couriers that convert on arrival are optional.
- FOB death → respawn with a penalty (food, morale, and/or time).

---

## 5. Irregular unit recipes (DAT / unit IDs)

### What we believe (sources disagree in wording; playtest settles edge cases)

- **Create-latched (existing units keep old recipe):** max HP, movement speed, graphics/flingy chain, elevation, start direction, dimensions — supported by SCRMaps, classic pre-place/recreate advice, and scmJS’s “most unit-type fields apply to units made after the write.”
- **Live table lookups:** `weapons.dat` damage / cooldown / range (scmJS: a weapon write hits every user at once). Armour is contested (SCRMaps: live; treat as playtest).
- **Ground/air weapon *pointer* on `units.dat`:** CUnit has **no** stored weapon id (EUD Book) — combat resolves via `unitType` → type table, which argues for **live** retargeting of all instances of that type. Mid-battle `SetUnitsDat` weapon swaps are a common Staredit pattern that also implies live behavior. scmJS does not specially document weapon-pointer latching. **Do not design around freezing different weapon pointers on the same unit id** until Remastered playtest proves it.

### Preferred approach for variance + double-click select (**locked**)

Predefine many unit types with nearly identical graphics/sounds/iscript, each permanently assigned a different weapon slot / damage recipe (e.g. “Red Army Rifleman” at damage 11 vs 13). They *look* the same; they are **not** the same type — that is required for mass-select-by-type and for reliable combat variance. Optionally still jitter create-latched max HP within a grade.

Documented in `/home/michael/maps/idiosyncrasies.md` and project agent rules. Do not design around freezing weapon pointers on one id.

---

## 6. Morale, shatter, desertion, generals

### Per-unit state

Primary loop is **walking the unit table** (needed anyway for combat power sums, AI dispatch, shatter rolls). Extra per-unit fields:

- Prefer **engine-native writable fields** when they fit (`rank`, non-caster `energy`, timers, etc.) or carefully chosen CUnit padding via `setMemory` after Remastered validation.
- `Map<Unit, T>` remains available when we need more state than spare fields allow (see §10). Not required for every value.

Shatter threshold can be stored per unit and rolled while iterating.

### Shatter flow (locked preference)

1. Low HP + RNG vs threshold → shatter.
2. **`give()` to Player 12**; issue move toward owner’s FOB (or general rally).
3. P12 units are attackable — **free kills** if the player fails to stabilize.
4. On reach FOB / general reclaim → return to owner, faster heal.
5. Separate RNG: desert → walk to empty settlement and `remove()` (optionally bump unrest).

Global morale (faction or dual Red/White scalars) modulates shatter HP threshold and desert chance. Political tools (propaganda, No-Step-Back, extract policy) push these dials.

**Decimate Battalion** (Bolshevik): selected/nearby units — lower shatter threshold, raise combat power, cut HP. UX may start as officer/FOB ability before true command-card buttons.

---

## 7. Black Army pressure cooker

- Computer faction in **1–8**, partisan / hard to pin down.
- As unrest accumulates, settlements (especially under-garrisoned) flip Black.
- Can be fought, but the clock is the point: win the war before the country dissolves.
- Year/event deck (legacy `Triggers/Events/` stubs) can ratchet unrest and doctrine.

---

## 8. Mobilization, training yards, doctrine

- Prefer **timed mobilization**, not pure instant spawn spam.
- **Training yards:** a visible rear/off-map area where mobilized units appear for inspection; quality rises over time (grades / recipes). Summon marches them to the FOB.
- **Conscription / forced mobilization:** same food cost, worse quality, morale hit, flush yard early.
- Waiting = better regulars as Red professionalization (and brutality/support erosion) advances on the calendar.

### Per-player vs per-theater yards

**Default: per-player (per theater command) yards.**

Reasons given the faction split (two Soviets, two Whites on different fronts):

- Ukraine vs Urals/Caucasus should not share one physical staging pile on a map that cannot hold all of Russia.
- Each commander’s food, unrest, and summon pressure stay local to their front.
- AI and humans both get a clear “my depot” without negotiating yard ownership.

**Shared across allied pair:** faction **doctrine / politics dials** (professionalization, War Communism intensity, purge state, foreign-aid flags) — not the physical yard stacks. Theater yards feed from shared policy modifiers (“this faction’s conscripts are currently trash”) while remaining separately summonable.

---

## 9. Politics and allied coordination

Personal/theater objectives can diverge; **faction-level political acts** should feel coordinated between the two allied players.

### Visualization / interaction (preferred sketch)

Keep a small set of **faction policies** (not a huge tree). Examples: economic tilt, expel Left SRs, Green feelers, foreign aid, No-Step-Back, War Communism intensity.

**Dual-consent board:** a shared “Smolny / Directory” location or FOB menu lists pending policies. Either ally may **propose** (pays a small cost / starts a timer). The partner must **confirm** within the window, or it fails/expires. Both see the pending proposal via text + minimap ping on the board.

Alternatives if dual-consent feels sticky in MP:

- **Political capital:** one rotating “commissar” token (unit or resource) that must be spent to enact faction policy; allies trade the token by agreement.
- **Veto:** one ally enacts; the other may spend unrest/morale to veto once per cooldown.

Theater-only decisions (local extract, local conscription, FOB tactics) need no consent.

Event deck can force joint crises (“Left SR uprising — both must respond”).

---

## 10. What `Map<Unit, MoraleState>` actually is

Not a fixed 1700-index array.

In TrigScript it is a **hash map keyed by live unit handles**:

- When you `map.set(u, state)`, you hang data on that unit identity.
- A unit that dies and is replaced in the same table slot is a **new** key (docs: recycled slot ≠ dead unit’s key).
- Dead keys linger until you `delete` them while iterating.

You still **loop units** for gameplay. The map is only a sidecar when CUnit/script fields are not enough. Cost is dominated by the unit scan, not by `get` vs reading a field on `u`. Prefer fields-on-unit when enough; use `Map` for overflow state (multi-field morale structs, cooldowns, AI tags).

Assault-group definitions and AI dispatch belong to the separate AI workstream; this vision only assumes unit-table iteration and sentinel “needs reinforce” patterns remain available to that work.

---

## 11. Build order (design priority, not a schedule)

1. Settlement control + food drain/income + extract/unrest  
2. FOB tank + march-to-FOB reinforce  
3. Unit grade IDs + create HP jitter  
4. Shatter → P12 + desert + general reclaim  
5. Global/faction morale + basic policies  
6. Training yards (per-player) + conscription vs train  
7. Black Army flips  
8. Allied dual-consent politics UX  
9. Courier logistics / custom buttonsets  
10. AI personalities (other workstream)  

---

## 12. Open items

- Final player-slot roster (Entente/Poland in or out; Black Army slot number)
- Minerals/gas exact semantic split
- Whether armour is live or latched on Remastered (playtest)
- Remastered playtest: does `stats(unit).groundWeapon = …` retarget existing instances?
- Title rename timing (“The Bolshevik Revolution”)
- How much AI workstream owns vs shared combat-power helpers
