# Balance Data

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Strategic Depth, Fair Competition

## Overview

Balance Data is the centralized repository of every gameplay-affecting constant in
Agents of Empires. All unit stats (cost, health, damage, speed, range), capability
flags, production chains, combat multipliers, mana values, and scoring weights live
in a single authoritative source: `AgentSDK/GameConstants.cs`. A derived layer
(`DerivedGameConstants.cs`) computes speed-scaled values from these base constants.

No other file may define or duplicate balance values. The Unity frontend reads from
AgentSDK at runtime — it does not maintain its own copy. This single-source-of-truth
architecture is critical for sim parity: if SimGame and Unity read different values,
match results diverge and the Fair Competition pillar is violated.

Balance Data is not a "system" in the gameplay sense — it has no behavior, no state
machine, no update loop. It is a static data layer that every gameplay system reads
from. Its design document serves as the authoritative reference for what every number
is, why it's that value, and what safe tuning ranges exist.

## Player Fantasy

**"Every strategy is viable. No single build dominates."**

Students should feel that their strategic choices matter — rushing warriors, massing
archers, booming economy, or mixing units should all be paths to victory depending
on what the opponent does. The balance data creates this possibility space. If the
numbers are wrong, one strategy dominates and the competition collapses into
"everyone does the same thing."

This serves the **Strategic Depth** pillar: "No single dominant strategy should
exist — the meta should reward adaptation and scouting." It also serves **Fair
Competition**: both agents read identical constants from the same source, so the
playing field is level by construction.

## Detailed Design

### Core Rules

#### 1. Architecture: Two Layers, One Source of Truth

```
AgentSDK/GameConstants.cs          ← Layer 1: Base constants (static, readonly)
    ↓
AgentSDK/DerivedGameConstants.cs   ← Layer 2: Speed-scaled values (computed from Layer 1 + GameSpeed)
    ↓
SimGame reads directly              ← No copy, no duplication
Unity reads directly                ← No copy, no duplication
```

**Layer 1 — GameConstants** (static, immutable):
- All base values: cost, health, base damage, creation time multipliers, attack
  range, heal range, mining capacity, mana constants
- All capability flags: CAN_MOVE, CAN_BUILD, CAN_TRAIN, CAN_ATTACK, CAN_GATHER,
  CAN_HEAL
- All relationships: DEPENDENCY, BUILDS, TRAINS
- Unit sizes, combat multipliers
- These values NEVER change at runtime

**Layer 2 — DerivedGameConstants** (computed per GameSpeed):
- Speed-scaled values: MovingSpeed, Damage, CreationTime, MiningSpeed, ManaRegen
- Scoring weights (UNIT_VALUE)
- Recomputed whenever GameSpeed changes
- All formulas derive from Layer 1 values × speed scalars

**Eliminated**: `RTS/Constants.cs` must not duplicate any balance dictionaries. It
may hold Unity-specific presentation values (colors, direction vectors) but all
gameplay values must be read from AgentSDK.

**Eliminated**: `GameConstants.MOVEMENT_SPEED` dictionary (legacy). Movement speed
is defined solely by `DerivedGameConstants.SPEED_MULTIPLIER`.

#### 2. Unit Stats

All 11 unit types and their base stats:

| Unit | Cost | Health | Base Damage | Speed Multi | Attack Range | Heal Range | Size | Creation Time Multi |
|------|------|--------|-------------|-------------|-------------|------------|------|---------------------|
| MINE | 0 | config | 0 | 0 | 0 | 0 | 3×3 | 0 |
| PAWN | 50 | 200 | 0 | 1.0 | 0 | 0 | 1×1 | 2.0 |
| WARRIOR | 100 | 1,500 | 50 | 2.1 | 1.0 | 0 | 1×1 | 7.0 |
| ARCHER | 80 | 600 | 38 | 3.0 | 9.0 | 0 | 1×1 | 4.0 |
| LANCER | 70 | 900 | 42 | 3.45 | 2.5 | 0 | 1×1 | 4.0 |
| MONK | 90 | 400 | 0 | 0.85 | 0 | 4.0 | 1×1 | 5.0 |
| BASE | 500 | 8,000 | 0 | 0 | 0 | 0 | 6×4 | 10.0 |
| BARRACKS | 400 | 4,000 | 0 | 0 | 0 | 0 | 3×3 | 20.0 |
| ARCHERY | 350 | 4,000 | 0 | 0 | 0 | 0 | 3×3 | 18.0 |
| TOWER | 300 | 3,000 | 0 | 0 | 0 | 0 | 2×2 | 15.0 |
| MONASTERY | 350 | 3,500 | 0 | 0 | 0 | 0 | 3×3 | 18.0 |

**Note**: Mine health is set at runtime from config (`StartingMineGold`, default
10,000). All other health values are static.

#### 3. Capability Matrix

| Unit | Move | Build | Train | Attack | Gather | Heal |
|------|------|-------|-------|--------|--------|------|
| MINE | — | — | — | — | — | — |
| PAWN | ✓ | ✓ | — | — | ✓ | — |
| WARRIOR | ✓ | — | — | ✓ | — | — |
| ARCHER | ✓ | — | — | ✓ | — | — |
| LANCER | ✓ | — | — | ✓ | — | — |
| MONK | ✓ | — | — | — | — | ✓ |
| BASE | — | — | ✓ | — | — | — |
| BARRACKS | — | — | ✓ | — | — | — |
| ARCHERY | — | — | ✓ | — | — | — |
| TOWER | — | — | ✓ | — | — | — |
| MONASTERY | — | — | ✓ | — | — | — |

#### 4. Production Chain

| Building | Trains | Requires |
|----------|--------|----------|
| BASE | PAWN | — (no prerequisite) |
| BARRACKS | WARRIOR | BASE (built) |
| ARCHERY | ARCHER | BASE (built) |
| TOWER | LANCER | BASE (built) |
| MONASTERY | MONK | BASE (built) |

Pawns can build: BASE, BARRACKS, ARCHERY, TOWER, MONASTERY.

#### 5. Combat Multipliers

| Attacker ↓ / Defender → | Warrior | Archer | Lancer | All Others |
|--------------------------|---------|--------|--------|------------|
| **Warrior** | 1.0 | **1.25** | 0.75 | 1.0 |
| **Archer** | 0.75 | 1.0 | **1.25** | 1.0 |
| **Lancer** | **1.25** | 0.75 | 1.0 | 1.0 |

#### 6. Mana & Healing Constants

| Constant | Value | Description |
|----------|-------|-------------|
| MAX_MANA | 100 | Monk's mana pool |
| MANA_COST | 10 | Cost per heal (10% of pool) |
| BASE_MANA_REGEN | 100/60 ≈ 1.667/s | Base regen rate (scaled by GameSpeed) |
| HEAL_AMOUNT | 100 | Flat HP restored per heal |
| Eligibility | Health ≤ MaxHealth - HEAL_AMOUNT | Target must be missing at least HEAL_AMOUNT HP |

#### 7. Economy Constants

| Constant | Value | Description |
|----------|-------|-------------|
| MINING_CAPACITY (Pawn) | 50 | Gold carried per trip |
| SCALAR_COST | 50 | Base unit of cost (used in formulas) |
| SCALAR_MINING_CAPACITY | 10 | Base mining scalar |

#### 8. Scoring (Timeout Win Condition)

Formula: `UNIT_VALUE = ceil(COST / SCORING_SCALAR)`

`SCORING_SCALAR = 20` (tuning knob)

| Unit | Cost | Value (points) |
|------|------|---------------|
| MINE | 0 | 0 |
| PAWN | 50 | 3 |
| WARRIOR | 100 | 5 |
| ARCHER | 80 | 4 |
| LANCER | 70 | 4 |
| MONK | 90 | 5 |
| BASE | 500 | 25 |
| BARRACKS | 400 | 20 |
| ARCHERY | 350 | 18 |
| TOWER | 300 | 15 |
| MONASTERY | 350 | 18 |

Scoring is proportional to gold invested — no unit type yields disproportionate
points. Buildings are high-value targets. Tiebreaker: agent with more gold wins.

**Status**: Pending validation through tournament testing.

### States and Transitions

Balance Data has no runtime state. It is a static data layer with two modes:

| Mode | When | What Happens |
|------|------|-------------|
| **Initialized** | Game launch | GameConstants loaded (compile-time static) |
| **Derived** | GameSpeed set or changed | DerivedGameConstants recomputed from GameConstants × GameSpeed |

There is no state machine. DerivedGameConstants is recomputed atomically whenever
GameSpeed changes — there is no partial or transitional state.

### Interactions with Other Systems

Balance Data is read-only. Every gameplay system reads from it; nothing writes to
it at runtime.

#### Consumers (systems that read Balance Data)

| System | What It Reads | Why |
|--------|-------------|-----|
| **Tick Engine** | DerivedGameConstants (all speed-scaled values) | Drives task advancement timers, damage, movement |
| **Command Validation** | COST, DEPENDENCY, CAN_*, BUILDS, TRAINS | Validates agent commands against rules |
| **Unit Lifecycle** | HEALTH, UNIT_SIZE | Sets initial HP and grid footprint on spawn |
| **Combat** | BASE_DAMAGE, ATTACK_RANGE, DamageMultiplier, UNIT_SIZE | Calculates DPS and effective range |
| **Economy** | COST, MINING_CAPACITY, MiningSpeed | Deducts gold, controls gather rate |
| **Production** | CREATION_TIME_MULTIPLIER, DEPENDENCY, TRAINS | Controls build/train timers and prerequisites |
| **Healing** | HEAL_FRACTION, HEAL_THRESHOLD, MANA_COST, MAX_MANA, HEAL_RANGE | Controls heal amounts, eligibility, mana costs |
| **Agent SDK** | All constants exposed via IGameState | Agents can query costs, ranges, capabilities |
| **Match Flow** | UNIT_VALUE | Computes timeout scoring |
| **Opponent Library** | All constants | AI strategies reference costs, damage, ranges for decision-making |

#### Ownership Boundary

Balance Data owns:
- All numeric gameplay constants (costs, health, damage, speeds, ranges, timers)
- All capability flags (what units can/can't do)
- All relationship data (dependencies, builds, trains)
- Combat multiplier table
- Scoring formula and weights
- The GameSpeed → derived constants computation

Balance Data does NOT own:
- Config values set per-match (StartingGold, StartingMineGold, MapSize) — owned
  by Match Flow / SimConfig
- How constants are used in gameplay logic — owned by individual gameplay systems
- GameSpeed itself — owned by Tick Engine / spectator controls

## Formulas

### Derived Constants (GameSpeed-scaled)

All derived values are computed from base constants × a speed scalar:

```
ScalarCreationTime = GameSpeed > 0 ? 1f / GameSpeed : float.PositiveInfinity
ScalarDamage       = GameSpeed
ScalarMovingSpeed  = GameSpeed × 0.05f (BASE_MOVE_SPEED)
ScalarMiningSpeed  = GameSpeed
ManaRegen          = (100f / 60f) × GameSpeed
MiningSpeed        = GameSpeed × 20f
```

### Per-Unit Derived Values

```
CreationTime[type]  = CREATION_TIME_MULTIPLIER[type] × ScalarCreationTime
Damage[type]        = BASE_DAMAGE[type] × ScalarDamage
MovingSpeed[type]   = ScalarMovingSpeed × SPEED_MULTIPLIER[type]
Cost[type]          = GameConstants.COST[type]  (not speed-scaled)
```

| Variable | Definition | Unit |
|----------|-----------|------|
| CreationTime | Seconds to train/build a unit | seconds |
| Damage | Damage dealt per second | HP/second |
| MovingSpeed | Grid cells traversed per tick | cells/tick |
| MiningSpeed | Gold extracted per second while mining | gold/second |
| ManaRegen | Mana restored per second | mana/second |

### Example: All Derived Values at GameSpeed 20

| Unit | CreationTime | Damage/s | MovingSpeed (cells/tick) |
|------|-------------|----------|--------------------------|
| PAWN | 0.10s | 0 | 1.0 |
| WARRIOR | 0.35s | 1,000 | 2.1 |
| ARCHER | 0.20s | 760 | 3.0 |
| LANCER | 0.20s | 840 | 3.45 |
| MONK | 0.25s | 0 | 0.85 |
| BASE | 0.50s | — | — |
| BARRACKS | 1.00s | — | — |
| ARCHERY | 0.90s | — | — |
| TOWER | 0.75s | — | — |
| MONASTERY | 0.90s | — | — |

At GameSpeed 20: MiningSpeed = 400 gold/s, ManaRegen = 33.3/s

### Combat Damage Formula

```
DamagePerTick = BASE_DAMAGE[attacker] × ScalarDamage × DamageMultiplier(attacker, defender) × TickDuration
```

Example: Warrior vs Archer at GameSpeed 20
```
= 50 × 20 × 1.25 × 0.05
= 62.5 HP per tick
= 1,250 DPS
```

### Effective Attack Range

```
EffectiveRange = ATTACK_RANGE[attacker] + max(UNIT_SIZE[target].X, UNIT_SIZE[target].Y) / 2.0f
```

Example: Archer attacking a Base (6×4)
```
= 9.0 + max(6, 4) / 2.0
= 9.0 + 3.0
= 12.0 grid cells
```

### Healing Formula

```
HealAmount = HEAL_AMOUNT (flat 100 HP)
Eligible   = target.Health <= HEALTH[target.UnitType] - HEAL_AMOUNT
ManaCost   = MANA_COST (flat 10)
ActualHeal = min(HEAL_AMOUNT, HEALTH[target.UnitType] - target.Health)
```

Example: Monk healing a Warrior at 1,300 / 1,500 HP
```
Eligible: 1300 ≤ 1500 - 100 = 1400 → YES
ActualHeal: min(100, 1500 - 1300) = 100
New HP: 1,400
Mana cost: 10
```

Example: Monk healing a Pawn at 150 / 200 HP
```
Eligible: 150 ≤ 200 - 100 = 100 → NO (only 50 HP missing)
```

Example: Monk healing a Pawn at 80 / 200 HP
```
Eligible: 80 ≤ 200 - 100 = 100 → YES
ActualHeal: min(100, 200 - 80) = 100
New HP: 180
```

### Repair Formula

```
RepairPerTick = 1.1 × HEALTH[building.UnitType] / CreationTime[building.UnitType] × TickDuration
```

Repair rate is 110% of the build rate — repairing is slightly faster than building.

### Scoring Formula

```
SCORING_SCALAR = 20
UNIT_VALUE[type] = ceil(COST[type] / SCORING_SCALAR)

AgentScore = Σ (count[type] × UNIT_VALUE[type]) for all owned units
Tiebreaker: agent with more gold wins
```

## Edge Cases

#### 1. Heal amount exceeds missing HP
`ActualHeal = min(HEAL_AMOUNT, MaxHealth - CurrentHealth)`. Health is capped at
max — never overheals.

#### 2. GameSpeed set to 0
`ScalarCreationTime = float.PositiveInfinity` — nothing can be created. All other
scalars become 0 — no movement, no damage, no mining. The game effectively freezes.
This is guarded by the GameSpeed range (minimum 1).

#### 3. Unit cost is 0 (Mine)
`UNIT_VALUE = ceil(0 / 20) = 0`. Mines contribute nothing to timeout scoring. This
is correct — mines are neutral resources, not player assets.

#### 4. New unit type added
Every dictionary in GameConstants must have an entry for the new type. Missing
entries cause runtime KeyNotFoundException. A compile-time or startup validation
should verify all dictionaries cover all UnitType enum values.

#### 5. Balance values changed between tournament rounds
Balance Data is static per build. If constants change between compiling opponent
DLLs and running the tournament, agents may behave unexpectedly. The AgentSDK
assembly version should be checked at DLL load time to ensure agents were compiled
against the current constants.

#### 6. DerivedGameConstants computed with different GameSpeed in SimGame vs Unity
Parity breaks immediately. Both engines must read GameSpeed from the same source
(SimConfig or equivalent) and compute derived constants identically. This is
verified by the Tick Engine's state hash.

#### 7. Float rounding in derived calculations
All derived values use `float` (32-bit). Division (`1f / GameSpeed`) may produce
slightly different results on different hardware. Both SimGame and Unity run on the
same machine in practice, but for cross-machine tournament replay, derived constants
should be computed once and shared, not computed independently.

#### 8. SCORING_SCALAR changed after tournament starts
Scoring formula changes mid-tournament invalidate earlier match results.
SCORING_SCALAR should be locked for the duration of a tournament and recorded in
tournament metadata.

## Dependencies

### Upstream (hard dependencies)

None. Balance Data is a foundation layer with zero dependencies.

### Downstream (systems that depend on Balance Data)

| System | Dependency Type | What It Reads |
|--------|----------------|---------------|
| **Tick Engine** | Hard | All derived speed-scaled constants |
| **Command Validation** | Hard | COST, DEPENDENCY, CAN_*, BUILDS, TRAINS |
| **Unit Lifecycle** | Hard | HEALTH, UNIT_SIZE |
| **Combat** | Hard | BASE_DAMAGE, ATTACK_RANGE, DamageMultiplier, UNIT_SIZE |
| **Economy** | Hard | COST, MINING_CAPACITY, MiningSpeed |
| **Production** | Hard | CREATION_TIME_MULTIPLIER, DEPENDENCY, TRAINS |
| **Healing** | Hard | HEAL_AMOUNT, MANA_COST, MAX_MANA, HEAL_RANGE, BASE_MANA_REGEN |
| **Agent SDK** | Hard | All constants (exposed to agents via IGameState) |
| **Match Flow** | Hard | UNIT_VALUE (scoring) |
| **Opponent Library** | Hard | All constants (AI decision-making) |
| **Sim Parity** | Hard | Identical constants required in both engines |

### Bidirectional Notes

- **Match Flow → Balance Data**: Match Flow reads UNIT_VALUE for scoring. Match
  Flow also owns per-match config (StartingGold, StartingMineGold) which sets
  Mine health at runtime. This is the one case where a runtime config value
  overrides a Balance Data default — it should be formalized as a config
  injection, not a mutation of GameConstants.

## Tuning Knobs

#### Unit Stats (Layer 1 — GameConstants)

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| COST[unit] | 50–500 | 10–2000 | Units spammed freely, no economic decisions | Unaffordable, game stalls | Economy pacing, build order depth |
| HEALTH[unit] | 200–8000 | 50–20000 | Units die instantly, no tactical play | Battles drag forever, timeout-heavy | Combat duration, healing value |
| BASE_DAMAGE[unit] | 38–50 | 10–200 | Battles take too long | Units die in one tick, no counterplay | Time-to-kill, R-P-S relevance |
| SPEED_MULTIPLIER[unit] | 0.85–3.45 | 0.5–10.0 | Units crawl, boring to watch | Units teleport, impossible to react | Map control, engagement timing |
| ATTACK_RANGE[unit] | 1.0–9.0 | 0.5–15.0 | All combat is melee, archers pointless | Archers dominate, melee can't close | Kiting viability, formation value |
| CREATION_TIME_MULTI[unit] | 2–20 | 1–60 | Instant armies, no scouting window | Too slow to recover from losses | Rush timing, comeback potential |

#### Combat Multipliers

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| R-P-S multiplier (strong) | 1.25 | 1.1–1.5 | Triangle irrelevant, unit choice doesn't matter | Hard counters feel unfair, scouting mandatory | Counter-play depth, meta diversity |
| R-P-S multiplier (weak) | 0.75 | 0.5–0.9 | (Inverse of strong) | (Inverse of strong) | (Same as above) |

#### Healing

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| HEAL_AMOUNT | 100 | 25–500 | Monks useless, never worth building | Monks make armies unkillable | Monk value, attrition rate |
| MANA_COST | 10 | 5–50 | Infinite healing, monks overpowered | One heal then empty, monks useless | Sustain duration, monk engagement time |
| MAX_MANA | 100 | 25–500 | Very few heals per pool | Monks never run dry | Burst healing capacity |
| BASE_MANA_REGEN | 1.667 | 0.5–5.0 | Long downtime between heal bursts | Effectively infinite mana | Sustained vs burst healing |

#### Economy

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| MINING_CAPACITY | 50 | 10–200 | Many trips, slow economy | Fast economy, short games | Game length, worker value |
| SCORING_SCALAR | 20 | 5–100 | Large score differences, buildings dominate | All units worth ~1, scoring granularity lost | Timeout outcome sensitivity |

#### Knob Interactions

- **COST × HEALTH**: The "gold per HP" ratio determines cost-efficiency. If one
  unit has drastically better gold/HP, it dominates defensive play.
- **BASE_DAMAGE × R-P-S multiplier**: Combined, these determine time-to-kill. Both
  must be tuned together — increasing damage by 20% has the same effect as
  increasing the multiplier from 1.25 to 1.50.
- **HEAL_AMOUNT × MANA_COST × BASE_MANA_REGEN**: These three together determine
  monk sustain. Changing one without the others shifts the monk from useless to
  overpowered.
- **CREATION_TIME × COST**: Together these determine "recovery speed" — how fast an
  army can be rebuilt after a loss. If both are low, the game becomes
  attrition-proof.

## Acceptance Criteria

#### Single Source of Truth
- [ ] All gameplay constants are defined in `AgentSDK/GameConstants.cs` — no other
  file defines balance values
- [ ] `RTS/Constants.cs` reads from AgentSDK at runtime — contains zero hardcoded
  balance values
- [ ] `GameConstants.MOVEMENT_SPEED` dictionary is removed (legacy)
- [ ] Every dictionary in GameConstants has an entry for every value in the UnitType
  enum (validated at startup or compile time)

#### Derived Constants
- [ ] `DerivedGameConstants.Compute(gameSpeed)` produces identical results in
  SimGame and Unity for the same GameSpeed
- [ ] Changing GameSpeed recomputes all derived values atomically — no partial state
- [ ] At GameSpeed 0, the game does not crash (graceful handling via
  PositiveInfinity guard)

#### Combat Balance
- [ ] R-P-S triangle is verified: Warrior beats Archer, Archer beats Lancer,
  Lancer beats Warrior (1v1 same-cost investment)
- [ ] No single unit type wins >60% of matches across all opponent strategies in
  automated testing
- [ ] Mixed-army strategies are viable against single-unit strategies

#### Healing Balance
- [ ] HEAL_AMOUNT (100) heals all mobile unit types (every mobile unit has >100
  max HP)
- [ ] Eligibility check prevents healing units missing less than HEAL_AMOUNT HP
- [ ] A single Monk cannot keep a Warrior alive indefinitely against one attacker
  (heal rate < damage rate for any single combat unit)

#### Scoring
- [ ] `UNIT_VALUE = ceil(COST / SCORING_SCALAR)` holds for all unit types
- [ ] No unit type has a value/cost ratio more than 2x any other unit type's ratio
- [ ] Timeout scoring produces decisive results (not always ties) in test matches
- [ ] Tiebreaker (gold) resolves remaining ties

#### Performance
- [ ] Reading any GameConstants dictionary is O(1) (dictionary lookup)
- [ ] `DerivedGameConstants.Compute()` completes in under 1ms
