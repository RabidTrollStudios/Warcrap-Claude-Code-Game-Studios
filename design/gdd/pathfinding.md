# Pathfinding

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Fair Competition (deterministic pathing)

## Overview

The Pathfinding system computes grid paths for all unit movement in Agents of
Empires. It implements A* search with 8-directional movement on the Map System's
grid, supporting two modes: normal pathing (walkable cells) and unit-avoidance
pathing (treats occupied cells as impassable). It also provides path-to-unit
queries for approaching multi-cell buildings.

Pathfinding is shared code in `AgentSDK/Pathfinder.cs`, consumed identically by
SimGame and Unity. It is a stateless service — it reads the grid, computes a path,
and returns a position list. It owns no persistent state and has no update loop.

## Player Fantasy

**"My units find their way — I focus on strategy, not navigation."**

Agents never manually route units cell-by-cell. They issue high-level commands
(move here, attack that, gather from this mine) and the pathfinding system handles
navigation. The quality of pathfinding directly affects how "smart" an agent's army
looks to spectators — units that get stuck or take absurd routes undermine the
spectator experience.

## Detailed Design

### Core Rules

#### 1. Algorithm

- **A* search** with Euclidean distance heuristic
- **8-directional movement**: N, NE, E, SE, S, SW, W, NW
- **Edge costs**: 1.0f for cardinal moves, 1.41421356f for diagonal moves (fixed
  constant, per Tick Engine GDD)
- **Tie-breaking**: lowest f-score, then highest g-score, then lowest node key
  (ensures deterministic path selection)
- **Max expansions**: 2000 (configurable) — if exceeded, returns empty list (no
  path found)

#### 2. Grid Modes

Two pathfinding modes using the Map System's two-layer grid:

| Mode | Grid Layer | Use Case |
|------|-----------|----------|
| **Normal** (`avoidUnits = false`) | Walkable layer | Default movement — mobile units are walkable, path through them |
| **Avoid Units** (`avoidUnits = true`) | Buildable layer | Collision detour — treats unit-occupied cells as impassable |

#### 3. Path Queries

| Query | Parameters | Returns | Used By |
|-------|-----------|---------|---------|
| `FindPath(start, end)` | Two positions | Position list (excludes start, includes end) | Move commands, agent queries |
| `FindPath(start, end, avoidUnits)` | Two positions + mode | Position list | Collision detour/repath |
| `FindPathToUnit(start, unitType, unitAnchor)` | Start + target unit info | Path to nearest buildable/walkable neighbor of unit | Gather, Build, Repair, Heal, Attack approach |

`FindPathToUnit` tries all neighbor ring positions plus walkable top-row cells of
the target building, returning the first successful path found.

#### 4. Path Format

- Returns `List<Position>` — ordered sequence of grid cells from near-start to
  destination
- Start position is **excluded** from the path
- End position (or neighbor cell for path-to-unit) is **included**
- Empty list means no path exists (or expansion budget exceeded)

#### 5. Start Position Exception

The start position is allowed to be on an impassable cell (the unit may be inside
a building footprint or on an obstacle due to forced placement). The pathfinder
skips the start cell's passability check and begins expanding from its neighbors.

#### 6. Collision Avoidance (owned by Tick Engine, uses Pathfinding)

When a unit's path is blocked by another mobile unit during Phase 2 movement, the
Tick Engine invokes a three-phase response:

1. **Wait** (0–3 ticks): Unit pauses, hoping the blocker moves
2. **Detour** (3+ ticks): Repath with `avoidUnits = true`
3. **Full repath** (10+ ticks): Full A* repath with `avoidUnits = true`

When blocked by terrain (building placed on path), immediate repath with
`avoidUnits = false`. If no path exists, unit goes IDLE.

The collision avoidance logic is owned by the Tick Engine — Pathfinding just
computes the routes on request.

### States and Transitions

Pathfinding is stateless. It is a pure function: grid state in → path out. No
internal state persists between calls.

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Map System** | Reads walkable/buildable layers | Grid state → passability checks |
| **Tick Engine** | Called during Phase 2 (movement, attack approach, gather walking) and collision repath | Path requests → position lists |
| **Agent SDK** | Exposed via `IGameState.GetPathBetween()` and `GetPathToUnit()` | Agents can query paths for planning |
| **Balance Data** | Reads UNIT_SIZE for building neighbor calculations | Footprint dimensions |

#### Ownership Boundary

Pathfinding owns:
- A* algorithm implementation
- Path computation (grid → position list)
- Max expansion budget enforcement
- Deterministic tie-breaking

Pathfinding does NOT own:
- Grid state (owned by Map System)
- When paths are computed (owned by Tick Engine)
- Collision avoidance decisions (owned by Tick Engine)
- Movement along paths (owned by Tick Engine)

## Formulas

### A* Cost Functions

```
g(node) = actual cost from start to node
h(node) = Euclidean distance from node to goal (heuristic)
f(node) = g(node) + h(node)

EdgeCost(cardinal) = 1.0f
EdgeCost(diagonal) = 1.41421356f

h(a, b) = sqrt((a.X - b.X)² + (a.Y - b.Y)²)
```

### Tie-Breaking Sort

```
Compare(a, b):
  1. Lowest f(node) first
  2. If f equal: highest g(node) first (prefer longer known paths)
  3. If g equal: lowest node key first (deterministic)
```

### Path-to-Unit Neighbor Enumeration

```
For unit at anchor (x, y) with size (sizeX, sizeY):
  neighbors = neighbor ring positions (see Map System GDD)
             + walkable top-row cells of building
               (y = anchor.Y, x in [anchor.X .. anchor.X + sizeX))
```

## Edge Cases

#### 1. No path exists
Returns empty list. The calling system (Tick Engine) handles this — typically the
unit goes IDLE.

#### 2. Max expansions exceeded (2000 default)
Returns empty list, same as no path. This prevents pathfinding from consuming
unbounded CPU on large or maze-like maps. The unit may retry next tick when the
grid has changed.

#### 3. Start and end are the same position
Returns empty list (no movement needed).

#### 4. Start position is impassable
Allowed — the pathfinder skips the start cell's passability check. This handles
units displaced onto buildings or obstacles.

#### 5. End position is impassable
Returns empty list. Cannot path to an unwalkable cell.

#### 6. Diagonal movement through narrow gap
A* allows diagonal movement between two cardinal-adjacent obstacles. This is
intentional — units can squeeze through 1-cell diagonal gaps. If this feels wrong
visually, it's a presentation concern, not a pathfinding concern.

#### 7. Grid changes between path computation and path consumption
Paths are computed once and then consumed over multiple ticks. If the grid changes
(building placed on path), the Tick Engine detects the blockage during movement
and triggers a repath. The pathfinding system itself is not responsible for path
invalidation.

#### 8. Multiple FindPathToUnit candidates
When multiple neighbor cells are reachable, the first successful path found is
returned. The order of neighbor enumeration is deterministic (top → bottom →
left → right) to ensure parity.

#### 9. Agent calls GetPathBetween during Update
Agents can query paths for planning purposes. These calls use the same A*
implementation and expansion budget. Agents making many path queries per tick will
consume CPU but not affect determinism.

## Dependencies

### Upstream (hard dependencies)

| System | What It Provides |
|--------|-----------------|
| **Map System** | Walkable/buildable grid layers |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Tick Engine** | Hard | Path computation during Phase 2 movement and collision repath |
| **Agent SDK** | Hard | Path queries exposed to agents via IGameState |
| **Combat** | Indirect | Attack approach uses FindPathToUnit |
| **Economy** | Indirect | Gather walking uses FindPathToUnit |
| **Production** | Indirect | Build walking uses FindPathToUnit |
| **Healing** | Indirect | Heal approach uses FindPathToUnit |

## Tuning Knobs

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| **MaxExpansions** | 2000 | 500–10000 | Paths fail on complex maps, units get stuck | CPU spike on maze maps with many units | Pathfinding success rate vs performance |
| **CollisionWaitTicks** | 3 | 1–10 | Detours too aggressively, jittery movement | Stands still too long before reacting | Movement smoothness |
| **CollisionRepathTicks** | 10 | 5–20 | Full repaths too often, CPU cost | Stuck units stand still for too long | Responsiveness vs performance |

Note: CollisionWaitTicks and CollisionRepathTicks are owned by the Tick Engine but
directly affect how often pathfinding is invoked.

## Acceptance Criteria

#### Correctness
- [ ] A* returns optimal (shortest) paths on open terrain
- [ ] Paths respect walkable layer in normal mode and buildable layer in
  avoid-units mode
- [ ] Diagonal cost (1.41421356f) is applied correctly for diagonal moves
- [ ] Tie-breaking produces identical paths in SimGame and Unity for the same
  grid state

#### Determinism
- [ ] Same grid state + same start/end → same path, every time
- [ ] FindPathToUnit neighbor enumeration order is deterministic
- [ ] Path results are identical between SimGame and Unity

#### Budget
- [ ] Pathfinding never expands more than MaxExpansions nodes per call
- [ ] A single FindPath call on a 30×30 grid completes in under 2ms
- [ ] 50 concurrent path requests per tick complete within 10ms total

#### Edge Cases
- [ ] Start == End returns empty list
- [ ] Impassable start is handled (path begins from neighbors)
- [ ] Impassable end returns empty list
- [ ] No-path scenario returns empty list (not null, not exception)
