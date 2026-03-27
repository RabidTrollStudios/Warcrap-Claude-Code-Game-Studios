# ADR-0001: Dual-Engine Parity Architecture

## Status
Accepted

## Date
2026-03-26

## Context

### Problem Statement
Agents of Empires requires two execution environments: a headless C# simulator
(SimGame) for fast agent testing and tournament execution, and a Unity visual
frontend for spectating matches. Students iterate using SimGame (millisecond test
runs) and expect their agents to behave identically in Unity. If the two engines
produce different results for the same inputs, match outcomes are unreliable,
tournament results are unfair, and student trust in the platform collapses.

### Constraints
- Unity uses MonoBehaviours and FixedUpdate; SimGame uses a manual tick loop
- Unity has rendering, physics, and input systems that SimGame does not
- Both must run on the same machine (no network synchronization needed)
- Student agents are compiled as .NET Standard 2.1 DLLs loaded by both engines
- Game speed is adjustable at runtime (1-30x) without breaking determinism

### Requirements
- Identical game state at every tick for identical inputs
- Verifiable parity (automated testing, not manual comparison)
- Shared game rules — changing a balance value in one place updates both engines
- Students compile against one SDK, not two separate APIs

## Decision

**All gameplay-affecting logic lives in a shared .NET Standard 2.1 assembly
(`AgentSDK`) consumed by both SimGame and Unity. Neither engine implements its
own version of any game rule.**

### Architecture Diagram

```
┌─────────────────────────────────────────────────┐
│                  AgentSDK                        │
│              (netstandard2.1)                    │
│                                                  │
│  GameConstants    DerivedGameConstants            │
│  GameGrid         Pathfinder                     │
│  CommandValidator  TaskEngine                    │
│  Position         UnitType / UnitAction          │
│  IPlanningAgent   IGameState   IAgentActions     │
│  PlanningAgentBase CommandResult                 │
└──────────┬──────────────────────┬────────────────┘
           │                      │
           ▼                      ▼
┌──────────────────┐   ┌──────────────────────────┐
│     SimGame      │   │      Unity (RTS)         │
│  (netstandard2.1)│   │   (Unity 6.0.3)          │
│                  │   │                           │
│  SimGame.cs      │   │  GameManager.cs           │
│  SimUnit.cs      │   │  Unit.cs                  │
│  SimMap.cs       │   │  MapManager.cs            │
│  SimGameState.cs │   │  AgentController.cs       │
│                  │   │  + Rendering, UI, Camera   │
│  Calls AgentSDK  │   │  Calls AgentSDK           │
│  for ALL rules   │   │  for ALL rules            │
└──────────────────┘   └──────────────────────────┘
```

**Rule**: Engine-specific code (SimGame.cs, GameManager.cs, Unit.cs) manages
entity storage, rendering, and I/O. It NEVER implements game rules — it
delegates all rule evaluation to AgentSDK methods.

### Key Interfaces

**Shared rule entry points in AgentSDK:**
- `CommandValidator.Validate(command, gameState)` — command validation
- `TaskEngine.Advance*(unit, tickDuration, ...)` — task advancement formulas
- `TaskEngine.ComputeDamagePerTick(...)` — damage calculation
- `TaskEngine.ComputeHealAmount(...)` — heal calculation
- `Pathfinder.FindPath(grid, start, end, ...)` — pathfinding
- `GameGrid.*` — all grid queries and mutations
- `DerivedGameConstants.Compute(gameSpeed)` — speed-scaled constants

**Parity verification:**
- `SimGame.GetStateHash()` — 64-bit FNV hash of complete game state
- 5 subsystem hashes for diagnostic isolation (Global, Positions, Health,
  Actions, Timers)
- Both engines compute hashes independently; test framework compares them

## Alternatives Considered

### Alternative 1: Single Engine (Unity Only)
- **Description**: Run everything in Unity. No SimGame. Students test in Unity.
- **Pros**: No parity problem. Single codebase. Simple.
- **Cons**: Unity startup is slow (~5-10s). Each test match takes seconds, not
  milliseconds. Tournament of 100 matches takes hours instead of minutes. Students
  can't run quick iteration loops. No headless CI testing.
- **Rejection Reason**: Unacceptable iteration speed for students and tournaments.

### Alternative 2: Separate Implementations with Sync Tests
- **Description**: SimGame and Unity each implement their own game logic. Parity
  tests catch divergences.
- **Pros**: Each engine can optimize independently. No shared assembly constraints.
- **Cons**: Every bug fix and balance change must be made twice. Divergences are
  inevitable and hard to diagnose. Maintenance cost is double. Parity tests can
  only catch known scenarios, not unknown edge cases.
- **Rejection Reason**: Unsustainable maintenance burden. Parity-by-testing is
  fundamentally weaker than parity-by-construction.

### Alternative 3: Code Generation from Spec
- **Description**: Define game rules in a DSL or data format. Generate C# for
  both engines from the same source.
- **Pros**: True single source of truth. Could generate optimized code per engine.
- **Cons**: Massive upfront investment. DSL design is hard. Generated code is hard
  to debug. Overkill for this project's scope.
- **Rejection Reason**: Complexity far exceeds the problem. Shared assembly
  achieves the same goal with standard tooling.

## Consequences

### Positive
- **Parity by construction**: Same code runs in both engines — no divergence possible
  for logic in AgentSDK
- **Single maintenance**: Balance changes, bug fixes, and new features update once
- **Fast student iteration**: SimGame runs matches in milliseconds
- **Automated verification**: State hashing confirms parity at every tick
- **Student trust**: "If it works in SimGame, it works in Unity" is a guarantee

### Negative
- **AgentSDK constraints**: Must target .NET Standard 2.1 (no Unity-specific or
  .NET 8-specific APIs in shared code)
- **Abstraction overhead**: Engine-specific code must call through AgentSDK methods
  rather than implementing logic directly — adds indirection
- **Tight coupling**: Changes to AgentSDK affect both engines simultaneously —
  no independent evolution
- **Unity-side duplication risk**: Developers may accidentally duplicate logic in
  Unity code (e.g., RTS/Constants.cs copying values). Requires discipline.

### Risks
- **Float precision**: Different computation order in engine wrappers could cause
  float divergence. **Mitigation**: All float math lives in AgentSDK. Fixed
  constants (DIAGONAL_COST = 1.41421356f). Never use double for game state.
- **Unity timing**: FixedUpdate may behave differently than SimGame's manual loop.
  **Mitigation**: Both process exactly one tick per call. No delta-time accumulation.
- **Accidental duplication**: Unity code reimplements a rule instead of calling
  AgentSDK. **Mitigation**: Eliminate RTS/Constants.cs duplication. Code review
  rule: any gameplay-affecting logic in Unity code is a bug.

## Performance Implications
- **CPU**: Negligible overhead from shared assembly indirection (~nanoseconds per call)
- **Memory**: AgentSDK assembly loaded once, shared by all references (~100KB)
- **Load Time**: No impact — AgentSDK is a small assembly
- **Network**: N/A — both engines run on the same machine

## Migration Plan

Current state has partial duplication (`RTS/Constants.cs` copies values from
AgentSDK). Migration steps:

1. **Audit RTS/Constants.cs** — identify all duplicated dictionaries
2. **Replace duplicates** with direct references to `GameConstants.*` and
   `DerivedGameConstants.*`
3. **Keep Unity-only values** in Constants.cs (colors, direction vectors, UI constants)
4. **Remove `GameConstants.MOVEMENT_SPEED`** (legacy, replaced by
   `DerivedGameConstants.SPEED_MULTIPLIER`)
5. **Run full parity test suite** after migration to verify no regressions
6. **Add CI check**: Grep for any new dictionary definitions in RTS/ that
   duplicate AgentSDK values

## Validation Criteria

- [ ] Full parity test suite passes (all 15+ opponent matchups, all map templates,
  all game speeds)
- [ ] All 5 subsystem hashes match at every tick across the full test suite
- [ ] `RTS/Constants.cs` contains zero duplicated balance dictionaries
- [ ] `GameConstants.MOVEMENT_SPEED` is removed
- [ ] No gameplay-affecting code exists in Unity that doesn't delegate to AgentSDK

## Related Decisions
- [design/gdd/tick-engine.md](../../design/gdd/tick-engine.md) — Tick phase ordering and state hash spec
- [design/gdd/balance-data.md](../../design/gdd/balance-data.md) — Single source of truth for constants
- [design/gdd/sim-parity.md](../../design/gdd/sim-parity.md) — Parity verification system
- [design/gdd/map-system.md](../../design/gdd/map-system.md) — Shared GameGrid, 2-layer simplification
- ADR-0002 (planned): Command Feedback & Gold Refund — new API additions to AgentSDK
