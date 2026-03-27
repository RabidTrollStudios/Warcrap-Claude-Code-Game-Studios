# Economy

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Strategic Depth (invest vs spend trade-offs)

## Overview

The Economy system manages gold — the single resource in Agents of Empires. Gold
flows from mines through pawns into the player's pool, and out of the pool to fund
buildings and units. Every strategic decision has an economic cost: more pawns means
faster income but delayed military; rushing combat units means starving future
production.

The economy is intentionally simple — one resource, one gatherer type, finite
mines. Complexity comes from timing and allocation decisions, not from resource
variety. The system operates within the Tick Engine's phases: gold is deducted
during Phase 1 (command validation), deposited during Phase 2 (gather completion),
and refunded before Phase 2 if commands fail.

## Player Fantasy

**"Do I invest in my economy or build an army now?"**

The tension between economic investment (more pawns, more bases) and military
spending (units, production buildings) is the core strategic axis. Agents that boom
too hard get rushed; agents that rush too hard run out of steam. Reading the
opponent's economy and responding is what separates good agents from great ones.

## Detailed Design

### Core Rules

#### 1. Gold Pool

Each agent has a single gold pool (integer). Gold can never go negative — Command
Validation rejects commands when `Gold < COST[unitType]`.

- **Starting gold**: Configurable per match (default 1,000)
- **Maximum gold**: Unbounded (no cap)
- **Gold is visible**: Both agents can query their own gold and the enemy's gold
  via IGameState (`MyGold`, `EnemyGold`)

#### 2. Gold Sources

| Source | When | Amount |
|--------|------|--------|
| **Starting gold** | Round start | SimConfig.StartingGold (default 1,000) |
| **Mining deposit** | Pawn completes TO_BASE gather phase | Gold carried by pawn (up to MINING_CAPACITY) |
| **Command refund** | Phase 1, when a validated command's action fails | Full cost refunded |

There are no other gold sources — no passive income, no kill bounties, no interest.

#### 3. Gold Sinks

| Sink | When | Amount |
|------|------|--------|
| **Train command** | Phase 1 validation | COST[unitType] |
| **Build command** | Phase 1 validation | COST[unitType] |

Only Train and Build cost gold. Move, Attack, Gather, Repair, and Heal are free.

#### 4. Mining Cycle

The Gather action is a three-phase loop managed by the Tick Engine. The Economy
system owns the gold flow:

1. **TO_MINE**: Pawn walks to assigned mine (no gold flow)
2. **MINING**: Timer countdown. On completion, extract gold from mine:
   - `goldMined = min(MINING_CAPACITY, mine.Health)`
   - `mine.Health -= goldMined`
   - Gold stored on pawn (in MiningTimer field)
3. **TO_BASE**: Pawn walks to assigned base. On arrival:
   - `agent.Gold += goldMined`
   - Pawn cycles back to TO_MINE

#### 5. Mine Depletion

Mines have finite gold (stored as Health, default 10,000). Each mining trip
extracts up to MINING_CAPACITY (50) gold. When a mine reaches 0 health, it is
removed by Unit Lifecycle in Phase 4. Pawns assigned to a depleted mine go IDLE.

#### 6. Gold Visibility

Both agents have full information about gold:
- `MyGold`: Own gold pool
- `EnemyGold`: Opponent's gold pool
- Mine health is visible via `GetUnit(mineNbr).Health`

This is a deliberate design choice — economic scouting is free, strategic
adaptation is based on information, not guessing.

### States and Transitions

The Economy system has no state machine. Gold is a single integer per agent,
modified atomically during Phase 1 (deductions/refunds) and Phase 2 (deposits).

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Tick Engine** | Phase 1: gold deductions/refunds. Phase 2: gather deposits | Gold mutations within tick phases |
| **Command Validation** | Checks Gold >= COST before approving Build/Train | Gold sufficiency queries |
| **Balance Data** | COST per unit type, MINING_CAPACITY, MiningSpeed | Economic parameters |
| **Unit Lifecycle** | Mine removal when Health reaches 0; pawn death loses carried gold | Resource depletion, gold loss |
| **Production** | Train/Build commands consume gold | Gold spending |
| **Agent SDK** | Exposes MyGold, EnemyGold via IGameState | Gold visibility to agents |
| **Match Flow** | Gold used as tiebreaker in timeout scoring | Final gold comparison |
| **Map System** | Mine positions determine gather distances | Economic geography |

#### Ownership Boundary

Economy owns:
- Gold pool per agent (the integer value)
- Gold flow rules (when gold is added/removed)
- Mining yield calculation (how much gold per trip)
- Mine depletion tracking (via mine Health)

Economy does NOT own:
- Gather movement/timing (Tick Engine)
- Command validation gold checks (Command Validation)
- Mine placement (Procedural Map Gen / Match Flow)
- Unit costs (Balance Data)

## Formulas

### Mining Time

```
MiningTime = MINING_CAPACITY / MiningSpeed
MiningSpeed = GameSpeed × 20

At GameSpeed 20: MiningTime = 50 / 400 = 0.125 seconds
```

### Gold Extracted Per Trip

```
GoldMined = min(MINING_CAPACITY, mine.Health)
MINING_CAPACITY = 50 (Pawn only)
```

### Income Rate Per Pawn

```
TripTime = WalkToMine + MiningTime + WalkToBase
IncomePerPawn = MINING_CAPACITY / TripTime

At GameSpeed 20, with mine ~5 cells from base:
  WalkToMine ≈ 5 / (20 × 0.05 × 1.0) = 5.0 ticks = 0.25s
  MiningTime = 0.125s
  WalkToBase ≈ 0.25s
  TripTime ≈ 0.625s
  IncomePerPawn ≈ 50 / 0.625 = 80 gold/second
```

### Mine Lifetime

```
MineLifetime = StartingMineGold / (MINING_CAPACITY × PawnsGathering / TripTime)

With 2 pawns at GameSpeed 20:
  MineLifetime ≈ 10000 / (80 × 2) = 62.5 seconds
```

### Total Gold Available Per Match

```
TotalGold = StartingGold + (MinesPerPlayer × StartingMineGold)
Default: 1000 + (1 × 10000) = 11,000 gold per player
```

## Edge Cases

#### 1. Pawn dies while carrying gold
Gold stored on the pawn (MiningTimer field) is lost permanently. It is not dropped
or returned to the pool. This makes protecting gatherers strategically important.

#### 2. All mines depleted
No new gold enters the game. Players must fight with what they have. The match
becomes a pure attrition game. Timeout scoring (unit value + gold tiebreaker)
determines the winner if neither player is eliminated.

#### 3. Base destroyed while pawns gathering
Per Tick Engine GDD edge case #11: pawns hold their gold until a new base is
built. They can execute other commands while holding gold. Once a base is
available, the pawn completes its current task, walks to base, deposits, and
goes IDLE.

#### 4. Two pawns deposit gold on the same tick
Both deposits are processed during Phase 2. Gold additions are atomic per pawn —
no race condition. Order doesn't matter since addition is commutative.

#### 5. Agent has exactly enough gold for a command
The check is `Gold >= COST`, so exact equality passes. Gold is deducted to 0.

#### 6. Gold refund after failed Build on same tick as gold spend on Train
Both happen during Phase 1 in sort order. The Build refund increases the pool; the
Train deduction decreases it. Final gold reflects all Phase 1 operations in
deterministic order.

#### 7. Mine health is not an integer multiple of MINING_CAPACITY
The last trip extracts whatever remains: `min(50, mine.Health)`. If the mine has
30 gold left, the pawn carries 30. No gold is lost.

#### 8. Enemy gold is negative (impossible state)
Command Validation prevents this — it rejects commands when gold is insufficient.
If gold somehow goes negative, it indicates a parity bug.

## Dependencies

### Upstream

| System | What It Provides |
|--------|-----------------|
| **Unit Lifecycle** | Pawns to gather, mines to extract from, bases to deposit at |
| **Command Validation** | Gold sufficiency checks on Build/Train |
| **Pathfinding** | Gather walking routes (mine → base → mine) |
| **Balance Data** | COST, MINING_CAPACITY, MiningSpeed |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Production** | Hard | Gold pool funds all unit creation |
| **Match Flow** | Hard | Gold as timeout tiebreaker |
| **Agent SDK** | Hard | MyGold, EnemyGold queries |
| **Analytics** | Soft | Income rate, spending patterns |

## Tuning Knobs

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| **StartingGold** | 1,000 | 200–5,000 | Can't afford first building, stalled start | Skip early economy, rush immediately | Opening strategy diversity |
| **StartingMineGold** | 10,000 | 2,000–50,000 | Mines deplete fast, short games | Effectively infinite resources, no pressure | Game length, resource scarcity timing |
| **MINING_CAPACITY** | 50 | 10–200 | Many trips, slow income | Few trips, fast income, short games | Pawn value, gather micro-optimization |
| **MinesPerPlayer** | 1 | 1–4 | Single chokepoint, mine control is critical | Abundant income, less economic pressure | Map control value, economic strategy |

### Knob Interactions

- **StartingGold × COST[BASE]**: Starting gold must be enough to build at least
  a Barracks (400g) after setup. At 1000g starting, the agent has 500g after the
  starting Base is pre-placed (Base is free at round start).
- **StartingMineGold × MinesPerPlayer**: Total available gold determines maximum
  army size. At 11,000 total gold, an agent can build roughly 110 archers (at 80g
  each) if they spend everything on archers.
- **MINING_CAPACITY × MiningSpeed**: Together determine income rate. Both scale
  with GameSpeed, keeping the ratio constant across speed settings.

## Acceptance Criteria

#### Gold Flow
- [ ] Starting gold is correctly set from SimConfig at round start
- [ ] Gold is deducted on successful Build/Train validation in Phase 1
- [ ] Gold is refunded for failed commands before Phase 2
- [ ] Gold is deposited when pawns complete TO_BASE gather phase
- [ ] Gold never goes negative

#### Mining
- [ ] Pawns extract min(MINING_CAPACITY, mine.Health) gold per trip
- [ ] Mine health decreases by the extracted amount
- [ ] Mines are removed when health reaches 0
- [ ] Pawns go IDLE when assigned mine is depleted

#### Visibility
- [ ] MyGold and EnemyGold are accurate and available to agents every tick
- [ ] Mine health is visible via GetUnit()

#### Parity
- [ ] Gold values are identical in SimGame and Unity at every tick
- [ ] Mining extraction amounts are identical in both engines
- [ ] Gold is included in the Tick Engine state hash

#### Performance
- [ ] Gold operations (add, subtract, query) are O(1)
