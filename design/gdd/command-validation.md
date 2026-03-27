# Command Validation

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Fair Competition (rules enforcement)

## Overview

Command Validation is the rules enforcement layer that sits between agent commands
and game execution. Every command issued by an agent during Phase 5 is validated
during Phase 1 of the next tick before it can affect game state. It checks
ownership, capability flags, resource availability, dependency prerequisites, and
spatial constraints. Invalid commands are rejected with a reason code, reported via
`GetFailedCommands()`, and gold is refunded if it was deducted.

The validator is shared code (`AgentSDK/CommandValidator.cs`) used identically by
SimGame and Unity. It is stateless — it reads current game state and Balance Data
to produce a pass/fail verdict for each command.

## Player Fantasy

**"The rules are clear and consistent — if my command fails, I know why."**

Agents should never be surprised by a command rejection. Every rule is documented,
every failure includes a reason code, and the rules are identical in both engines.
This serves **Fair Competition**: no agent can gain advantage through invalid
commands, and no agent is penalized by opaque rejection logic.

## Detailed Design

### Core Rules

#### 1. Validation Timing

- Commands are queued by agents during Phase 5 (Agent Update)
- **Queue-time validation**: `IAgentActions` methods perform basic checks and
  return a `CommandResult` immediately (ownership, unit exists, basic capability)
- **Processing-time validation**: Full validation occurs during Phase 1 of the
  next tick, in deterministic sort order. This is when gold is deducted, spatial
  checks run, and dependency checks execute
- Commands that pass queue-time but fail processing-time validation appear in
  `GetFailedCommands()` on the next tick

#### 2. Validation Rules Per Command

**Move(unitNbr, target)**
- Unit exists and is owned by the commanding agent
- `CAN_MOVE[unit.UnitType] == true`
- Target position is within map bounds
- Target position is walkable

**Attack(unitNbr, targetNbr)**
- Unit exists and is owned by the commanding agent
- `CAN_ATTACK[unit.UnitType] == true`
- Target exists and is NOT owned by the commanding agent
- Target is not a Mine (mines cannot be attacked)

**Train(buildingNbr, unitType)**
- Building exists and is owned by the commanding agent
- `CAN_TRAIN[building.UnitType] == true`
- Building's `IsBuilt == true`
- `unitType` is in `TRAINS[building.UnitType]`
- Agent has sufficient gold: `Gold >= COST[unitType]`
- Building is currently IDLE (not already training)

**Build(unitNbr, target, unitType)**
- Unit exists and is owned by the commanding agent
- `CAN_BUILD[unit.UnitType] == true`
- `unitType` is in `BUILDS[unit.UnitType]`
- Agent has sufficient gold: `Gold >= COST[unitType]`
- All dependencies met: every type in `DEPENDENCY[unitType]` has at least one
  built instance owned by the agent
- Target area is buildable: `IsAreaBuildable(unitType, target)` (or
  `IsBoundedAreaBuildable` depending on context)

**Gather(pawnNbr, mineNbr, baseNbr)**
- Pawn exists and is owned by the commanding agent
- `CAN_GATHER[pawn.UnitType] == true`
- Mine exists and is of type MINE
- Base exists, is owned by the commanding agent, and is of type BASE
- Base `IsBuilt == true`

**Repair(pawnNbr, buildingNbr)**
- Pawn exists and is owned by the commanding agent
- `CAN_BUILD[pawn.UnitType] == true` (repair uses build capability)
- Building exists, is owned by the commanding agent
- Building is stationary (`CAN_MOVE[building.UnitType] == false`)
- Building `IsBuilt == true`
- Building health is below maximum

**Heal(monkNbr, targetNbr)**
- Monk exists and is owned by the commanding agent
- `CAN_HEAL[monk.UnitType] == true`
- Monk has sufficient mana: `Mana >= MANA_COST`
- Target exists, is owned by the commanding agent
- Target is mobile: `CAN_MOVE[target.UnitType] == true`
- Target health eligible: `target.Health <= HEALTH[target.UnitType] - HEAL_AMOUNT`

#### 3. Gold Handling

- Gold is deducted immediately when a Build or Train command passes validation in
  Phase 1
- If the command's resulting action fails (spatial conflict, spawn blocked), gold
  is refunded before Phase 2 begins
- Gold is never deducted for Move, Attack, Gather, Repair, or Heal commands

#### 4. CommandResult Codes

| Code | Meaning |
|------|---------|
| `Success` | Command accepted |
| `InvalidUnit` | Unit does not exist |
| `NotOwned` | Unit is not owned by this agent |
| `IncapableUnit` | Unit lacks the required capability flag |
| `InsufficientGold` | Not enough gold for this action |
| `DependencyNotMet` | Required prerequisite building not built |
| `InvalidTarget` | Target does not exist or is invalid |
| `AreaNotBuildable` | Build site is not available |
| `BuildingNotBuilt` | Building is still under construction |
| `BuildingBusy` | Building is already training a unit |
| `TargetNotEligible` | Heal target above health threshold or repair target at full HP |
| `InsufficientMana` | Monk does not have enough mana |
| `OutOfBounds` | Target position is outside map |

### States and Transitions

Command Validation is stateless. It is a pure function: game state + command in →
pass/fail + reason out.

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Tick Engine** | Called during Phase 1 for each command in sort order | Command → validated/rejected |
| **Balance Data** | Reads COST, CAN_*, DEPENDENCY, BUILDS, TRAINS, HEALTH, HEAL_AMOUNT, MANA_COST | Rule parameters |
| **Map System** | Reads buildability for Build commands | Spatial validation |
| **Unit Lifecycle** | Reads unit existence, ownership, health, IsBuilt, CurrentAction | Unit state checks |
| **Economy** | Deducts/refunds gold on Build/Train commands | Gold mutations |
| **Agent SDK** | Queue-time validation returns CommandResult; processing-time failures reported via GetFailedCommands() | Feedback to agents |

#### Ownership Boundary

Command Validation owns:
- All validation rules for all 7 command types
- CommandResult codes and failure reasons
- The failed commands list (populated during Phase 1, read by agents in Phase 5)

Command Validation does NOT own:
- Command sorting or one-per-unit enforcement (Tick Engine)
- Command execution (individual gameplay systems)
- Gold pool (Economy)

## Formulas

No formulas — Command Validation is boolean logic (pass/fail), not mathematical
computation. All threshold values (costs, health eligibility) are read from
Balance Data.

## Edge Cases

#### 1. Agent commands a unit that died this tick
The unit was removed in Phase 4 of the previous tick. The command fails with
`InvalidUnit` during Phase 1 and appears in `GetFailedCommands()`.

#### 2. Agent commands a unit that another command already moved
The one-command-per-unit-per-tick rule (Tick Engine) silently drops the second
command before validation runs. It does NOT appear in `GetFailedCommands()`.

#### 3. Build command with gold deducted, then area becomes blocked
Gold is deducted during Phase 1 validation. If the building placement fails (e.g.,
another build command in the same tick claimed overlapping cells earlier in sort
order), gold is refunded and the command appears in `GetFailedCommands()` with
`AreaNotBuildable`.

#### 4. Train command when building is already training
Fails with `BuildingBusy`. The agent should check
`GetUnit(buildingNbr).CurrentAction` before issuing Train.

#### 5. Heal command when monk's mana is exactly equal to MANA_COST
Passes — the check is `Mana >= MANA_COST`, not strictly greater than.

#### 6. Two agents issue Build on overlapping areas in the same tick
Processed in sort order (AgentNbr ASC). First agent's build succeeds and claims
the cells. Second agent's build fails with `AreaNotBuildable` and gold is refunded.

#### 7. Dependency check with unbuilt building
`DEPENDENCY[BARRACKS]` requires BASE. If the agent's only Base has
`IsBuilt = false`, the dependency is not met. Fails with `DependencyNotMet`.

## Dependencies

### Upstream

| System | What It Provides |
|--------|-----------------|
| **Tick Engine** | Phase 1 execution context, command sort order |
| **Balance Data** | All rule parameters (costs, flags, dependencies) |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **All gameplay systems** | Hard | Only validated commands reach execution |
| **Agent SDK** | Hard | CommandResult feedback and GetFailedCommands() |

## Tuning Knobs

Command Validation has no independent tuning knobs. All thresholds and rules are
derived from Balance Data constants. Changing a cost, capability flag, or
dependency in Balance Data automatically changes validation behavior.

## Acceptance Criteria

#### Rule Enforcement
- [ ] Every command type validates all rules listed in Core Rules section 2
- [ ] Invalid commands never reach gameplay execution
- [ ] Valid commands always reach gameplay execution

#### Gold Handling
- [ ] Gold is deducted on successful Build/Train validation
- [ ] Gold is refunded if the resulting action fails before Phase 2
- [ ] Gold is never deducted for Move, Attack, Gather, Repair, Heal

#### Feedback
- [ ] Queue-time validation returns correct CommandResult codes
- [ ] Processing-time failures appear in `GetFailedCommands()` on the next tick
- [ ] Silently dropped duplicate-unit commands do NOT appear in GetFailedCommands()
- [ ] Every failure includes the correct reason code

#### Parity
- [ ] Identical commands produce identical validation results in SimGame and Unity
- [ ] CommandValidator is shared code — no engine-specific validation logic

#### Performance
- [ ] Validating 100 commands per tick completes in under 1ms
