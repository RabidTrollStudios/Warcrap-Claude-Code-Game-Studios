# Sim Parity

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Fair Competition (identical results across engines)

## Overview

Sim Parity is the verification system that ensures SimGame (headless C# simulator)
and Unity (visual frontend) produce identical game state at every tick for the same
inputs. This is the #1 technical priority — if parity breaks, match results are
unreliable, tournament outcomes are unfair, and the entire competition framework
loses credibility.

Parity is achieved through shared code (AgentSDK), verified through state hashing
(per-tick 64-bit hash comparison), and diagnosed through subsystem hashes (5
independent hash channels that isolate where divergence occurs). The system does
not own any gameplay logic — it is a pure verification and diagnostic layer.

## Player Fantasy

**"I can test my agent in the headless sim and know it will behave identically in the visual game."**

Students iterate fast using SimGame (millisecond test runs, no rendering overhead)
and trust that their agent will perform identically when loaded into Unity for
spectating or tournaments. Without parity, fast iteration is impossible — students
would need to test in Unity every time, which is orders of magnitude slower.

## Detailed Design

### Core Rules

#### 1. Parity Definition

Two engines are "in parity" when, given:
- Same map (identical grid, spawn positions, mine positions)
- Same agents (identical DLL binaries)
- Same config (GameSpeed, StartingGold, etc.)
- Same random seed (if any randomness exists)

They produce **identical state hashes at every tick** for the entire match.

#### 2. Parity Architecture

Parity is achieved by construction, not by synchronization:

```
AgentSDK (shared code)
├── GameConstants.cs      ← Single source of balance values
├── DerivedGameConstants.cs ← Single derivation formulas
├── Pathfinder.cs         ← Single A* implementation
├── GameGrid.cs           ← Single grid data structure
├── CommandValidator.cs   ← Single validation logic
├── TaskEngine.cs         ← Single task advancement formulas
└── Position.cs           ← Single coordinate math

SimGame reads AgentSDK ──→ Identical behavior
Unity reads AgentSDK   ──→ Identical behavior
```

The rule: **any gameplay logic that affects state must live in AgentSDK**. Engine-
specific code (Unity MonoBehaviours, rendering, UI) must never modify game state.

#### 3. State Hash Verification

After each tick's Phase 4 (Remove Dead Units), a 64-bit hash is computed over the
complete game state. See Tick Engine GDD Formulas section for the full hash
specification.

Both engines compute their hash independently. Parity testing compares hashes at
every tick. If any hash differs, the engines have diverged.

#### 4. Subsystem Hashes (Diagnostic)

Five independent hashes isolate divergence to a specific area:

| Subsystem | What It Hashes | Divergence Means |
|-----------|---------------|-----------------|
| **Global** | Tick count, gold, unit count, NextUnitNbr | Tick timing, economy, or spawn count mismatch |
| **UnitPositions** | UnitNbr, UnitType, Owner, GridPosition | Movement or spawn location mismatch |
| **UnitHealth** | Health, IsBuilt | Damage, healing, or build completion mismatch |
| **UnitActions** | CurrentAction, all target references | Command processing or task assignment mismatch |
| **UnitTimers** | MoveAccumulator, TrainTimer, BuildTimer, MiningTimer, Mana, PathIndex, etc. | Timer advancement or accumulation mismatch |

When the full hash diverges, check subsystem hashes to narrow the cause. If
UnitHealth diverges but UnitPositions matches, the problem is in damage/healing,
not movement.

#### 5. Parity Test Framework

The existing NUnit test suite (`PlanningAgent.Tests`) runs parity scenarios:

1. **Scenario setup**: Configure map, agents, game speed
2. **Dual execution**: Run the same scenario in SimGame and capture command logs
3. **Command replay**: Replay the captured commands in a second SimGame instance
4. **Hash comparison**: Compare state hashes at every tick
5. **Divergence report**: If hashes differ, report tick number, subsystem, and
   state snapshots

#### 6. Parity Risk Categories

| Risk | Source | Mitigation |
|------|--------|-----------|
| **Float precision** | Different computation order in SimGame vs Unity | Use shared code; fixed float constants (DIAGONAL_COST = 1.41421356f); avoid double |
| **Collection ordering** | Dictionary iteration order differs | Always sort by UnitNbr before iterating |
| **Unity-specific state** | MonoBehaviour modifies game state | Enforce rule: Unity code never mutates AgentSDK state |
| **Timing differences** | FixedUpdate vs manual tick loop | Both must call identical tick phases in identical order |
| **Duplicate constants** | RTS/Constants.cs copies values | Eliminate duplication — read from AgentSDK directly |

#### 7. Parity-Breaking Changes

Any change to these requires simultaneous update in both engines:
- Tick phase order
- Command sort order
- Task advancement logic
- Pathfinding algorithm
- Grid mutation logic
- Hash computation

If any of these differ between engines, parity is broken. The shared-code
architecture prevents most of these — but Unity-side overrides or workarounds are
the primary risk.

### States and Transitions

Sim Parity has no runtime state. It is a verification layer that runs during
testing, not during gameplay. During a live match, only the hash computation runs
(as part of the Tick Engine) — comparison happens externally.

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Tick Engine** | Hash computed after Phase 4 each tick | State → hash |
| **All gameplay systems** | Their behavior must be identical in both engines | Behavioral contract |
| **Agent SDK** | Shared code guarantees identical logic | Parity by construction |
| **Match Recording** | Command logs enable replay-based parity testing | Replay data |
| **Tournament Runner** | Parity verification before tournament matches | Pre-match validation |

#### Ownership Boundary

Sim Parity owns:
- State hash computation (full hash + 5 subsystem hashes)
- Parity test framework (scenario setup, dual execution, comparison)
- Divergence diagnosis (subsystem isolation, state snapshots)
- Parity rules (what must be shared, what can differ)

Sim Parity does NOT own:
- Gameplay logic (owned by individual systems)
- Shared code maintenance (owned by each system's responsible team)
- The hash algorithm itself (defined in Tick Engine GDD)

## Formulas

### State Hash

See Tick Engine GDD Formulas section for the complete 64-bit FNV-style hash
specification, including per-unit field enumeration and float-to-bits conversion.

### Divergence Location

```
If FullHash differs:
  Check SubsystemHash[Global]      → Gold, tick, counts
  Check SubsystemHash[Positions]   → Movement, spawn locations
  Check SubsystemHash[Health]      → Damage, healing, build
  Check SubsystemHash[Actions]     → Command processing, tasks
  Check SubsystemHash[Timers]      → Accumulators, progress

First diverging subsystem = likely source of the bug.
```

## Edge Cases

#### 1. Parity breaks on tick 1
Most likely cause: round setup differs (different starting positions, different
starting gold, different map initialization). Check Global subsystem hash first.

#### 2. Parity breaks after many ticks
Gradual float accumulation drift. Check UnitTimers subsystem — MoveAccumulator
and MiningTimer are the most likely culprits. Ensure both engines use float
(not double) and identical computation order.

#### 3. Parity holds for simple agents but breaks for complex ones
Agent issues many commands per tick, exposing command sort order or one-per-unit
deduplication differences. Test with the same agent in both engines to isolate.

#### 4. Parity holds in isolation but breaks with specific map
Map generation differs between engines. Ensure both use the same map source
(either both generate with same seed, or both load from serialized map data).

#### 5. Hash collision (different state, same hash)
Extremely unlikely with 64-bit hash (1 in 2^64). If suspected, compare full state
snapshots instead of hashes. The subsystem hashes provide additional collision
resistance (5 independent 64-bit values).

#### 6. Agent uses System.Random or other non-deterministic APIs
Agent behavior becomes non-deterministic — parity breaks unpredictably. Agent
Security (Tier 2) should restrict access to non-deterministic APIs. For now,
document as a known risk.

#### 7. Unity FixedUpdate timing drift
Unity's FixedUpdate may be called at slightly different intervals than SimGame's
manual tick loop. Both must process exactly one tick per call, not accumulate
delta time. If Unity's time step differs from TickDuration, state will diverge.

## Dependencies

### Upstream

| System | What It Provides |
|--------|-----------------|
| **Tick Engine** | Hash computation infrastructure, tick phase ordering |
| **All gameplay systems** | Correct behavior in both engines via shared code |
| **Agent SDK** | Shared code that guarantees behavioral identity |
| **Match Recording** | Command logs for replay-based testing |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Tournament Runner** | Hard | Parity verification before competitive matches |
| **Student confidence** | Soft | Trust that SimGame testing is reliable |

## Tuning Knobs

| Knob | Current | Safe Range | Impact |
|------|---------|------------|--------|
| **Hash frequency** | Every tick | Every tick – every 10 ticks | Less frequent = less CPU, slower divergence detection |
| **Subsystem hash enabled** | Yes | Yes/No | Disabling saves ~0.2ms/tick but loses diagnostic granularity |
| **Max state snapshot size** | Unlimited | 1–100 MB | Limits memory for divergence debugging |

## Acceptance Criteria

#### Core Parity
- [ ] SimGame and Unity produce identical full state hashes for 100% of ticks in
  a standard test suite (all 15+ opponent matchups)
- [ ] All 5 subsystem hashes match at every tick
- [ ] Parity holds across all GameSpeed settings (1, 10, 20, 30)
- [ ] Parity holds for all map templates (OpenField, Forest, Maze)

#### Shared Code
- [ ] All gameplay-affecting logic lives in AgentSDK — no engine-specific
  game state mutations
- [ ] RTS/Constants.cs contains zero duplicated balance values
- [ ] GameConstants.MOVEMENT_SPEED (legacy) is removed

#### Diagnostics
- [ ] Divergence report identifies the first tick where hashes differ
- [ ] Subsystem hashes correctly isolate the diverging area
- [ ] State snapshots can be captured for manual comparison

#### Regression
- [ ] Parity test suite runs as part of the build pipeline (build-all.sh)
- [ ] Any code change that breaks parity is detected before merge
- [ ] New gameplay features include parity tests

#### Performance
- [ ] Hash computation adds no more than 0.5ms per tick
- [ ] Parity test suite (full run) completes in under 60 seconds