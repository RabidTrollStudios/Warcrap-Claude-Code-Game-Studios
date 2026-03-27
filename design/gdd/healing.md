# Healing

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Strategic Depth (support unit decisions)

## Overview

The Healing system governs how Monks restore health to friendly units. Monks are
the only unit type that can heal, using a mana-gated flat HP restoration. Healing
creates a support role in army composition — agents must decide whether the cost of
a Monk (90g, no combat ability) is worth the sustained fighting power it provides.

Healing operates within the Tick Engine's Phase 2 (advance HEAL task) and Phase 3
(mana regeneration). It reads heal parameters from Balance Data and uses
Pathfinding for approach movement.

## Player Fantasy

**"My monks keep my army in the fight longer than theirs."**

Monks add a sustain dimension to combat. An army with monk support can survive
engagements that would otherwise be losses, but monks are fragile (400 HP) and
expensive relative to their combat contribution (zero damage). The decision to
invest in monks vs. more combat units is a meaningful strategic trade-off.

## Detailed Design

### Core Rules

#### 1. Healing Unit

Only MONK can heal. One heal per approach — the monk walks into range, heals once,
then can heal again after mana regen allows.

| Property | Value | Source |
|----------|-------|--------|
| HEAL_AMOUNT | 100 HP (flat) | Balance Data |
| MANA_COST | 10 | Balance Data |
| MAX_MANA | 100 | Balance Data |
| BASE_MANA_REGEN | 100/60 ≈ 1.667/s | Balance Data (scaled by GameSpeed) |
| HEAL_RANGE | 4.0 | Balance Data |

#### 2. Heal Flow

Each tick during Phase 2, a healing monk follows this priority:

1. **Is target alive?** If no → go IDLE
2. **Is target eligible?** If no (healed above threshold since command) → go IDLE
3. **Is target in range?** If yes → check mana → heal
4. **Not in range?** → move closer along path

#### 3. Eligibility

A unit can be healed when:
```
target.Health <= HEALTH[target.UnitType] - HEAL_AMOUNT
```

The target must be missing at least HEAL_AMOUNT (100) HP. This prevents wasting
heals on nearly-full units. Only mobile units can be healed — buildings cannot.

#### 4. Heal Application

When in range with sufficient mana:
```
ActualHeal = min(HEAL_AMOUNT, HEALTH[target.UnitType] - target.Health)
target.Health += ActualHeal
monk.Mana -= MANA_COST
```

Health is capped at max — never overheals.

#### 5. Mana System

- Mana regenerates during Phase 3 of every tick (all units with mana pools)
- Regen formula: `mana = min(mana + ManaRegen × TickDuration, MAX_MANA)`
- ManaRegen = BASE_MANA_REGEN × GameSpeed
- At GameSpeed 20: 33.3 mana/second → full pool (100) in ~3 seconds
- Monks start each round with full mana (100)

#### 6. Heal Command

`Heal(monkNbr, targetNbr)` — assigns the monk to heal a specific friendly unit.
The monk approaches and heals once per mana charge. The monk does NOT automatically
continue healing — the agent must re-issue Heal commands to sustain healing.

### States and Transitions

A healing monk cycles through states within the HEAL action:

| State | Behavior | Transition |
|-------|----------|------------|
| **Approaching** | Move along path toward target | In range → Check Mana |
| **In Range** | Check mana and eligibility | Sufficient mana + eligible → Heal; else → IDLE |
| **Healing** | Apply heal, deduct mana | Complete → IDLE |

After a heal completes, the monk goes IDLE. The agent must issue a new Heal
command for the next heal cycle. This gives agents fine-grained control over
monk behavior.

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Tick Engine** | Phase 2: advance HEAL task. Phase 3: mana regen | Heal within tick phases |
| **Balance Data** | HEAL_AMOUNT, MANA_COST, MAX_MANA, BASE_MANA_REGEN, HEAL_RANGE | Heal parameters |
| **Pathfinding** | FindPathToUnit for approach | Approach paths |
| **Unit Lifecycle** | Reads target existence; restores Health | Health mutation |
| **Command Validation** | Validates Heal commands (CAN_HEAL, mana, eligibility) | Command filtering |
| **Combat** | Healing counteracts combat damage, extending unit lifetime | Strategic interaction |
| **Micro-interactions** | Heal events trigger heal beam VFX, particles | Visual feedback (Tier 4) |

#### Ownership Boundary

Healing owns:
- Heal amount calculation and application
- Mana cost deduction per heal
- Mana regeneration (Phase 3)
- Heal range checking
- Heal eligibility checking

Healing does NOT own:
- Monk movement (Tick Engine / Pathfinding)
- Mana pool initialization (Unit Lifecycle)
- Heal command validation (Command Validation)
- Balance values (Balance Data)

## Formulas

### Heal Amount

```
ActualHeal = min(HEAL_AMOUNT, HEALTH[target.UnitType] - target.Health)
HEAL_AMOUNT = 100 (flat, all targets)
```

### Mana Regeneration

```
ManaRegen = BASE_MANA_REGEN × GameSpeed
         = (100 / 60) × GameSpeed

Per tick: mana += ManaRegen × TickDuration
Capped at MAX_MANA (100)
```

At GameSpeed 20:
- ManaRegen = 33.33/s
- Per tick: +1.667 mana
- Full pool recharge: ~3.0 seconds
- Heals per full pool: 10 (at 10 mana each)
- Total healing per pool: 1,000 HP

### Effective Heal Range

```
EffectiveHealRange = HEAL_RANGE + max(UNIT_SIZE[target].X, UNIT_SIZE[target].Y) / 2.0f
```

For 1×1 targets: 4.0 + 0.5 = 4.5

### Sustain Rate (continuous healing)

```
HealsPerSecond = ManaRegen / MANA_COST
HealPerSecond = HealsPerSecond × HEAL_AMOUNT

At GameSpeed 20:
  HealsPerSecond = 33.33 / 10 = 3.33 heals/s
  HealPerSecond = 3.33 × 100 = 333 HP/s
```

### Heal vs Damage Comparison

At GameSpeed 20, one Monk sustains 333 HP/s. Combat DPS:
- Warrior: 1,000 DPS (neutral) → Monk heals 33% of damage
- Archer: 760 DPS (neutral) → Monk heals 44% of damage
- Lancer: 840 DPS (neutral) → Monk heals 40% of damage

A single Monk cannot outheal any single attacker. Multiple Monks can sustain
against single attackers but not against focused fire.

## Edge Cases

#### 1. Heal target dies before monk arrives
Monk discovers target is gone on next Phase 2 advance. Goes IDLE.

#### 2. Target healed above threshold by another monk
Second monk's eligibility check fails — target now above
`HEALTH - HEAL_AMOUNT` threshold. Monk goes IDLE.

#### 3. Monk has exactly 10 mana
Passes — check is `Mana >= MANA_COST`. Mana drops to 0 after heal.

#### 4. Monk ordered to heal a building
Rejected by Command Validation — target must be mobile
(`CAN_MOVE[target.UnitType] == true`).

#### 5. Monk ordered to heal an enemy unit
Rejected by Command Validation — target must be owned by commanding agent.

#### 6. Monk ordered to heal itself
Rejected by Command Validation — if monk is at full health, eligibility fails.
If monk is damaged below threshold, self-heal is valid (monk is a mobile owned
unit).

#### 7. Heal on a Pawn with 101 HP (out of 200 max)
Eligible: 101 <= 200 - 100 = 100? No — 101 > 100. Not eligible. Pawn must be
at 100 HP or below.

#### 8. Multiple monks heal the same target in one tick
Both heals apply independently during Phase 2. Target receives up to 200 HP
(2 × 100). Health capped at max.

## Dependencies

### Upstream

| System | What It Provides |
|--------|-----------------|
| **Unit Lifecycle** | Monks and targets, health state |
| **Command Validation** | Validated Heal commands |
| **Pathfinding** | Approach paths |
| **Balance Data** | HEAL_AMOUNT, MANA_COST, MAX_MANA, BASE_MANA_REGEN, HEAL_RANGE |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Combat** | Indirect | Healing extends unit lifetime, changes combat outcomes |
| **Micro-interactions** | Soft | Heal events for VFX |
| **Audio** | Soft | Heal events for SFX |

## Tuning Knobs

All healing tuning is in Balance Data. See Balance Data GDD section 6 and
Tuning Knobs for full ranges.

### Key Balance Relationships

| Relationship | Current | Design Intent |
|-------------|---------|---------------|
| Monk cost (90g) vs combat unit cost (70-100g) | Comparable | Monks are a real investment, not free support |
| Heal/s (333) vs lowest DPS (570, Archer vs Warrior) | ~58% of DPS | One monk meaningfully extends life but can't outheal |
| Mana pool (100) / MANA_COST (10) = 10 heals | 1,000 HP burst | Enough to turn a close fight, not enough for sustained war |
| Monk HP (400) vs Archer damage | Dies in ~0.53s | Monks are fragile — protect them or lose them |

## Acceptance Criteria

#### Healing
- [ ] Heal applies flat HEAL_AMOUNT (100) HP, capped at max health
- [ ] Eligibility check: target.Health <= MaxHealth - HEAL_AMOUNT
- [ ] Only mobile friendly units can be healed (no buildings, no enemies)
- [ ] Mana deducted by MANA_COST (10) per heal

#### Mana
- [ ] Mana regenerates during Phase 3 at ManaRegen × TickDuration per tick
- [ ] Mana is capped at MAX_MANA (100)
- [ ] Monks start each round with full mana
- [ ] Mana regen scales correctly with GameSpeed

#### Sustain Balance
- [ ] A single Monk cannot keep any unit alive against a single attacker
  indefinitely (heal rate < damage rate for all combat units)
- [ ] Two Monks can meaningfully extend a Warrior's life against one attacker

#### Parity
- [ ] Heal amounts and mana values are identical in SimGame and Unity at every tick
- [ ] Mana regen produces identical mana values in both engines