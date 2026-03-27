# Procedural Map Gen

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Fair Competition (symmetric, connected maps), Strategic Depth (map variety)

## Overview

Procedural Map Gen creates randomized but fair game maps for Agents of Empires.
It generates terrain obstacles, places spawn points and mines symmetrically, and
validates that all critical locations are connected. Three templates (OpenField,
Forest, Maze) provide distinct gameplay experiences while maintaining the fairness
guarantees required for competitive play.

The generator runs once before each round. It outputs a configured GameGrid with
blocked cells, spawn positions, and mine positions. It is used by both SimGame
(AgentTestHarness/MapGenerator) and Unity (ProceduralMapGenerator) — both must
produce identical maps for the same seed and config.

## Player Fantasy

**"Every map is different, but every map is fair."**

Map variety prevents agents from memorizing a single optimal strategy. But
randomness never creates an unfair advantage — both players have symmetric access
to resources, equal distance to the center, and guaranteed pathable routes. The map
is a puzzle to solve, not a lottery to win.

## Detailed Design

### Core Rules

#### 1. Generation Inputs

| Parameter | Default | Description |
|-----------|---------|-------------|
| Seed | 42 | Random seed for deterministic generation |
| Width | 30 | Grid width (minimum 15) |
| Height | 30 | Grid height (minimum 15) |
| PlayerCount | 2 | Only 2 supported currently |
| ObstacleDensity | Template-dependent | Target fraction of cells as obstacles (0.0–1.0) |
| MinesPerPlayer | 1 | Mines per player |
| Symmetry | Mirror | Mirror (180° point) or None |
| Template | OpenField | OpenField, Forest, or Maze |

#### 2. Generation Output

| Output | Description |
|--------|-------------|
| `GameGrid` | Configured grid with blocked cells |
| `SpawnPositions[]` | One per player, symmetric |
| `MinePositions[]` | Mines per player, symmetric |
| `Groves[]` | Tree grove data for visual rendering (tree type + cell list) |
| `BlockedCells` | Set of all impassable cell positions |

#### 3. Generation Pipeline

```
1. Create empty grid (all cells walkable + buildable)
2. Compute spawn positions (corners, symmetric)
3. Compute mine positions (near spawns, symmetric)
4. Define exclusion zones (spawn, mine, center)
5. Generate obstacles (template-specific algorithm)
6. Apply symmetry (mirror obstacles to secondary half)
7. Validate connectivity (BFS: spawns ↔ mines ↔ center ↔ each other)
8. If validation fails → retry with new RNG state (max 50 retries)
9. If all retries fail → fallback to open map (no obstacles)
```

#### 4. Spawn Placement

```
margin = max(3, Width / 10)
Player 0: (margin, margin) — bottom-left area
Player 1: Mirror(Player0) = (Width-1-margin, Height-1-margin) — top-right area
```

Spawns are point-symmetric through the map center.

#### 5. Mine Placement

- Distance from spawn: 5–8 cells (randomized within range)
- Constrained to player's primary half: `x in [2, Width/2)`, `y in [2, Height/2)`
- Player 1's mine is the mirror of Player 0's mine
- Each player gets MinesPerPlayer mines

#### 6. Exclusion Zones

No obstacles may be placed within these radii:

| Zone | Radius | Purpose |
|------|--------|---------|
| Spawn | 4 cells | Room for Base (6×4) + buffer |
| Mine | 3 cells | Accessible mining area |
| Center | 2 cells | Open engagement area |

#### 7. Templates

**OpenField** — Low density, small scattered groves
- Grove size: 2–5 cells
- Growth probability: 0.80, falloff: 0.10
- Grove spacing: 0 (no buffer)
- Result: Mostly open with small obstacles. Favors ranged units and kiting.

**Forest** — Medium-high density, large organic groves
- Grove size: 15–40 cells
- Growth probability: 0.90, falloff: 0.015
- Grove spacing: 3 (buffer between groves)
- Result: Thick forests with clearings. Creates chokepoints and ambush positions.

**Maze** — Cellular automata cave system
- Algorithm: Random fill → 5 smoothing passes (4-5 rule)
- Density: ObstacleDensity clamped to 0.20–0.55
- Smoothing: Cell is wall if ≥5 of 9 neighbors (including self) are walls
- Symmetry applied before grove grouping
- Result: Cave-like corridors. Forces narrow engagements, limits kiting.

#### 8. Grove Growth Algorithm (OpenField/Forest)

```
1. Pick random seed cell (not in exclusion zone, not in existing grove)
2. BFS expansion from seed
3. For each neighbor:
   probability = baseProbability - (distance from seed) × falloff
   If random() < probability → add to grove
4. Stop when grove reaches max size or probability drops to 0
5. Apply grove spacing buffer (Forest only)
6. Mark all grove cells as blocked (not walkable, not buildable)
```

#### 9. Symmetry Application

**Mirror (180° Point Reflection)**:
- Generate obstacles only in the primary half (Player 0's side)
- Mirror each blocked cell to the secondary half: `(W-1-x, H-1-y)`
- Both halves are identical — perfectly fair

#### 10. Connectivity Validation

After obstacle generation, BFS validates:
- Player 0 spawn → Player 0 mine(s)
- Player 1 spawn → Player 1 mine(s)
- Player 0 spawn → Player 1 spawn (both players can reach each other)
- All spawns → map center

If any check fails, the entire map is discarded and regenerated with the next
RNG state. After 50 failures, an open map (no obstacles) is returned as fallback.

### States and Transitions

Procedural Map Gen runs once per round as a preprocessing step. It has no runtime
state — after generation, the output (GameGrid + positions) is consumed by the
game and the generator is not called again until the next round.

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Map System** | Writes blocked cells to GameGrid | Grid initialization |
| **Match Flow** | Called during round setup, provides spawn/mine positions | Map generation trigger |
| **Unit Lifecycle** | Spawn positions used for starting unit placement | Position data |
| **Economy** | Mine positions determine gather distances | Economic geography |
| **Pathfinding** | Generated grid determines all path calculations | Terrain layout |
| **Sim Parity** | Both engines must generate identical maps for same seed | Deterministic generation |
| **Spectator UI** | Groves provide visual data for tree rendering | Presentation |

#### Ownership Boundary

Procedural Map Gen owns:
- Map generation algorithm (all three templates)
- Spawn and mine position calculation
- Exclusion zone enforcement
- Connectivity validation
- Retry/fallback logic
- Grove data for visual rendering

Procedural Map Gen does NOT own:
- The grid data structure (Map System)
- When generation runs (Match Flow)
- How the map looks visually (Presentation layer)
- Hand-crafted maps (loaded directly, bypassing this system)

## Formulas

### Spawn Margin

```
margin = max(3, Width / 10)
```

### Mine Distance from Spawn

```
distance = random(minDistance, maxDistance)
minDistance = 5, maxDistance = 8
position constrained to: x in [2, Width/2), y in [2, Height/2)
```

### Mirror Formula

```
Mirror(x, y) = (Width - 1 - x, Height - 1 - y)
MirrorUnit(x, y, sizeX, sizeY) = (Width - sizeX - x, Height - 1 + sizeY - y)
```

### Grove Growth Probability

```
P(cell) = baseProbability - distance(cell, seed) × falloff
Cell added if random() < P(cell) and P(cell) > 0
```

### Cellular Automata Smoothing (Maze)

```
For each cell, count walls in 3×3 neighborhood (including self):
  If wallCount >= 5 → cell becomes wall
  If wallCount < 5  → cell becomes floor
Repeat 5 times
```

## Edge Cases

#### 1. Map too small for Base placement
Minimum 15×15 prevents this. A 6×4 Base with 4-cell spawn exclusion fits
comfortably. Maps below 15×15 are rejected at config validation.

#### 2. ObstacleDensity too high — no valid map possible
After 50 retries, fallback to open map (zero obstacles). Logged as a warning.
The Maze template clamps density to 0.20–0.55 to reduce this risk.

#### 3. Mine placement fails (no valid position in range)
Retry with new RNG state. If mine placement fails on all retries, fallback to
placing mines at fixed offset from spawn.

#### 4. Asymmetric map due to odd dimensions
Mirror formula `(W-1-x, H-1-y)` works correctly for both odd and even dimensions.
With odd width, the center column maps to itself.

#### 5. Two groves merge into one large obstacle
OpenField has no grove spacing, so merging is possible. Forest uses 3-cell spacing
to prevent this. Connectivity validation catches any problematic merges.

#### 6. Same seed produces different map on different platforms
Both engines must use the same PRNG implementation. If System.Random differs
between .NET versions, maps diverge. The generator should use a shared,
deterministic PRNG from AgentSDK.

#### 7. Hand-crafted map loaded instead of procedural
Match Flow decides whether to generate or load a map. When loading, this system
is bypassed entirely. Loaded maps must still satisfy connectivity validation.

## Dependencies

### Upstream

| System | What It Provides |
|--------|-----------------|
| **Map System** | GameGrid API for writing blocked cells |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Match Flow** | Hard | Generated map for round setup |
| **Pathfinding** | Indirect | Terrain layout affects all paths |
| **Economy** | Indirect | Mine distances affect income rate |
| **Sim Parity** | Hard | Identical maps from same seed in both engines |

## Tuning Knobs

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| **ObstacleDensity** | Template-dependent | 0.0–0.55 | Empty maps, no terrain variety | Impassable maps, connectivity failures | Strategic variety, path complexity |
| **SpawnExclusion** | 4 | 2–8 | Buildings blocked near spawn | Large safe zone, less variety | Early game fairness |
| **MineExclusion** | 3 | 2–6 | Mining blocked | Large clear mining area | Resource accessibility |
| **CenterExclusion** | 2 | 0–4 | Center may be blocked | Large open center | Mid-map engagement |
| **MineMinDistance** | 5 | 3–10 | Mine trivially close | Long gather trips | Economic pacing |
| **MineMaxDistance** | 8 | 5–15 | Narrow distance range | Very far mines possible | Economic variance |
| **MaxRetries** | 50 | 10–200 | Quick fallback to open map | Slow generation on hard configs | Generation reliability vs speed |
| **Seed** | 42 | Any int | — | — | Map content (fully deterministic) |

### Template Knobs

| Template | baseProbability | falloff | minGrove | maxGrove | spacing |
|----------|----------------|---------|----------|----------|---------|
| OpenField | 0.80 | 0.10 | 2 | 5 | 0 |
| Forest | 0.90 | 0.015 | 15 | 40 | 3 |
| Maze | N/A (cellular automata) | N/A | N/A | N/A | N/A |

## Acceptance Criteria

#### Fairness
- [ ] Spawn positions are point-symmetric for all map sizes
- [ ] Mine positions are point-symmetric for all map sizes
- [ ] Shortest path spawn→mine is equal (±1 cell) for both players
- [ ] Both players can reach each other, their mines, and map center

#### Determinism
- [ ] Same seed + same config → identical map, every time
- [ ] SimGame and Unity produce identical maps for the same seed
- [ ] PRNG implementation is shared (not platform-dependent)

#### Templates
- [ ] OpenField produces mostly open maps with small scattered obstacles
- [ ] Forest produces thick forests with clearings and chokepoints
- [ ] Maze produces cave-like corridors with narrow passages
- [ ] All three templates pass connectivity validation

#### Robustness
- [ ] Connectivity validation catches all disconnected maps
- [ ] Retry mechanism produces valid maps within 50 attempts for standard configs
- [ ] Fallback (open map) is applied when retries exhaust
- [ ] Minimum map size (15×15) is enforced

#### Performance
- [ ] Map generation completes in under 100ms for 30×30 maps
- [ ] Map generation completes in under 500ms for 100×100 maps
- [ ] Connectivity validation (BFS) is not a bottleneck