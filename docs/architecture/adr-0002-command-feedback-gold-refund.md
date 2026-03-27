# ADR-0002: Command Feedback & Gold Refund

## Status
Accepted

## Date
2026-03-26

## Context

### Problem Statement
Currently, agents have limited visibility into command outcomes. When an agent
issues a command that fails during Phase 1 processing (e.g., build site became
occupied, target died between queue and processing), the agent receives no
feedback. Additionally, gold deducted for Build/Train commands that subsequently
fail is never refunded — the agent loses gold with no explanation and no recourse.

This creates two problems:
1. **Silent failures**: Agents can't adapt to failed commands because they don't
   know which commands failed or why
2. **Unfair gold loss**: Gold lost to failed commands punishes agents for
   unavoidable timing conflicts, not strategic mistakes

### Constraints
- Must maintain determinism — feedback must be identical in SimGame and Unity
- Must not break existing agent DLLs (additive API changes only)
- Feedback must be available during the agent's next Update() call
- Gold refund must happen before Phase 2 to maintain consistent state

### Requirements
- Agents must know which commands failed and why (deferred feedback)
- Agents must receive immediate validation results at queue time (existing)
- Gold must be refunded for commands that pass validation but fail execution
- Both feedback mechanisms must be deterministic and parity-safe

## Decision

**Two-layer command feedback with automatic gold refund:**

1. **Immediate feedback** (existing, expanded): `IAgentActions` methods return
   `CommandResult` with expanded status codes at queue time (Phase 5). This
   catches basic errors (wrong unit, insufficient gold, missing capability).

2. **Deferred feedback** (new): `IGameState.GetFailedCommands()` returns a list
   of commands from the previous tick that passed queue-time validation but failed
   during Phase 1 processing. Each entry includes the command, unit, and failure
   reason.

3. **Gold refund** (new): Gold deducted during Phase 1 for Build/Train commands
   is automatically refunded if the resulting action fails before Phase 2 begins.

### Key Interfaces

```csharp
// Expanded CommandResult (existing enum, new values added)
public enum CommandResult
{
    Success,
    InvalidUnit,
    NotOwned,
    IncapableUnit,
    InsufficientGold,
    DependencyNotMet,
    InvalidTarget,
    AreaNotBuildable,
    BuildingNotBuilt,
    BuildingBusy,
    TargetNotEligible,
    InsufficientMana,
    OutOfBounds
}

// New: Failed command record
public struct FailedCommand
{
    public int UnitNbr;
    public CommandType Type;
    public CommandResult Reason;
    // Additional context fields as needed
}

// New method on IGameState
IReadOnlyList<FailedCommand> GetFailedCommands();
```

### Processing Flow

```
Phase 5 (tick N):
  Agent calls actions.Build(pawn, target, BARRACKS)
  → Queue-time validation: Success (pawn exists, gold sufficient, area looks OK)
  → CommandResult.Success returned to agent
  → Command queued for Phase 1

Phase 1 (tick N+1):
  Process commands in sort order
  → Reach the Build command
  → Full validation: area is now blocked (another build claimed it first in sort order)
  → Gold refund: agent.Gold += COST[BARRACKS]
  → Add to failed commands list: { UnitNbr=pawn, Type=Build, Reason=AreaNotBuildable }

Phase 5 (tick N+1):
  Agent calls state.GetFailedCommands()
  → Returns [{ UnitNbr=pawn, Type=Build, Reason=AreaNotBuildable }]
  → Agent adapts: tries a different build location
```

## Alternatives Considered

### Alternative 1: No Feedback, No Refund (Current Behavior)
- **Description**: Keep current behavior — silent failures, no gold refund
- **Pros**: Simplest implementation. No API changes.
- **Cons**: Agents can't adapt to failures. Gold loss is unfair. Students
  frustrated by invisible bugs.
- **Rejection Reason**: Violates Easy Onboarding and Fair Competition pillars.

### Alternative 2: Callback/Event System
- **Description**: Agent registers callbacks that fire when commands fail
- **Pros**: Real-time notification. Rich event data.
- **Cons**: Callbacks during Phase 1 could cause re-entrant command issues.
  Complex to make deterministic. Breaks the clean Phase 5 boundary.
- **Rejection Reason**: Too complex, risks determinism.

### Alternative 3: Deferred Feedback Only (No Immediate)
- **Description**: All feedback deferred to next tick via GetFailedCommands()
- **Pros**: Simple, clean separation
- **Cons**: Agents can't catch obvious errors (wrong unit ID) until next tick.
  Wastes a tick for errors that could be caught immediately.
- **Rejection Reason**: Immediate feedback for obvious errors is too useful to
  drop. The two-layer approach gives both.

## Consequences

### Positive
- Agents can detect and adapt to failed commands
- Gold is never lost to timing conflicts — only to strategic decisions
- Debugging is easier — students see exactly what failed and why
- Deterministic — failed command list is populated in sort order, identical in
  both engines

### Negative
- API surface grows (new method on IGameState, new FailedCommand struct)
- Per-tick memory allocation for the failed commands list (small — typically 0-5
  entries)
- Existing agents don't use GetFailedCommands() — benefit is only for new agents

### Risks
- **Duplicate-unit commands in failed list**: Silently dropped commands (one-per-
  unit rule) must NOT appear in GetFailedCommands(). Only processing-time failures
  count. **Mitigation**: Clear rule in Tick Engine GDD, tested explicitly.
- **Refund timing**: Gold must be refunded before Phase 2 starts, not at end of
  tick. **Mitigation**: Refund happens inline during Phase 1 command processing.

## Performance Implications
- **CPU**: Negligible — one list append per failed command
- **Memory**: Small — list cleared each tick, typically 0-5 entries
- **Load Time**: None
- **Network**: N/A

## Migration Plan

1. **Add `FailedCommand` struct** to AgentSDK
2. **Add `GetFailedCommands()` method** to IGameState interface (returns empty
   list by default — existing implementations won't break)
3. **Add failed command list** to SimGameState and Unity GameState implementations
4. **Modify Phase 1 processing**: When a command fails validation after gold
   deduction, refund gold and append to failed list
5. **Clear failed list** at the start of each tick's Phase 1
6. **Add parity test**: Verify failed command lists are identical in both engines
7. **Update PlanningAgentBase**: Add `FailedCommands` property for convenience
8. **Update student docs**: Document GetFailedCommands() with examples

## Validation Criteria

- [ ] Failed commands appear in GetFailedCommands() on the tick after they fail
- [ ] Silently dropped duplicate-unit commands do NOT appear
- [ ] Gold is refunded before Phase 2 for all failed Build/Train commands
- [ ] Failed command list is identical in SimGame and Unity
- [ ] Existing agents (compiled against old SDK) still work (additive change)
- [ ] CommandResult enum has all documented status codes

## Related Decisions
- [ADR-0001: Dual-Engine Parity](adr-0001-dual-engine-parity.md) — Shared code architecture
- [design/gdd/tick-engine.md](../../design/gdd/tick-engine.md) — Phase 1 command processing rules
- [design/gdd/command-validation.md](../../design/gdd/command-validation.md) — Validation rules and result codes
- [design/gdd/agent-sdk.md](../../design/gdd/agent-sdk.md) — Student-facing API contract
