# Map System

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Fair Competition (symmetric, connected maps)

## Overview

The Map System defines the 2D grid that all gameplay takes place on. Every unit
position, building footprint, pathfinding query, and buildability check operates
against this grid. It provides three boolean layers per cell — walkable, buildable,
and passage — that together determine where units can move, where buildings can be
placed, and how building top-rows interact with movement.

The grid is shared between SimGame and Unity via `AgentSDK/GameGrid.cs`, ensuring
both engines operate on identical spatial data. The Map System owns the grid
structure and cell queries but does NOT own pathfinding (Pathfinding system) or map
generation (Procedural Map Gen system). It is the spatial foundation everything
else sits on.

## Player Fantasy

**"The battlefield is fair — what matters is what you do with it."**

Agents never think about the map system directly, but they depend on it constantly.
The map guarantees that both players start with equal access to resources, equal
distance to the enemy, and identical terrain. The grid is the silent contract that
makes competition meaningful.

This serves the **Fair Competition** pillar: symmetric maps with validated
connectivity ensure neither player has a positional advantage.

## Detailed Design

### Core Rules

#### 1. Grid Structure

- **Coordinate system**: (0,0) at bottom-left, +X right, +Y up
- **Dimensions**: Width × Height integer grid cells (default 30×30, configurable)
- **Minimum size**: 15×15
- **Cell size**: 1 unit = 1 grid cell (integer positions)

#### 2. Three-Layer Cell Model

Each cell has three boolean properties:

| Layer | Default | Meaning |
|-------|---------|---------|
| **Walkable** | true | Mobile units can path through this cell |
| **Buildable** | true | New units/buildings can be placed on this cell |
| **Passage** | false | Building top-row cell — walkable, not buildable, units move freely without claiming/releasing |

The passage flag is required because two different game states share the same
walkable=true/buildable=false combination: building top-row cells (passage) and
cells occupied by mobile units. The movement system must distinguish these —
units move freely through passage cells but wait/detour around occupied cells.

#### 3. Building Footprint System

Buildings occupy multiple cells defined by `UNIT_SIZE[UnitType]` from Balance Data:

- **Anchor**: Top-left corner of the footprint at position (x, y)
- **Extent**: Rightward (+X) and downward (-Y)
- **Cells occupied**: `(x+i, y-j)` for `i in [0..sizeX)`, `j in [0..sizeY)`

Example: 3×3 Barracks anchored at (10, 20) occupies:
```
(10,20) (11,20) (12,20)   ← top row (walkable, not buildable)
(10,19) (11,19) (12,19)   ← not walkable, not buildable
(10,18) (11,18) (12,18)   ← not walkable, not buildable
```

#### 4. Building Cell Rules

When a building is placed:
- All footprint cells → `buildable = false`
- For stationary buildings with height > 1: **top row** (j=0) remains
  `walkable = true` (units can walk over the building's front)
- All other rows → `walkable = false`
- Mobile units (1×1) mark their cell as `buildable = false` but remain
  `walkable = true`

When a building is destroyed:
- All footprint cells revert to `buildable = true` and `walkable = true`

#### 5. Neighbor Ring

The neighbor ring of a unit is all cells immediately surrounding its footprint:
- Top row: `y = anchor.Y + 1`, `x in [anchor.X - 1 .. anchor.X + sizeX]`
- Bottom row: `y = anchor.Y - sizeY`, `x in [anchor.X - 1 .. anchor.X + sizeX]`
- Left column: `x = anchor.X - 1`, `y in [anchor.Y - sizeY + 1 .. anchor.Y]`
- Right column: `x = anchor.X + sizeX`, `y in [anchor.Y - sizeY + 1 .. anchor.Y]`

Neighbor ring positions are used for:
- Determining where pawns stand to build/repair
- Where trained units spawn
- Where gatherers approach mines/bases

Buildable neighbor positions include the neighbor ring PLUS any walkable top-row
cells of the building (these are valid approach positions for interacting units).

#### 6. Buildability Checks

Three levels of buildability validation:

| Check | What It Validates |
|-------|------------------|
| `IsPositionBuildable(pos)` | Single cell buildable query |
| `IsAreaBuildable(type, anchor)` | All cells in footprint are buildable |
| `IsBoundedAreaBuildable(type, anchor)` | Footprint AND 1-cell border are buildable |

`IsBoundedAreaBuildable` is used for `FindProspectiveBuildPositions()` — it
ensures buildings don't touch other buildings or map edges.

#### 7. Position Struct

Immutable integer coordinate pair:
- `int X`, `int Y`
- `Distance(a, b)`: Euclidean distance (float)
- `Center(topLeft, size)`: Center cell of a footprint
- Equality, addition, subtraction operators

### States and Transitions

The Map System has no runtime state machine. The grid is initialized once per
round and then mutated only by:

| Event | Mutation |
|-------|---------|
| **Building placed** | Footprint cells → not buildable; non-top rows → not walkable; top row stays walkable |
| **Building destroyed** | Footprint cells → buildable, walkable |
| **Mobile unit placed** | Cell → not buildable (remains walkable) |
| **Mobile unit moves** | Old cell → buildable; new cell → not buildable |
| **Mobile unit dies** | Cell → buildable |
| **Terrain blocked** | Cell → not walkable, not buildable (set during map generation, never changes) |

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Pathfinding** | Reads walkable/buildable layers | Grid state → path queries |
| **Tick Engine** | Calls grid mutations during Phase 1 (build commands) and Phase 2 (movement/death) | Command results → cell updates |
| **Balance Data** | Reads UNIT_SIZE for footprint dimensions | Unit sizes → footprint calculations |
| **Unit Lifecycle** | Calls grid mutations on spawn/death | Spawn/death events → cell updates |
| **Command Validation** | Reads buildability for build commands | Grid queries → valid/invalid |
| **Agent SDK** | Exposes grid queries via IGameState | IsPositionBuildable, FindProspectiveBuildPositions, etc. |
| **Camera** | Reads grid dimensions for bounds clamping | Map size → camera limits |
| **Debug Overlays** | Reads both layers for visualization | Grid state → color overlays |
| **Procedural Map Gen** | Writes terrain obstacles during generation | Blocked cells → grid setup |

#### Ownership Boundary

The Map System owns:
- Grid structure (dimensions, three boolean layers: walkable, buildable, passage)
- Cell query API (walkable, buildable, area checks)
- Building footprint logic (placement, removal, top-row walkability)
- Neighbor ring calculation
- Position struct and distance calculations

The Map System does NOT own:
- Pathfinding algorithm (owned by Pathfinding system)
- Map generation logic (owned by Procedural Map Gen)
- When/why cells are mutated (owned by Tick Engine / gameplay systems)

## Formulas

### Footprint Cell Enumeration

```
For unit at anchor (x, y) with size (sizeX, sizeY):
  cells = { (x+i, y-j) | i in [0..sizeX), j in [0..sizeY) }
```

### Footprint Center

```
Center(anchor, size) = (anchor.X + floor(sizeX/2), anchor.Y - floor(sizeY/2))
```

For 1×1 units: center = anchor position.

### Euclidean Distance

```
Distance(a, b) = sqrt((a.X - b.X)² + (a.Y - b.Y)²)
```

### Mirror Position (180° Point Reflection)

```
Mirror(pos) = (Width - 1 - pos.X, Height - 1 - pos.Y)

MirrorUnit(pos, sizeX, sizeY) = (Width - sizeX - pos.X, Height - 1 + sizeY - pos.Y)
```

The unit mirror formula accounts for footprint size so that the visual placement
is symmetric.

### Spawn Placement

```
margin = max(3, Width / 10)
Player0.Spawn = (margin, margin)
Player1.Spawn = Mirror(Player0.Spawn)
```

### Mine Placement

```
minDistance = 5 cells from spawn
maxDistance = 8 cells from spawn
Constrained to player's primary half: x in [2, Width/2), y in [2, Height/2)
Player1.Mine = Mirror(Player0.Mine)
```

### Exclusion Radii

```
SpawnExclusion = 4 cells (no obstacles within 4 cells of spawn)
MineExclusion  = 3 cells (no obstacles within 3 cells of mine)
CenterExclusion = 2 cells (no obstacles within 2 cells of map center)
```

## Edge Cases

#### 1. Building placed at map edge
`IsBoundedAreaBuildable` requires a 1-cell border around the footprint. Buildings
cannot be placed within 1 cell of the map edge. This is by design — it prevents
units from being trapped between a building and the boundary.

#### 2. Building footprint partially off-map
`IsAreaBuildable` checks all cells in the footprint. Any cell outside grid bounds
returns false. The build command is rejected.

#### 3. Mobile unit moves onto a building's top-row cell
The cell is walkable (top row of stationary building). The unit passes through
normally. The cell remains not buildable — no building can be placed on top of it.

#### 4. Two mobile units on the same cell
Cells occupied by mobile units are `walkable = true`, so pathfinding allows other
units through. Collision avoidance is handled by the Tick Engine's movement system
(wait → detour → repath), not by the Map System.

#### 5. Building destroyed while unit stands on its footprint
Footprint cells revert to walkable and buildable. The unit standing there is
unaffected — it was already on a walkable cell (top row) or inside the building
footprint (edge case from forced movement). The cell becomes buildable again.

#### 6. Mine depleted (health reaches 0)
Mine is removed by Unit Lifecycle. Its footprint cells (3×3) revert to walkable
and buildable. The space can now be used for building.

#### 7. Map smaller than 15×15
Rejected at generation time. Minimum 15×15 ensures room for two spawns, mines,
and meaningful terrain.

#### 8. All buildable positions exhausted
`FindProspectiveBuildPositions()` returns an empty list. The agent's Build command
will fail validation. This is a legitimate game state — map control through
building placement is a valid strategy.

#### 9. Grid state diverges between SimGame and Unity
Both engines share `AgentSDK/GameGrid.cs`. Any mutation must go through the shared
API. If Unity maintains a separate `GridCells` array, it must be kept in perfect
sync with GameGrid — or eliminated entirely (preferred).

## Dependencies

### Upstream (hard dependencies)

None. The Map System is a foundation layer.

### Service Reads

| System | What It Reads | Nature |
|--------|-------------|--------|
| **Balance Data** | UNIT_SIZE, CAN_MOVE | Footprint dimensions, mobile vs stationary |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Pathfinding** | Hard | Walkable/buildable layers for A* |
| **Tick Engine** | Hard | Grid mutations during command/task processing |
| **Command Validation** | Hard | Buildability queries for build commands |
| **Unit Lifecycle** | Hard | Grid mutations on spawn/death |
| **Agent SDK** | Hard | Grid queries exposed to agents |
| **Camera** | Hard | Grid dimensions for bounds |
| **Debug Overlays** | Soft | Cell state for visualization |
| **Procedural Map Gen** | Hard | Grid API for writing terrain |

## Tuning Knobs

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| **MapWidth** | 30 | 15–100 | Cramped, buildings overlap, no room to maneuver | Matches take forever, units walk for ages | Game length, strategic space, pathfinding cost |
| **MapHeight** | 30 | 15–100 | (Same as width) | (Same as width) | (Same as width) |
| **SpawnExclusion** | 4 | 2–8 | Buildings block spawn, unfair starts | Too much guaranteed safe space, reduces map variety | Early game fairness |
| **MineExclusion** | 3 | 2–6 | Obstacles block mining access | Large guaranteed clear zone around mines | Resource accessibility |
| **CenterExclusion** | 2 | 0–4 | Center may be impassable | Large open area in center reduces terrain variety | Mid-map engagement space |
| **MineMinDistance** | 5 | 3–10 | Mine too close to base, trivial economy | Mine far from base, long gather trips | Economy pacing, early game tempo |
| **MineMaxDistance** | 8 | 5–15 | (Combined with min) | Mine very far, map feels empty | (Same as above) |

### Knob Interactions

- **MapSize × GameSpeed**: Larger maps need higher game speed to avoid tedium. At
  30×30 with speed 20, cross-map travel takes ~1.5s for a Warrior. At 60×60 it
  would take ~3s.
- **SpawnExclusion × UNIT_SIZE[BASE]**: The Base is 6×4. SpawnExclusion of 4
  guarantees room for the base plus a buffer. Reducing exclusion below 4 risks
  the base not fitting.

## Acceptance Criteria

#### Grid Integrity
- [ ] Every cell is queryable for walkable and buildable state in O(1)
- [ ] Grid dimensions match between SimGame and Unity for the same map
- [ ] No cell is ever walkable=false and buildable=true (buildable implies walkable
  for terrain cells)

#### Building Footprints
- [ ] Placing a 3×3 building marks exactly 9 cells as not buildable
- [ ] Top row of stationary buildings remains walkable after placement
- [ ] Destroying a building restores all footprint cells to walkable and buildable
- [ ] `IsBoundedAreaBuildable` rejects placements within 1 cell of map edge

#### Symmetry & Fairness
- [ ] Player 0 and Player 1 spawn positions are point-symmetric through map center
- [ ] Mine positions are point-symmetric through map center
- [ ] Shortest path from spawn to mine is equal (±1 cell) for both players
- [ ] Both spawns can reach each other, their mines, and map center (BFS validated)

#### Parity
- [ ] `GameGrid` state is identical in SimGame and Unity after identical mutation
  sequences
- [ ] Unity does not maintain any secondary grid state that diverges from GameGrid

#### Performance
- [ ] Single cell query (walkable/buildable) completes in O(1)
- [ ] `FindProspectiveBuildPositions` completes in under 5ms on a 30×30 grid
- [ ] `IsAreaBuildable` for largest building (6×4 Base) completes in under 0.1ms
