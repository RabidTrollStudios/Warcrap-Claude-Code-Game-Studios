# Unit Lifecycle

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Fair Competition (deterministic spawn/death)

## Overview

The Unit Lifecycle system manages the existence of every entity in the game — from
the moment a unit is spawned to the moment it is removed after death. It assigns
unique IDs, initializes stats from Balance Data, tracks ownership, manages the
master unit collection, and handles cleanup when units die.

Every other gameplay system operates on units that this system creates and
maintains. Combat damages them, Economy funds them, Production births them — but
Unit Lifecycle owns the bookkeeping of what exists, who owns it, and when it's
gone. It runs within the Tick Engine's phases: spawning happens during Phase 2
(task completion), death removal happens during Phase 4.

## Player Fantasy

**"Build your army, protect your investments."**

Every unit represents a gold investment and a strategic choice. Losing units
hurts — both economically and in timeout scoring. The lifecycle system makes this
tangible: units exist, they can be counted and tracked, and when they die, they're
gone. Agents must weigh the cost of losing units against the value of aggressive
play.

## Detailed Design

### Core Rules

#### 1. Unit Identity

- Every unit receives a globally unique `UnitNbr` (integer, monotonically
  increasing)
- `NextUnitNbr` counter is shared across both agents — never reused, never reset
  within a round
- Units are stored in a master dictionary keyed by UnitNbr
- Each unit tracks: UnitNbr, UnitType, OwnerAgentNbr, GridPosition, Health,
  IsBuilt, CurrentAction, and all task-specific state

#### 2. Spawn Sources

Units enter the game through three paths:

| Source | When | Initial State |
|--------|------|--------------|
| **Round setup** | Before first tick | Starting units (Base, Pawns, Mines) placed at spawn positions. `IsBuilt = true` |
| **Train completion** | Phase 2, when a building's train timer expires | Spawned at first available buildable neighbor of the training building. `IsBuilt = true` |
| **Build command** | Phase 1, when a Build command is validated | Building placed immediately at target position. `IsBuilt = false` (under construction). Pawn walks to site, then build timer counts down |

#### 3. Spawn Initialization

When a unit is created:
1. Assign `UnitNbr = NextUnitNbr++`
2. Set `UnitType`, `OwnerAgentNbr`, `GridPosition` from the spawn source
3. Set `Health = HEALTH[UnitType]` from Balance Data (except Mine, which uses
   config StartingMineGold)
4. Set `Mana = MAX_MANA[UnitType]` (0 for non-Monk units)
5. Set `CurrentAction = IDLE`
6. Set `IsBuilt` based on spawn source (true for trained units and round setup,
   false for build commands)
7. Update Map System: mark footprint cells as not buildable (and not walkable for
   non-top rows of buildings)

#### 4. Death Condition

A unit is dead when `Health <= 0`. Death is checked during **Phase 4 (Remove Dead
Units)** of the tick cycle — not immediately when damage is applied.

#### 5. Death Cleanup

When a dead unit is removed:
1. Remove from master unit dictionary
2. Update Map System: revert all footprint cells to walkable and buildable
3. If the unit was carrying gold (Pawn with held resources from gathering), the
   gold is lost
4. Any other units targeting this unit (attacking, healing, repairing) will
   discover the target is gone on their next Phase 2 advance and retarget or go
   IDLE
5. The UnitNbr is never reused

#### 6. IsBuilt Flag

Buildings placed via Build commands start with `IsBuilt = false`:
- Unbuilt buildings cannot train units
- Unbuilt buildings do not satisfy dependency checks (e.g., an unbuilt Base
  doesn't enable Barracks construction)
- Unbuilt buildings can still be attacked and destroyed
- When the build timer completes, `IsBuilt = true`

#### 7. Unit Queries

The system provides these query interfaces (exposed via IGameState):
- `GetMyUnits(UnitType)` — all owned units of a type
- `GetEnemyUnits(UnitType)` — all enemy units of a type
- `GetAllUnits(UnitType)` — all units of a type (both players + neutral)
- `GetUnit(unitNbr)` — single unit by ID, returns null if dead/nonexistent

### States and Transitions

A unit progresses through a simple lifecycle:

```
SPAWNING → ALIVE → DEAD → REMOVED
```

| State | Description | Duration |
|-------|------------|----------|
| **Spawning** | Unit is being created (assign ID, set stats, update grid) | Instantaneous (within one phase) |
| **Alive** | Unit exists in the game, can receive commands and be targeted | Until Health ≤ 0 |
| **Dead** | Health ≤ 0, awaiting removal | Between damage application (Phase 2) and cleanup (Phase 4) of the same tick |
| **Removed** | Unit no longer exists in any data structure | Permanent |

There is no respawn mechanic. Dead units are gone permanently.

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Tick Engine** | Phase 2 triggers spawns (train/build completion); Phase 4 triggers death removal | Lifecycle events within tick phases |
| **Balance Data** | Reads HEALTH, MAX_MANA, UNIT_SIZE at spawn time | Initial stats |
| **Map System** | Updates grid on spawn (claim cells) and death (release cells) | Footprint mutations |
| **Combat** | Applies damage to Health during Phase 2 | Health reduction |
| **Economy** | Funds unit creation (gold deducted by Command Validation) | Gold flow |
| **Production** | Triggers spawn on train/build completion | New unit creation |
| **Healing** | Restores Health during Phase 2 | Health increase |
| **Agent SDK** | Exposes unit queries via IGameState | Unit lists, UnitInfo structs |
| **Match Flow** | Reads alive unit counts for win condition and scoring | Unit census |
| **Sim Parity** | Unit state is included in tick hash | Hash input |

#### Ownership Boundary

Unit Lifecycle owns:
- Master unit dictionary (the authoritative collection of all units)
- UnitNbr assignment (monotonic counter)
- Spawn initialization (stats, grid update)
- Death detection (Health ≤ 0) and cleanup (removal, grid revert)
- Unit query API

Unit Lifecycle does NOT own:
- Why units take damage (Combat system)
- When units are created (Production system decides train/build timing)
- Gold costs (Economy / Command Validation)
- What units do while alive (Tick Engine / individual action systems)

## Formulas

### Spawn Health

```
Health = HEALTH[UnitType]         (for all types except Mine)
Health = StartingMineGold         (for Mine, from match config)
```

### Spawn Mana

```
Mana = MAX_MANA[UnitType]         (100 for Monk, 0 for all others)
```

### Unit ID Assignment

```
UnitNbr = NextUnitNbr
NextUnitNbr = NextUnitNbr + 1
```

Monotonically increasing, never reset within a round. Starting value is
implementation-defined (typically 0 or 1 after round-setup units).

### Train Spawn Position

```
SpawnPosition = first buildable position in GetBuildablePositionsNearUnit(building)
If no buildable position exists: spawn fails, building remains in TRAIN state
```

### Death Check

```
IsDead = (Health <= 0)
```

Applied to all units during Phase 4. No partial death, no death threshold —
strictly ≤ 0.

## Edge Cases

#### 1. No buildable neighbor for trained unit spawn
The building's train timer has completed but all neighbor cells are occupied. The
spawn is deferred — the building continues holding the trained unit until a
neighbor cell becomes available. Gold is NOT refunded (it was spent when the Train
command was processed).

#### 2. Building destroyed while under construction (IsBuilt = false)
The building is removed normally. The pawn constructing it goes IDLE. Gold is not
refunded — it was deducted at command time.

#### 3. Pawn dies while carrying gold
The held gold (stored in MiningTimer) is lost. It is not dropped on the ground or
returned to the player's pool.

#### 4. Multiple units die in the same tick
All units with Health ≤ 0 are removed in Phase 4. Removal order is deterministic
(by UnitNbr). Grid cells are reverted for each removed unit.

#### 5. Unit killed by multiple attackers in the same tick
Both attackers deal damage in Phase 2 (ordered by command sort). The unit's health
may go well below 0. It is removed once in Phase 4 regardless of how negative the
health is.

#### 6. Last building destroyed (win condition check)
Unit Lifecycle removes the building and reverts its grid cells. It does NOT check
win conditions — that is Match Flow's responsibility, reading from the unit census
after Phase 4.

#### 7. UnitNbr overflow
With `int` (32-bit), overflow occurs after ~2 billion units. At 20 ticks/second
with 10 spawns/tick, this would take ~3.4 years of continuous play. Not a
practical concern.

#### 8. GetUnit() called with ID of dead unit
Returns null. Agents must handle null returns — a unit that existed last tick may
be gone this tick.

## Dependencies

### Upstream (hard dependencies)

| System | What It Provides |
|--------|-----------------|
| **Tick Engine** | Phase structure (when spawns and removals happen) |
| **Balance Data** | HEALTH, MAX_MANA, UNIT_SIZE for spawn initialization |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Combat** | Hard | Units to damage |
| **Economy** | Hard | Units to fund and gather with |
| **Production** | Hard | Spawn mechanism for new units |
| **Healing** | Hard | Units to heal |
| **Map System** | Hard | Footprint claims/releases |
| **Match Flow** | Hard | Unit census for win conditions and scoring |
| **Agent SDK** | Hard | Unit queries for agents |
| **Sim Parity** | Hard | Unit state in tick hash |

## Tuning Knobs

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| **Starting units per player** | 1 Base, 2 Pawns, 1 Mine | 1–5 Pawns | Too few pawns = slow start | Too many = rush strategies dominate | Early game pacing |
| **MinesPerPlayer** | 1 (2 total) | 1–4 | Single resource bottleneck, aggressive mine control | Abundant resources, less conflict over economy | Economic pressure, map control value |

These are match config values (owned by Match Flow / SimConfig), not Balance Data
constants. They affect Unit Lifecycle because they determine how many units exist
at round start.

## Acceptance Criteria

#### Identity
- [ ] Every unit has a unique UnitNbr that is never reused within a round
- [ ] NextUnitNbr is identical in SimGame and Unity at every tick

#### Spawning
- [ ] Trained units spawn at a buildable neighbor cell of the training building
- [ ] Built structures start with `IsBuilt = false` and transition to `true` on
  build completion
- [ ] Spawn initialization sets Health, Mana, and grid state correctly for all 11
  unit types
- [ ] Round setup places starting units at spawn positions symmetrically

#### Death
- [ ] All units with Health ≤ 0 are removed during Phase 4, every tick
- [ ] Dead unit grid cells are reverted to walkable and buildable
- [ ] Dead units are no longer returned by any query (GetUnit returns null)
- [ ] Units targeting a dead unit retarget or go IDLE on next advance

#### Parity
- [ ] Unit dictionary contents are identical in SimGame and Unity at every tick
- [ ] Spawn and removal operations produce identical grid state in both engines

#### Performance
- [ ] GetUnit(id) lookup is O(1) (dictionary)
- [ ] GetMyUnits(type) returns in under 0.1ms with 100 units on the field
- [ ] Phase 4 removal of 50 dead units completes in under 1ms
