# Combat

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Strategic Depth (R-P-S triangle), Spectator Appeal (battles)

## Overview

Combat is the core spectacle of Agents of Empires — the system that resolves armed
conflict between units. When an agent issues an Attack command, the attacking unit
approaches its target, enters range, and deals continuous damage per tick until the
target dies or the attacker is retargeted. Damage is modified by the
rock-paper-scissors triangle (Warrior > Archer > Lancer > Warrior), making army
composition the central strategic decision.

Combat operates within the Tick Engine's Phase 2 (Advance Tasks). It reads damage
values and multipliers from Balance Data, uses Pathfinding for approach movement,
and triggers Unit Lifecycle death when targets reach 0 HP. Three unit types
participate in combat: Warrior (melee), Archer (ranged), and Lancer (cavalry).

## Player Fantasy

**"My army composition counters theirs — adaptation wins wars."**

The R-P-S triangle creates a strategic puzzle: no single unit dominates. Massing
archers loses to warriors rushing in; warriors crumble to lancers flanking from
range; lancers die to focused archer fire. The student who reads the battlefield
and adapts their composition wins. This is the heart of **Strategic Depth** — and
the main reason spectators watch.

## Detailed Design

### Core Rules

#### 1. Combat Units

Only three unit types can attack:

| Unit | Role | Attack Range | Base Damage | Speed Multi |
|------|------|-------------|-------------|-------------|
| WARRIOR | Melee | 1.0 | 50 | 2.1 |
| ARCHER | Ranged | 9.0 | 38 | 3.0 |
| LANCER | Cavalry | 2.5 | 42 | 3.45 |

All other units (Pawn, Monk, buildings, Mine) cannot attack.

#### 2. Attack Flow

Each tick during Phase 2, an attacking unit follows this priority:

1. **Is target alive?** If no → retarget (find closest enemy in range) or go IDLE
2. **Is target in range?** If yes → deal damage
3. **Not in range?** → move closer along path (uses Pathfinding)

#### 3. Range Check

A unit is "in range" when:
```
Distance(attacker.Position, target.Position) <= EffectiveRange(attacker, target)
```

EffectiveRange accounts for the target's footprint size (see Formulas).

#### 4. Damage Application

Damage is dealt continuously every tick while the attacker is in range:
```
DamagePerTick = BASE_DAMAGE[attacker] × ScalarDamage
              × DamageMultiplier(attacker, target) × TickDuration
target.Health -= DamagePerTick
```

Damage is applied during Phase 2. The target's health may go below 0 within
Phase 2, but the unit is not removed until Phase 4.

#### 5. Rock-Paper-Scissors Multipliers

| Attacker → Defender | Multiplier | Meaning |
|---------------------|-----------|---------|
| Warrior → Archer | 1.25 | Warrior strong vs Archer |
| Archer → Warrior | 0.75 | Archer weak vs Warrior |
| Archer → Lancer | 1.25 | Archer strong vs Lancer |
| Lancer → Archer | 0.75 | Lancer weak vs Archer |
| Lancer → Warrior | 1.25 | Lancer strong vs Warrior |
| Warrior → Lancer | 0.75 | Warrior weak vs Lancer |
| All other matchups | 1.0 | Neutral (vs buildings, monks, same type, etc.) |

#### 6. Retargeting

When an attacker's target dies:
1. Search for the closest enemy unit within the attacker's attack range
2. If found → switch target, continue attacking
3. If none found → go IDLE

Retargeting happens within the same tick's Phase 2 — no tick is wasted.

#### 7. Approach Movement

When out of range, the attacker moves toward the target using the same movement
system as MOVE actions:
- Path computed via `FindPathToUnit(start, target.UnitType, target.Position)`
- Movement consumes MoveAccumulator per tick
- Collision avoidance applies (wait → detour → repath)
- Once in range, movement stops and damage begins

### States and Transitions

An attacking unit cycles through these states within the ATTACK action:

| State | Behavior | Transition |
|-------|----------|------------|
| **In Range** | Deal DamagePerTick to target each tick | Target dies → Retarget |
| **Approaching** | Move along path toward target | Enters range → In Range |
| **Retargeting** | Search for nearest enemy in range | Found → In Range/Approaching; None → IDLE |

These are not separate CurrentAction values — the unit's CurrentAction remains
ATTACK throughout. The sub-state is implicit from whether the unit has a path and
whether it's in range.

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Tick Engine** | Phase 2 advances ATTACK tasks | Tick-by-tick damage and movement |
| **Balance Data** | BASE_DAMAGE, ATTACK_RANGE, UNIT_SIZE, DamageMultiplier | Damage calculation parameters |
| **Pathfinding** | FindPathToUnit for approach, repath on collision | Path computation |
| **Unit Lifecycle** | Reads target existence; damage reduces Health toward death | Health mutation, death trigger |
| **Command Validation** | Validates Attack commands (CAN_ATTACK, target ownership) | Command filtering |
| **Map System** | Reads grid for approach movement | Walkability |
| **Micro-interactions** | Combat events trigger hit sparks, projectiles, death FX | Visual feedback (Tier 4) |
| **Audio** | Combat events trigger attack/hit/death sounds | Audio feedback (Tier 4) |

#### Ownership Boundary

Combat owns:
- Damage calculation (per-tick DPS with multipliers)
- Range checking (effective range with footprint adjustment)
- Retargeting logic (closest enemy search)
- Attack approach movement decisions

Combat does NOT own:
- Movement mechanics (Tick Engine / Pathfinding)
- Unit death (Unit Lifecycle — Phase 4)
- Attack command validation (Command Validation)
- Balance values (Balance Data)

## Formulas

### Damage Per Tick

```
DamagePerTick = BASE_DAMAGE[attacker.UnitType]
              × ScalarDamage
              × DamageMultiplier(attacker.UnitType, target.UnitType)
              × TickDuration
```

| Variable | Source | Example (GameSpeed 20) |
|----------|--------|----------------------|
| BASE_DAMAGE[WARRIOR] | Balance Data | 50 |
| ScalarDamage | DerivedGameConstants | 20 |
| DamageMultiplier(WARRIOR, ARCHER) | Balance Data | 1.25 |
| TickDuration | Tick Engine | 0.05 |
| **Result** | | **62.5 HP/tick = 1,250 DPS** |

### Effective Attack Range

```
EffectiveRange = ATTACK_RANGE[attacker.UnitType]
               + max(UNIT_SIZE[target.UnitType].X, UNIT_SIZE[target.UnitType].Y) / 2.0f
```

Examples:
- Warrior vs Pawn (1×1): 1.0 + 0.5 = 1.5
- Archer vs Barracks (3×3): 9.0 + 1.5 = 10.5
- Archer vs Base (6×4): 9.0 + 3.0 = 12.0
- Lancer vs Warrior (1×1): 2.5 + 0.5 = 3.0

### Time to Kill (1v1, no healing)

```
TTK = HEALTH[defender] / (BASE_DAMAGE[attacker] × ScalarDamage × DamageMultiplier)
```

At GameSpeed 20:

| Attacker | Defender | DPS | Defender HP | TTK |
|----------|----------|-----|------------|-----|
| Warrior | Archer | 1,250 | 600 | 0.48s |
| Warrior | Lancer | 750 | 900 | 1.20s |
| Warrior | Warrior | 1,000 | 1,500 | 1.50s |
| Archer | Lancer | 950 | 900 | 0.95s |
| Archer | Warrior | 570 | 1,500 | 2.63s |
| Archer | Archer | 760 | 600 | 0.79s |
| Lancer | Warrior | 1,050 | 1,500 | 1.43s |
| Lancer | Archer | 630 | 600 | 0.95s |
| Lancer | Lancer | 840 | 900 | 1.07s |

### Cost Efficiency (gold per kill in favorable matchup)

```
CostEfficiency = attacker.COST / TTK_favorable vs TTK_unfavorable
```

Lower cost + faster TTK in favorable = more efficient.

## Edge Cases

#### 1. Multiple attackers on one target
All attackers deal damage independently during Phase 2. Damage stacks — there is
no damage cap per tick. The target may go far below 0 HP.

#### 2. Target dies mid-Phase 2 from another attacker
The target's Health is below 0 but it's not removed until Phase 4. Subsequent
attackers in Phase 2 still deal damage (overkill). This is wasted damage but does
not cause errors.

#### 3. Attacker killed while approaching
Attacker is removed in Phase 4. Its ATTACK action simply ceases — no special
handling needed.

#### 4. Target moves out of range between ticks
On the next Phase 2, the range check fails. The attacker begins approach movement.
If the target is faster, a chase ensues — the attacker may never catch up (e.g.,
Warrior chasing Archer).

#### 5. Attacker ordered to attack own unit
Rejected by Command Validation (`NotOwned` — target must not be owned by
commanding agent). Never reaches Combat.

#### 6. Attacker ordered to attack a Mine
Rejected by Command Validation (`InvalidTarget` — mines cannot be attacked). Never
reaches Combat.

#### 7. Retarget finds no enemies
Attacker goes IDLE. It does not automatically search for distant enemies — only
enemies currently within attack range are considered for retarget.

#### 8. Two units attack each other simultaneously
Both deal damage during the same Phase 2. If both die, both are removed in
Phase 4. This is a valid mutual kill.

#### 9. Archer attacks from maximum range while target approaches
Archer deals damage each tick while the target closes distance. This is kiting — a
valid and important tactic. The archer's high speed (3.0) relative to warriors
(2.1) makes kiting viable.

## Dependencies

### Upstream

| System | What It Provides |
|--------|-----------------|
| **Unit Lifecycle** | Units to damage, target existence checks |
| **Command Validation** | Validated Attack commands |
| **Pathfinding** | Approach paths |
| **Balance Data** | BASE_DAMAGE, ATTACK_RANGE, UNIT_SIZE, DamageMultiplier |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Unit Lifecycle** | Hard | Health reduction triggers death |
| **Match Flow** | Hard | Combat outcomes determine round winner |
| **Micro-interactions** | Soft | Attack/hit/death events for VFX |
| **Audio** | Soft | Attack/hit/death events for SFX |
| **Analytics** | Soft | Damage dealt, kills, unit matchup data |

## Tuning Knobs

All combat tuning is done through Balance Data. See Balance Data GDD for:
- BASE_DAMAGE per unit type
- ATTACK_RANGE per unit type
- R-P-S multipliers (strong = 1.25, weak = 0.75)
- HEALTH per unit type (determines TTK)
- SPEED_MULTIPLIER per unit type (determines kiting viability)

### Key Balance Relationships

| Relationship | Current State | Design Intent |
|-------------|--------------|---------------|
| Warrior vs Archer TTK | 0.48s (warrior wins fast) | Warriors should close and kill archers quickly |
| Archer vs Warrior TTK | 2.63s (archer loses slow) | Archers have time to kite but lose if caught |
| Lancer speed vs Archer speed | 3.45 vs 3.0 | Lancers can close on archers but not instantly |
| Warrior speed vs Archer speed | 2.1 vs 3.0 | Archers can kite warriors indefinitely on open ground |

## Acceptance Criteria

#### Damage
- [ ] DamagePerTick matches the formula for all attacker/defender combinations
- [ ] R-P-S multipliers apply correctly (1.25 for strong, 0.75 for weak, 1.0 for
  neutral)
- [ ] Damage is applied every tick while in range (not once per attack animation)
- [ ] Overkill damage does not cause errors

#### Range
- [ ] EffectiveRange formula correctly accounts for target footprint size
- [ ] Archer range (9.0) allows attacking buildings from a significant distance
- [ ] Warrior range (1.0) requires adjacent positioning for 1×1 targets

#### Retargeting
- [ ] Attacker retargets to closest enemy in range when target dies
- [ ] Attacker goes IDLE if no enemy is in range after target dies
- [ ] Retarget happens within the same tick (no wasted tick)

#### Approach
- [ ] Out-of-range attackers path toward target using FindPathToUnit
- [ ] Collision avoidance works during approach (wait → detour → repath)
- [ ] Attacker begins dealing damage immediately upon entering range

#### Balance Verification
- [ ] In 1v1 same-cost investment: Warrior beats Archer, Archer beats Lancer,
  Lancer beats Warrior
- [ ] No unit type has >60% win rate across all matchups in automated testing
- [ ] Kiting is viable for archers vs warriors on open ground

#### Parity
- [ ] Identical attack sequences produce identical damage in SimGame and Unity
- [ ] Retarget selection is deterministic and identical in both engines
