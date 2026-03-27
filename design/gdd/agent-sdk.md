# Agent SDK

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Easy Onboarding (simple API), Fair Competition (equal access)

## Overview

The Agent SDK is the student-facing API contract — the set of interfaces, data
types, and base classes that students use to build their AI agents. It defines what
agents can see (`IGameState`), what they can do (`IAgentActions`), and how they
participate in the match lifecycle (`IPlanningAgent`). The SDK also contains the
shared game constants and pathfinding code used by both SimGame and Unity.

The SDK is the most important public surface in the project. It must be simple
enough for a student with basic C# knowledge to use in under an hour, stable
enough that agents compiled against it work reliably, and restrictive enough that
agents cannot cheat or break the simulation. Everything a student needs to build
an agent is in this package — nothing outside it should be required.

## Player Fantasy

**"The API is small, clear, and everything I need."**

A student opens the SDK, sees three interfaces (IPlanningAgent, IGameState,
IAgentActions), extends PlanningAgentBase for convenience, and starts writing
strategy. They don't need to understand the engine, the tick loop, or Unity. The
SDK is their entire world — and it's well-documented, well-typed, and
unsurprising.

## Detailed Design

### Core Rules

#### 1. Three Core Interfaces

| Interface | Role | Direction |
|-----------|------|-----------|
| `IPlanningAgent` | Agent lifecycle — the student implements this | Engine calls agent |
| `IGameState` | Read-only query API — what the agent can see | Agent reads from engine |
| `IAgentActions` | Command API — what the agent can do | Agent writes to engine |

#### 2. IPlanningAgent (Student Implements)

```csharp
public interface IPlanningAgent
{
    void InitializeMatch();
    void InitializeRound();
    void Update(IGameState state, IAgentActions actions);
    void Learn(IGameState state);
}
```

| Method | When Called | Purpose |
|--------|-----------|---------|
| `InitializeMatch()` | Once, before first round | Allocate resources, load strategy config |
| `InitializeRound()` | Before each round | Reset per-round state |
| `Update(state, actions)` | Phase 5 of every tick | Observe state, issue commands |
| `Learn(state)` | After each round | Analyze outcome, adapt strategy |

#### 3. IGameState (Agent Reads)

Provides a read-only snapshot of the game after Phase 4 (dead units removed):

**Agent Info:**
- `int MyAgentNbr` — this agent's ID
- `int EnemyAgentNbr` — opponent's ID
- `int MyGold` — own gold pool
- `int EnemyGold` — opponent's gold pool
- `int MyWins` — rounds won so far
- `Position MapSize` — grid dimensions

**Unit Queries:**
- `IReadOnlyList<int> GetMyUnits(UnitType)` — own units by type
- `IReadOnlyList<int> GetEnemyUnits(UnitType)` — enemy units by type
- `IReadOnlyList<int> GetAllUnits(UnitType)` — all units by type
- `UnitInfo? GetUnit(int unitNbr)` — detailed info for one unit (null if dead)

**Spatial Queries:**
- `bool IsPositionBuildable(Position)` — single cell check
- `bool IsAreaBuildable(UnitType, Position)` — full footprint check
- `bool IsBoundedAreaBuildable(UnitType, Position)` — footprint + border check
- `IReadOnlyList<Position> FindProspectiveBuildPositions(UnitType)` — all valid build sites

**Pathfinding Queries:**
- `IReadOnlyList<Position> GetPathBetween(Position start, Position end)`
- `IReadOnlyList<Position> GetPathBetween(Position start, Position end, bool avoidUnits)`
- `IReadOnlyList<Position> GetPathToUnit(Position start, UnitType, Position unitPos)`
- `IReadOnlyList<Position> GetBuildablePositionsNearUnit(UnitType, Position unitPos)`

**Command Feedback:**
- `IReadOnlyList<FailedCommand> GetFailedCommands()` — commands that failed last tick

#### 4. IAgentActions (Agent Writes)

Each method returns a `CommandResult` with immediate queue-time validation:

```csharp
CommandResult Move(int unitNbr, Position target);
CommandResult Attack(int unitNbr, int targetNbr);
CommandResult Train(int buildingNbr, UnitType unitType);
CommandResult Build(int unitNbr, Position target, UnitType unitType);
CommandResult Gather(int pawnNbr, int mineNbr, int baseNbr);
CommandResult Repair(int pawnNbr, int buildingNbr);
CommandResult Heal(int monkNbr, int targetNbr);
void Log(string message);
```

Commands are queued and processed in the next tick's Phase 1. One command per unit
per tick — extras are silently dropped.

#### 5. UnitInfo (Data Transfer Object)

Returned by `GetUnit()`, contains all visible state for a single unit:

- `int UnitNbr`
- `UnitType UnitType`
- `int OwnerAgentNbr`
- `Position GridPosition`
- `float Health`
- `bool IsBuilt`
- `UnitAction CurrentAction`
- `float Mana` (0 for non-Monk units)

UnitInfo is a read-only snapshot — modifying it has no effect on the game.

#### 6. PlanningAgentBase (Convenience Class)

Optional base class students can extend instead of implementing IPlanningAgent
directly. Provides:

- Auto-categorized unit lists (`myPawns`, `myWarriors`, `enemyArchers`, etc.)
- `UpdateGameState()` helper that refreshes all unit lists
- Cached buildable position queries
- Pre-sorted unit collections by type

Students are not required to use PlanningAgentBase — they can implement
IPlanningAgent directly for full control.

#### 7. Shared Code in SDK

The AgentSDK assembly also contains code shared between SimGame and Unity:

- `GameConstants` — all balance values (read-only)
- `DerivedGameConstants` — speed-scaled values
- `Pathfinder` — A* implementation
- `GameGrid` — grid data structure
- `Position` — coordinate struct
- `CommandValidator` — command validation logic
- `TaskEngine` — shared task advancement formulas

This shared code is the parity guarantee — both engines call the same methods.

#### 8. Target Framework

- **AgentSDK**: .NET Standard 2.1 (compatible with Unity and standalone .NET)
- **Student agents**: .NET Standard 2.1 (compile against AgentSDK only)
- Students reference AgentSDK.dll — no Unity dependency, no test harness dependency

### States and Transitions

The Agent SDK is stateless infrastructure — it defines interfaces, not behavior.
The only state is within student agent implementations, which the SDK does not
control.

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Tick Engine** | Calls IPlanningAgent methods at correct lifecycle points | Agent lifecycle orchestration |
| **Match Flow** | Calls InitializeMatch, InitializeRound, Learn | Match lifecycle |
| **All gameplay systems** | IGameState exposes their state to agents | Read-only state exposure |
| **Command Validation** | IAgentActions commands flow through validation | Command submission |
| **Agent Loader** | Loads student DLLs implementing IPlanningAgent | Assembly loading |
| **Agent Security** | Restricts what agents can access beyond the SDK | Sandboxing |
| **Balance Data** | GameConstants accessible to agents | Balance visibility |
| **Pathfinding** | Exposed via IGameState path queries | Agent planning |

#### Ownership Boundary

Agent SDK owns:
- IPlanningAgent, IGameState, IAgentActions interfaces
- UnitInfo, CommandResult, FailedCommand data types
- PlanningAgentBase convenience class
- UnitType, UnitAction, Position shared types
- Shared code (GameConstants, Pathfinder, GameGrid, CommandValidator, TaskEngine)

Agent SDK does NOT own:
- How agents are loaded (Agent Loader)
- How agents are sandboxed (Agent Security)
- How the game loop calls agents (Tick Engine)
- Student agent implementations

## Formulas

No formulas — the SDK is an interface definition layer. All formulas are in the
systems that implement the behavior exposed through the SDK (Combat, Economy,
Pathfinding, etc.).

## Edge Cases

#### 1. Agent calls GetUnit() with an invalid ID
Returns null. Agents must null-check.

#### 2. Agent issues commands for units it doesn't own
CommandResult returns `NotOwned`. Command is not queued.

#### 3. Agent calls pathfinding with same start and end
Returns empty list. Agent should interpret empty list as "already there" or
"no path."

#### 4. Agent issues 100+ commands in one Update
All commands are queued. The one-per-unit rule drops duplicates during Phase 1.
No performance penalty beyond queue-time validation cost.

#### 5. Agent stores references to IGameState between ticks
The IGameState instance may be reused/recycled between ticks. Agents should NOT
store references to it — copy any needed data during Update. UnitInfo values
are safe to store (they are value copies).

#### 6. Agent calls Log() with very long messages
Log messages are truncated at a reasonable limit (e.g., 1000 characters). Logging
does not affect game state or determinism.

#### 7. Agent modifies UnitInfo fields
UnitInfo is a read-only struct/class. If implemented as a class, fields should be
readonly. Modifying a local copy has no effect on game state.

#### 8. GetFailedCommands() returns commands from the previous tick only
The list is cleared each tick. Agents cannot query failures from older ticks.

#### 9. Student agent references assemblies outside AgentSDK
Agent Security (Tier 2) will restrict this. Currently unprotected — students
could reference System.IO, System.Net, etc. The SDK design assumes sandboxing
will be added later.

## Dependencies

### Upstream

| System | What It Provides |
|--------|-----------------|
| **Balance Data** | GameConstants shared in the SDK assembly |
| **Map System** | GameGrid shared in the SDK assembly |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Agent Loader** | Hard | IPlanningAgent interface to discover and instantiate agents |
| **Agent Security** | Hard | SDK surface to whitelist |
| **Opponent Library** | Hard | Interfaces to implement AI strategies |
| **Student Docs** | Hard | API surface to document |
| **Tick Engine** | Hard | IPlanningAgent interface to call |
| **Match Flow** | Hard | Agent lifecycle methods |

## Tuning Knobs

| Knob | Current | Safe Range | Impact |
|------|---------|------------|--------|
| **Max path queries per Update** | Unlimited | 10–1000 | Performance protection; too low limits agent intelligence |
| **Max commands per Update** | Unlimited | 50–500 | Performance protection; too low limits micro-management |
| **Log message max length** | 1000 chars | 100–10000 | Debug utility vs memory cost |

These limits are not currently enforced — they are recommendations for the Agent
Security system (Tier 2).

## Acceptance Criteria

#### Interface Completeness
- [ ] IGameState exposes all information an agent needs to make decisions (gold,
  units, map, paths, failed commands)
- [ ] IAgentActions exposes all 7 command types with CommandResult feedback
- [ ] IPlanningAgent has all 4 lifecycle methods
- [ ] UnitInfo contains all visible unit state

#### Usability
- [ ] A student can implement a basic agent (gather + train) in under 50 lines
  using PlanningAgentBase
- [ ] PlanningAgentBase correctly categorizes all unit types into named lists
- [ ] All interfaces are well-typed — no string-based APIs, no magic numbers
- [ ] Null returns are documented (GetUnit returns null for dead/invalid units)

#### Stability
- [ ] AgentSDK assembly version is tracked and checked at DLL load time
- [ ] Adding a new IGameState query method does not break existing agent DLLs
  (interface extension via default methods or new interface)
- [ ] Removing or renaming an existing method breaks compilation (caught at
  build time, not runtime)

#### Parity
- [ ] Shared code (GameConstants, Pathfinder, GameGrid, CommandValidator,
  TaskEngine) is identical in SimGame and Unity — single source, single assembly
- [ ] IGameState returns identical data in both engines for the same tick

#### Framework Compatibility
- [ ] AgentSDK targets .NET Standard 2.1
- [ ] Student agents can reference AgentSDK.dll with no other dependencies
- [ ] AgentSDK compiles without Unity references