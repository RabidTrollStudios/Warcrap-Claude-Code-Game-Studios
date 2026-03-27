# Production

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Strategic Depth (build order decisions)

## Overview

The Production system governs how new units enter the game — both building
construction (pawns placing structures) and unit training (buildings producing
combat units and workers). It manages creation timers, prerequisite checks,
spawn placement, and the two-phase build process (place immediately, then
construct over time).

Production is the bridge between Economy (gold spent) and Unit Lifecycle (units
created). Every unit on the field passed through this system. Build order
decisions — what to build, when, and in what sequence — are the primary expression
of an agent's strategy in the early game.

## Player Fantasy

**"My build order gives me the edge before the first battle."**

The sequence and timing of building placement and unit training defines an agent's
opening strategy. Fast Barracks into Warriors is a rush; double Pawn into Archery
is a boom. The Production system makes these choices meaningful by imposing real
time costs and prerequisite chains.

## Detailed Design

### Core Rules

#### 1. Two Production Paths

| Path | Command | Who | What Happens |
|------|---------|-----|-------------|
| **Build** | `Build(pawnNbr, target, unitType)` | Pawn | Building placed immediately at target (IsBuilt=false). Pawn walks to site. Build timer counts down. Building becomes IsBuilt=true on completion. |
| **Train** | `Train(buildingNbr, unitType)` | Building | Building enters TRAIN state. Creation timer counts down. Unit spawns at buildable neighbor on completion. |

#### 2. Build Process (Structures)

1. **Command accepted** (Phase 1): Gold deducted. Building entity created at
   target position with `IsBuilt = false`. Grid cells claimed (not buildable,
   non-top rows not walkable). Pawn's action set to BUILD.
2. **Walking phase** (Phase 2, multiple ticks): Pawn walks to an adjacent cell
   of the building via `FindPathToUnit`.
3. **Construction phase** (Phase 2, multiple ticks): Build timer counts down by
   `TickDuration` each tick.
4. **Completion**: `IsBuilt = true`. Building can now train units and satisfy
   dependency checks. Pawn goes IDLE.

The building exists and occupies space from the moment the command is accepted —
it can be attacked and destroyed while under construction.

#### 3. Train Process (Units)

1. **Command accepted** (Phase 1): Gold deducted. Building's action set to TRAIN.
   Train timer initialized to `CreationTime[unitType]`.
2. **Countdown phase** (Phase 2, multiple ticks): Train timer decremented by
   `TickDuration` each tick.
3. **Completion**: New unit spawned at first buildable neighbor of the building.
   Building goes IDLE. Trained unit starts with `IsBuilt = true`, full health,
   full mana, action IDLE.

#### 4. Prerequisites

Build commands require all dependencies to be met:
- `DEPENDENCY[unitType]` lists required building types
- Each required type must have at least one **built** instance (`IsBuilt = true`)
  owned by the agent
- Currently, all military buildings require BASE

Train commands require:
- Building is `IsBuilt = true`
- Building is IDLE (not already training)
- `unitType` is in `TRAINS[building.UnitType]`

#### 5. One Train At a Time

A building can only train one unit at a time. If a building is already in TRAIN
state, additional Train commands fail with `BuildingBusy`. There is no production
queue — agents must re-issue Train commands after each completion.

#### 6. Pawn Build Assignment

Only one pawn builds a given structure. If a pawn is commanded to build while
already building something else, the new Build command overrides the old one (the
old building remains under construction at its current progress — another pawn can
be assigned to finish it).

### States and Transitions

#### Build States

```
COMMAND_ACCEPTED → WALKING_TO_SITE → CONSTRUCTING → COMPLETE
```

| State | Duration | What Happens |
|-------|----------|-------------|
| Command Accepted | Instantaneous (Phase 1) | Building placed, gold deducted, pawn assigned |
| Walking to Site | Variable (movement speed) | Pawn paths to building neighbor |
| Constructing | CreationTime[buildingType] ticks | Timer counts down each tick |
| Complete | Instantaneous | IsBuilt = true, pawn goes IDLE |

#### Train States

```
COMMAND_ACCEPTED → COUNTING_DOWN → SPAWN
```

| State | Duration | What Happens |
|-------|----------|-------------|
| Command Accepted | Instantaneous (Phase 1) | Gold deducted, timer initialized |
| Counting Down | CreationTime[unitType] ticks | Timer decremented each tick |
| Spawn | Instantaneous | Unit placed at buildable neighbor, building goes IDLE |

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Tick Engine** | Phase 1: build/train commands. Phase 2: timer advancement, spawn | Production within tick phases |
| **Command Validation** | Validates gold, prerequisites, buildability, building state | Command filtering |
| **Economy** | Gold deducted on command acceptance | Gold spending |
| **Balance Data** | COST, CREATION_TIME_MULTIPLIER, DEPENDENCY, BUILDS, TRAINS | Production parameters |
| **Unit Lifecycle** | Creates new units on completion | Spawn trigger |
| **Map System** | Build claims grid cells; spawn queries buildable neighbors | Spatial operations |
| **Pathfinding** | Pawn walks to build site | Approach pathing |
| **Agent SDK** | Agents query building state, issue Build/Train commands | Agent interface |

#### Ownership Boundary

Production owns:
- Build process (place → walk → construct → complete)
- Train process (command → countdown → spawn)
- Creation timer management
- Prerequisite chain enforcement (delegates check to Command Validation)

Production does NOT own:
- Gold deduction (Economy / Command Validation)
- Unit initialization (Unit Lifecycle)
- Grid cell claiming (Map System)
- What units cost or how long they take (Balance Data)

## Formulas

### Creation Time

```
CreationTime[type] = CREATION_TIME_MULTIPLIER[type] × ScalarCreationTime
ScalarCreationTime = 1 / GameSpeed
```

At GameSpeed 20:

| Unit/Building | Multiplier | Creation Time |
|--------------|-----------|---------------|
| PAWN | 2.0 | 0.10s |
| WARRIOR | 7.0 | 0.35s |
| ARCHER | 4.0 | 0.20s |
| LANCER | 4.0 | 0.20s |
| MONK | 5.0 | 0.25s |
| BASE | 10.0 | 0.50s |
| BARRACKS | 20.0 | 1.00s |
| ARCHERY | 18.0 | 0.90s |
| TOWER | 15.0 | 0.75s |
| MONASTERY | 18.0 | 0.90s |

### Total Build Time (structure)

```
TotalBuildTime = WalkTime + CreationTime[buildingType]
WalkTime = PathLength / MovingSpeed[PAWN]
```

### Production Throughput

```
UnitsPerSecond[building] = 1 / CreationTime[trainedUnit]
GoldPerSecond[building] = COST[trainedUnit] / CreationTime[trainedUnit]
```

At GameSpeed 20:
- Barracks: 1 Warrior / 0.35s = 2.86/s, costs 286 gold/s
- Archery: 1 Archer / 0.20s = 5.0/s, costs 400 gold/s
- Tower: 1 Lancer / 0.20s = 5.0/s, costs 350 gold/s

## Edge Cases

#### 1. Building destroyed while under construction
Building removed by Unit Lifecycle. Pawn goes IDLE. Gold is not refunded — it was
spent at command time. The grid cells are released.

#### 2. Pawn killed while constructing
Construction pauses — no other unit automatically continues. The building remains
at its current build progress (timer value preserved). Another pawn can be
commanded to Repair (which uses the same build mechanic) to finish it, or a new
Build command can assign a pawn to resume.

#### 3. No buildable neighbor for trained unit spawn
The building's train timer has completed but all neighbors are occupied. Spawn is
deferred — building holds the unit until a neighbor cell opens. Building remains
in TRAIN state (cannot accept new Train commands until spawn completes).

#### 4. Build site becomes partially blocked during pawn walking phase
The building was already placed (cells claimed) at command time. The pawn is
walking to an adjacent cell. If the pawn's path is blocked, normal collision
avoidance applies. The building itself is unaffected.

#### 5. Agent issues Train to a building that's still under construction
Rejected by Command Validation with `BuildingNotBuilt`. Buildings must have
`IsBuilt = true` to train.

#### 6. Dependency building destroyed after build command accepted
The build command was already validated and the building placed. Destruction of
the dependency does not cancel in-progress construction. However, the newly built
structure won't satisfy dependencies for OTHER builds until the dependency is
rebuilt.

#### 7. Multiple pawns ordered to build the same structure type at the same time
Each Build command creates a separate building at a separate location. There is no
shared construction — each building is an independent entity with its own timer.

## Dependencies

### Upstream

| System | What It Provides |
|--------|-----------------|
| **Unit Lifecycle** | Pawns to build, buildings to train from |
| **Command Validation** | Validated Build/Train commands with gold checks |
| **Economy** | Gold pool for spending |
| **Balance Data** | COST, CREATION_TIME_MULTIPLIER, DEPENDENCY, BUILDS, TRAINS |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Unit Lifecycle** | Hard | New units spawned on train/build completion |
| **Map System** | Hard | Grid cell claims on build placement |
| **Combat** | Indirect | Produced units become combat participants |
| **Match Flow** | Indirect | Building count affects win conditions |

## Tuning Knobs

All production tuning is done through Balance Data:
- `CREATION_TIME_MULTIPLIER` per unit type — how long to build/train
- `COST` per unit type — gold investment
- `DEPENDENCY` per unit type — prerequisite chain depth

### Key Balance Relationships

| Relationship | Design Intent |
|-------------|---------------|
| Barracks (20.0) vs Archery (18.0) creation time | Barracks slightly slower — warriors are tankier, worth the wait |
| Warrior (7.0) vs Archer (4.0) train time | Archers train faster — compensates for lower HP |
| Lancer (4.0) same as Archer | Fast production matches cavalry's role as responsive counter |
| All military buildings require BASE | Forces base-first opening; losing base is devastating |

## Acceptance Criteria

#### Build Process
- [ ] Building is placed immediately on command acceptance (IsBuilt=false)
- [ ] Building grid cells are claimed at placement time
- [ ] Pawn walks to building then construction timer counts down
- [ ] IsBuilt transitions to true when timer reaches 0
- [ ] Unbuilt buildings cannot train units or satisfy dependencies

#### Train Process
- [ ] Train timer starts at CreationTime[unitType] and counts down by TickDuration
- [ ] Unit spawns at first buildable neighbor on completion
- [ ] Building goes IDLE after spawn
- [ ] Building rejects Train commands while already training (BuildingBusy)

#### Prerequisites
- [ ] Build commands check DEPENDENCY list against agent's built structures
- [ ] Unbuilt structures do not satisfy dependency checks
- [ ] All military buildings require at least one built BASE

#### Parity
- [ ] Creation timers produce identical completion ticks in SimGame and Unity
- [ ] Spawn positions are identical in both engines for the same grid state
- [ ] Build placement claims identical grid cells in both engines

#### Performance
- [ ] Build/Train command processing adds negligible cost to Phase 1
- [ ] Timer advancement is O(n) where n = active builds + active trains
