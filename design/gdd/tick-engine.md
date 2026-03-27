# Tick Engine

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Fair Competition (determinism)

## Overview

The Tick Engine is the deterministic simulation loop that drives all game logic in
Agents of Empires. It advances the game state in fixed 0.05-second ticks (20 Hz),
processing agent commands and advancing unit tasks in a strict, repeatable order.
The engine exists in two identical implementations — SimGame (headless C#) and Unity
(GameManager) — and its determinism guarantee is the foundation of fair competition:
given identical inputs, both implementations must produce identical state hashes at
every tick.

No player interacts with the Tick Engine directly. It is invisible infrastructure —
the "physics of the world" that agents operate within. Without it, no other system
can function, and without its determinism, no match result can be trusted.

## Player Fantasy

**"The rules are the same for everyone."**

This is infrastructure, not a player-facing system. The fantasy it serves is indirect:
students trust that their agent's success or failure is determined by their code, not
by engine quirks. Spectators trust that what they see matches what happened. Tournament
results are legitimate because the simulation is reproducible.

The Tick Engine serves the **Fair Competition** pillar: "Every match outcome must be
explainable by the agents' decisions, never by engine quirks or timing artifacts."

## Detailed Design

### Core Rules

#### 1. Tick Cycle

Each tick executes the following phases in strict order. No phase may interleave
with another.

| Phase | Action | Description |
|-------|--------|-------------|
| 0 | Increment tick counter | `CurrentTick++` |
| 1 | **Process Commands** | Validate and execute all queued agent commands in deterministic sort order |
| 2 | **Advance Tasks** | Progress all in-flight unit tasks (movement, training, building, gathering, attacking, repairing, healing) |
| 3 | **Regenerate Mana** | All units with a mana pool regenerate toward their maximum |
| 4 | **Remove Dead Units** | Units with `Health <= 0` are removed from the game state |
| 5 | **Agent Update** | Each agent's `Update(IGameState, IAgentActions)` is called; agents observe post-advance state and queue commands for the NEXT tick |

#### 2. Tick Timing

- **Tick duration**: 0.05 seconds (20 ticks per second at game speed 1)
- **Game speed**: Multiplier (1–30) that scales all time-dependent values (movement,
  damage, creation, mining, mana regen) but does NOT change the tick rate itself
- Game speed affects derived constants, not the tick loop frequency

#### 3. Command Processing Rules

- Commands are queued by agents during Phase 5 and processed in the NEXT tick's
  Phase 1
- All commands from both agents are collected into a single list
- The list is sorted deterministically: `(AgentNbr ASC, CommandType ASC, UnitNbr ASC)`
- Commands are processed in sorted order, one at a time
- **One command per unit per tick**: The first command for a given UnitNbr is
  processed; all subsequent commands for that unit in the same tick are silently
  dropped
- Command validation occurs at processing time (Phase 1), not at queue time
  (Phase 5)
- **Immediate feedback**: `IAgentActions` methods return a `CommandResult` with a
  status code and failure reason (e.g., `InvalidTarget`, `InsufficientGold`,
  `PathBlocked`). This reflects queue-time validation only (basic checks like
  "do you own this unit?").
- **Deferred feedback**: `IGameState.GetFailedCommands()` returns a list of commands
  from the previous tick that were accepted at queue time but failed during Phase 1
  processing (e.g., target died between queue and processing, build site became
  occupied). Each entry includes the command, the unit, and the failure reason.
- **Gold refund on failure**: Gold is deducted when a command is validated in Phase 1.
  If the resulting action subsequently fails (e.g., build site blocked, target no
  longer valid), the gold is refunded to the agent's pool before Phase 2 begins.

#### 4. Task Advancement Rules

During Phase 2, each unit's current task is advanced by one tick. The advancement
logic depends on the unit's `CurrentAction`:

| Action | Advancement |
|--------|-------------|
| IDLE | No-op |
| MOVE | Consume accumulated movement along path; handle collisions |
| TRAIN | Countdown creation timer; spawn unit when complete |
| BUILD | Walk to site (movement), then countdown build timer; mark built when complete |
| GATHER | Three-phase cycle: walk to mine → extract gold → walk to base → deposit → repeat |
| ATTACK | If in range: deal damage. If out of range: move closer. If target dead: retarget or idle |
| REPAIR | Walk to building, then restore HP at 110% of build rate |
| HEAL | Walk into range, then restore 50% of target max HP (costs mana) |

#### 5. Determinism Contract

Given identical inputs (same map, same agents, same random seed if applicable),
the Tick Engine MUST produce identical state at every tick. This is verified by a
64-bit hash computed after each tick cycle, covering:

- Global state: CurrentTick, Gold[0], Gold[1], unit count, next unit ID
- Per-unit state (sorted by UnitNbr): position, health, action, all timers, all
  target references, mana

Five independent subsystem hashes are also computed for diagnostic isolation when
parity diverges:

1. **Global** — tick, gold, counts
2. **UnitPositions** — identity + grid location
3. **UnitHealth** — health + built status
4. **UnitActions** — current action + target references
5. **UnitTimers** — accumulators + progress timers

### States and Transitions

The Tick Engine itself has two state layers: the **match state machine** (managed
by Match Flow, documented here for completeness) and the **tick state** (which
phase of the current tick is executing).

#### Tick Phase State

This is not a traditional state machine — it's a strict sequential pipeline that
runs every tick:

```
TICK_START → PROCESS_COMMANDS → ADVANCE_TASKS → REGEN_MANA → REMOVE_DEAD → AGENT_UPDATE → TICK_END
```

No phase may be skipped. No phase runs concurrently with another. The engine is
always in exactly one phase.

#### Per-Unit Action State

Each unit has a `CurrentAction` that determines how Phase 2 (Advance Tasks)
processes it. Valid transitions are governed by command processing in Phase 1:

| From | To | Trigger |
|------|----|---------|
| Any | IDLE | Task completes, target dies with no retarget, or new command overrides |
| IDLE | MOVE | Move command accepted |
| IDLE | TRAIN | Train command accepted (buildings only) |
| IDLE | BUILD | Build command accepted (pawns only) |
| IDLE | GATHER | Gather command accepted (pawns only) |
| IDLE | ATTACK | Attack command accepted (combat units only) |
| IDLE | REPAIR | Repair command accepted (pawns only) |
| IDLE | HEAL | Heal command accepted (monks only) |
| MOVE | Any action | New command overrides current movement |
| ATTACK | IDLE | Target dies and no nearby enemy found |
| ATTACK | ATTACK | Target dies, retarget to nearest enemy in range |
| GATHER | GATHER | Cycles internally (TO_MINE → MINING → TO_BASE → TO_MINE) |
| Any non-IDLE | Any | New command always overrides current action |

#### Gather Sub-States

The GATHER action has an internal three-phase cycle:

```
TO_MINE → MINING → TO_BASE → TO_MINE → ...
```

| Sub-State | Behavior | Transition |
|-----------|----------|------------|
| TO_MINE | Pawn walks toward assigned mine | Arrives at mine → MINING |
| MINING | Countdown timer, extract gold from mine | Timer expires → TO_BASE |
| TO_BASE | Pawn walks toward assigned base | Arrives at base → deposit gold → TO_MINE |

### Interactions with Other Systems

The Tick Engine is the foundation layer. It does not depend on any other system,
but nearly every system depends on it.

#### Downstream (systems that depend on Tick Engine)

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Command Validation** | Tick Engine calls validator during Phase 1 | Commands in → validated commands + rejections out |
| **Unit Lifecycle** | Phase 2 spawns units (Train/Build completion), Phase 4 removes dead units | Spawn/death events |
| **Combat** | Phase 2 advances ATTACK tasks, applying damage per tick | Damage values from Balance Data, health deducted from target |
| **Economy** | Phase 1 deducts/refunds gold; Phase 2 advances GATHER tasks depositing gold | Gold changes per tick |
| **Production** | Phase 2 advances TRAIN and BUILD timers | Creation time from Balance Data, unit spawn on completion |
| **Healing** | Phase 2 advances HEAL tasks; Phase 3 regenerates mana | Mana and health changes per tick |
| **Pathfinding** | Phase 2 calls pathfinding during movement, attack approach, gather walking | Path requests → cell lists |
| **Match Flow** | Reads CurrentTick for time limit checks; reads unit state for win conditions | Tick count, unit/building alive status |
| **Agent SDK** | Phase 5 calls `IPlanningAgent.Update()`; provides `IGameState` snapshot | Read-only state in, commands out |
| **Sim Parity** | Consumes state hash after each tick for cross-engine comparison | Hash values from Phase 4 post-cleanup state |
| **Match Recording** | Captures command list each tick for replay | Command log per tick |

#### Upstream (systems the Tick Engine depends on)

None. The Tick Engine is a pure foundation system. It calls into other systems
(Balance Data for constants, Pathfinding for routes) but those are service calls,
not structural dependencies — the tick loop runs regardless.

#### Ownership Boundary

The Tick Engine owns:
- The tick loop and phase ordering
- Tick counter and timing
- Command collection, sorting, and dispatch (delegates validation to Command Validation)
- The "advance all units" orchestration (delegates per-action logic to gameplay systems)
- State hashing for parity
- Agent lifecycle calls (InitializeMatch, InitializeRound, Update, Learn)

The Tick Engine does NOT own:
- Game rules (owned by Balance Data, Command Validation, individual gameplay systems)
- Match structure — rounds, win conditions (owned by Match Flow)
- What agents see or can do (owned by Agent SDK)

## Formulas

### Tick Timing

```
TickDuration = 0.05 seconds (constant, not affected by game speed)
ElapsedTime = CurrentTick × TickDuration
```

### Game Speed Scaling

Game speed is a multiplier applied to all time-dependent derived constants, not
to the tick rate.

```
ScalarCreationTime = 1 / GameSpeed
ScalarDamage = GameSpeed
ScalarMovingSpeed = GameSpeed × BASE_MOVE_SPEED    (BASE_MOVE_SPEED = 0.05)
ScalarMiningSpeed = GameSpeed
ManaRegen = BASE_MANA_REGEN × GameSpeed
```

| Variable | Definition | Range |
|----------|-----------|-------|
| GameSpeed | User-configurable simulation speed | 1–30 (integer) |
| BASE_MOVE_SPEED | Movement base constant | 0.05 (fixed) |
| BASE_MANA_REGEN | Mana regen base constant | 100/60 ≈ 1.667 (fixed) |
| TickDuration | Seconds per tick | 0.05 (fixed) |

### Command Sort Key

```
SortKey(command) = (AgentNbr, (int)CommandType, UnitNbr)
Comparison: lexicographic ascending on all three fields
```

### State Hash

64-bit FNV-style hash with seed 17:

```
hash = 17
hash = hash × 31 + CurrentTick
hash = hash × 31 + Gold[0]
hash = hash × 31 + Gold[1]
hash = hash × 31 + Units.Count
hash = hash × 31 + NextUnitNbr

for each unit (sorted by UnitNbr ascending):
    hash = hash × 31 + unit.UnitNbr
    hash = hash × 31 + (int)unit.UnitType
    hash = hash × 31 + unit.OwnerAgentNbr
    hash = hash × 31 + unit.GridPosition.X
    hash = hash × 31 + unit.GridPosition.Y
    hash = hash × 31 + FloatToStableBits(unit.Health)
    hash = hash × 31 + (unit.IsBuilt ? 1 : 0)
    hash = hash × 31 + (int)unit.CurrentAction
    hash = hash × 31 + FloatToStableBits(unit.MoveAccumulator)
    hash = hash × 31 + unit.PathIndex
    hash = hash × 31 + (unit.Path?.Count ?? 0)
    hash = hash × 31 + FloatToStableBits(unit.TrainTimer)
    hash = hash × 31 + (int)unit.TrainTarget
    hash = hash × 31 + FloatToStableBits(unit.BuildTimer)
    hash = hash × 31 + (int)unit.BuildTarget
    hash = hash × 31 + unit.BuildSite.X
    hash = hash × 31 + unit.BuildSite.Y
    hash = hash × 31 + (unit.BuildPlaced ? 1 : 0)
    hash = hash × 31 + unit.GatherMineNbr
    hash = hash × 31 + unit.GatherBaseNbr
    hash = hash × 31 + (int)unit.GatherPhase
    hash = hash × 31 + FloatToStableBits(unit.MiningTimer)
    hash = hash × 31 + unit.AttackTargetNbr
    hash = hash × 31 + unit.RepairBuildingNbr
    hash = hash × 31 + unit.HealTargetNbr
    hash = hash × 31 + FloatToStableBits(unit.Mana)
    hash = hash × 31 + unit.LocalAvoidWaitTicks

FloatToStableBits(f) = BitConverter.SingleToInt32Bits(f)
```

### Movement Accumulation

```
Per tick: unit.MoveAccumulator += MovingSpeed[unit.UnitType]
Per cell consumed: unit.MoveAccumulator -= CellCost

CellCost:
  Cardinal (N/S/E/W): 1.0f
  Diagonal (NE/SE/SW/NW): 1.41421356f (fixed constant, not computed at runtime)

MovingSpeed[type] = ScalarMovingSpeed × SPEED_MULTIPLIER[type]
```

**Example at GameSpeed 20:**
- Warrior: 20 × 0.05 × 2.1 = 2.1 cells/tick → crosses ~2 cardinal cells per tick
- Archer: 20 × 0.05 × 3.0 = 3.0 cells/tick
- Pawn: 20 × 0.05 × 1.0 = 1.0 cells/tick

## Edge Cases

#### 1. Two agents issue commands to the same enemy target in the same tick
Both commands are processed. The deterministic sort order determines which agent's
command executes first. If the first command kills the target, the second command
fails at validation (target dead) and is added to that agent's failed command list.
No gold refund needed since Attack commands don't cost gold.

#### 2. Agent issues command to a unit that dies in the same tick
Commands are processed in Phase 1 BEFORE dead units are removed in Phase 4.
However, a unit could have been killed by another command earlier in Phase 1
(since commands process sequentially). If the unit is dead when its command is
reached, the command fails and is reported via `GetFailedCommands()`.

#### 3. Gold deducted for Build/Train but action fails before Phase 2
Gold is refunded before Phase 2 begins. The failed command is reported via
`GetFailedCommands()` with the reason (e.g., `BuildSiteOccupied`, `SpawnBlocked`).

#### 4. Multiple commands issued for the same unit in one tick
Only the first command (by sort order) is processed. All others are silently
dropped. They do NOT appear in `GetFailedCommands()` — this is an expected agent
behavior, not a failure.

#### 5. Agent's Update() throws an exception
The agent's Update call is wrapped in a try/catch. If it throws, the agent issues
zero commands for that tick. The game continues — the other agent's commands still
process normally. The exception is logged. The agent is NOT disqualified (a
single-tick failure should not end a match).

#### 6. Agent's Update() takes too long
**[DESIGN DECISION NEEDED]**: Currently no timeout on agent Update calls. For
tournament play, a per-tick time budget should be enforced (e.g., 50ms). If
exceeded, the agent's commands for that tick are discarded. This interacts with
Agent Security and should be specified there.

#### 7. Game speed changed mid-tick
Game speed changes take effect at the START of the next tick, never mid-tick. All
derived constants are recomputed from the new speed before Phase 1.

#### 8. Float precision divergence between SimGame and Unity
All float comparisons use `BitConverter.SingleToInt32Bits()` for hashing, ensuring
bitwise comparison. Both implementations must use `float` (32-bit single precision),
never `double`, for all game state values. The diagonal movement cost is a fixed
constant (`1.41421356f`), not computed at runtime.

#### 9. Unit path becomes invalid mid-movement (building placed on path)
During Phase 2 movement, if the next cell in the path is now unwalkable (blocked
by terrain or a newly placed building), the unit triggers an immediate A* repath
without the avoid-units flag. If no path exists, the unit goes IDLE.

#### 10. Mine depletes mid-gathering
If a mine's remaining gold is less than the pawn's mining capacity, the pawn
extracts only what's left. If the mine reaches 0 health, it is removed in Phase 4.
Pawns currently walking TO_MINE for a depleted mine will fail to find it and go
IDLE.

#### 11. All bases destroyed while pawns are gathering
Pawns in the TO_BASE gather phase hold their gold (stored in MiningTimer) until a
new base is built. While holding gold, pawns can accept and execute other commands
(Move, Build, Repair). Once a friendly base becomes available, the pawn completes
its current task, then automatically walks to the base, deposits the held gold,
and goes IDLE. Gold is never lost due to base destruction — it persists on the
pawn until deposited.

#### 12. Zero units alive for one agent
The tick continues normally. The agent's Update is still called (it can observe but
can't act). Match Flow determines if this triggers a win condition — the Tick Engine
does not check.

## Dependencies

### Upstream (hard dependencies)

None. The Tick Engine is the foundation layer with zero structural dependencies.

### Service Calls (soft dependencies)

The Tick Engine calls into these systems during execution, but can run its loop
without them:

| System | Call | Phase | Nature |
|--------|------|-------|--------|
| **Balance Data** | Read game constants (speeds, damage, timers) | 1, 2, 3 | Read-only lookup |
| **Command Validation** | Validate each command before execution | 1 | Delegate call |
| **Pathfinding** | Compute routes during movement/approach | 2 | Service call |

### Downstream (systems that depend on Tick Engine)

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Command Validation** | Hard | Called during Phase 1 |
| **Unit Lifecycle** | Hard | Phase 2 spawns, Phase 4 removes |
| **Combat** | Hard | Phase 2 advances attack tasks |
| **Economy** | Hard | Phase 1 gold changes, Phase 2 gathering |
| **Production** | Hard | Phase 2 train/build timers |
| **Healing** | Hard | Phase 2 heal tasks, Phase 3 mana regen |
| **Match Flow** | Hard | Reads tick counter and unit state |
| **Agent SDK** | Hard | Phase 5 agent lifecycle calls |
| **Sim Parity** | Hard | Consumes state hash after each tick |
| **Match Recording** | Soft | Captures command log per tick (optional) |
| **Pathfinding** | Hard | Called during Phase 2 movement |

### Bidirectional Notes

- **Balance Data ↔ Tick Engine**: Tick Engine reads constants; Balance Data does
  not reference Tick Engine. One-directional (Balance Data is upstream service).
- **Agent SDK ↔ Tick Engine**: Tick Engine calls agent lifecycle methods; Agent SDK
  defines the interfaces agents implement. Tick Engine depends on the interface
  definition, Agent SDK depends on being called at the right time.

## Tuning Knobs

| Knob | Current Value | Safe Range | Too Low | Too High | Affects |
|------|-------------|------------|---------|----------|---------|
| **TickDuration** | 0.05s (20 Hz) | 0.01–0.1s | CPU-heavy, more ticks per second than needed | Jerky movement, imprecise timing, agents miss events | All time-dependent systems; movement smoothness; agent reaction time |
| **GameSpeed** | 20 | 1–30 | Extremely slow matches (boring to watch) | Units teleport, actions resolve instantly, hard to spectate | All derived constants: movement, damage, creation time, mining, mana regen |
| **MaxGameSpeed** | 30 | 10–100 | Limits spectator fast-forward | At high speeds, float precision issues and visual artifacts | Spectator experience; derived constant magnitudes |

### Knob Interactions

- **TickDuration × GameSpeed**: These are independent. TickDuration controls
  simulation resolution (how many ticks per second). GameSpeed controls how fast
  the game world moves within each tick. Changing TickDuration affects CPU load;
  changing GameSpeed affects gameplay pacing.
- **TickDuration and Parity**: If TickDuration differs between SimGame and Unity,
  parity WILL break. This value must be identical in both implementations — it
  should be a shared constant in AgentSDK, not configurable per-engine.

### Locked Constants (not tunable)

These values are fixed by design and should NOT be exposed as tuning knobs:

| Constant | Value | Rationale |
|----------|-------|-----------|
| Command sort order | (AgentNbr, CommandType, UnitNbr) | Changing sort order changes game outcomes — breaks replay compatibility |
| One-command-per-unit-per-tick | Enforced | Removing this creates command spam exploits |
| Tick phase order | Fixed 6-phase pipeline | Reordering phases changes game semantics |
| Hash algorithm | FNV-style, seed 17 | Changing breaks all existing parity baselines |
| DIAGONAL_COST | 1.41421356f | Fixed for parity; must not be computed at runtime |

## Acceptance Criteria

#### Determinism
- [ ] Running the same two agents on the same map 100 times produces identical
  state hashes at every tick in all 100 runs
- [ ] SimGame and Unity produce identical full state hashes at every tick for the
  same match
- [ ] All five subsystem hashes (Global, UnitPositions, UnitHealth, UnitActions,
  UnitTimers) match between SimGame and Unity at every tick

#### Tick Ordering
- [ ] Phase order is verifiable by logging: no Phase 2 work occurs before Phase 1
  completes, etc.
- [ ] Commands queued in Phase 5 are never processed until the next tick's Phase 1
- [ ] Dead units (Phase 4) are never advanced (Phase 2) or given mana (Phase 3) in
  the same tick they die — they are removed AFTER those phases

#### Command Processing
- [ ] Commands from both agents in the same tick are interleaved by sort order, not
  processed per-agent
- [ ] A unit receiving two commands in one tick only executes the first (by sort
  order); second is dropped
- [ ] Gold deducted for a Build/Train command is refunded if the action fails
  before Phase 2
- [ ] Failed commands appear in `GetFailedCommands()` on the next tick's IGameState
- [ ] Silently dropped duplicate-unit commands do NOT appear in `GetFailedCommands()`

#### Agent Interaction
- [ ] An agent throwing an exception in Update() does not crash the game or affect
  the other agent
- [ ] An agent issuing zero commands in a tick does not stall the tick loop
- [ ] `IGameState` presented to agents in Phase 5 reflects the complete
  post-Phase-4 state (dead units removed, gold updated, tasks advanced)

#### Game Speed
- [ ] Changing game speed mid-match does not break determinism (both engines
  recompute derived constants identically)
- [ ] At GameSpeed 1, a match at 300-second time limit runs for exactly 6,000 ticks
  (300 / 0.05)
- [ ] At GameSpeed 30, unit movement and damage are 30x faster than at GameSpeed 1,
  verified by state comparison

#### Performance
- [ ] A single tick completes in under 2ms (excluding agent Update time) at default
  map size (30x30) with 50 units per side
- [ ] State hash computation adds no more than 0.5ms per tick
