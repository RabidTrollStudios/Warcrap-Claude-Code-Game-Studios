# Sprint 1 — 2026-03-26 to 2026-04-02

## Sprint Goal
Implement the foundational design decisions that unify SimGame and Unity into a
single source of truth, establishing the parity baseline for all future work.

## Capacity
- Total days: 5 (1 week, solo developer)
- Buffer (20%): 1 day reserved for unplanned work
- Available: 4 days

## Tasks

### Must Have (Critical Path)

| ID | Task | Source | Est. Hours | Dependencies | Acceptance Criteria |
|----|------|--------|-----------|-------------|-------------------|
| S1-01 | **Eliminate RTS/Constants.cs duplication** — Replace all copied Balance Data dictionaries with direct references to `GameConstants.*` and `DerivedGameConstants.*`. Keep only Unity-specific values (colors, directions). | ADR-0001, Balance Data GDD | 4h | — | Zero duplicated balance dictionaries in RTS/Constants.cs. All gameplay values read from AgentSDK. |
| S1-02 | **Remove `GameConstants.MOVEMENT_SPEED`** — Delete the legacy dictionary. Verify all code uses `DerivedGameConstants.SPEED_MULTIPLIER` instead. | Balance Data GDD | 1h | S1-01 | MOVEMENT_SPEED dictionary removed. No compile errors. All movement uses SPEED_MULTIPLIER. |
| S1-03 | **Simplify grid to 2 layers** — Remove the `isPassage` field from GameGrid and GridCell. Building top-row cells use `walkable=true, buildable=false` (no dedicated flag). | Map System GDD | 3h | — | Passage field removed. All passage checks replaced with walkable+!buildable. Parity tests pass. |
| S1-04 | **Update healing constants** — Change HEAL_FRACTION → flat HEAL_AMOUNT=100, MANA_COST=25→10, eligibility from threshold% to `Health ≤ MaxHealth - HEAL_AMOUNT`. Remove HEAL_FRACTION and HEAL_THRESHOLD constants. | Balance Data GDD, Healing GDD | 3h | S1-01 | Healing uses flat 100 HP. Mana cost is 10. Old constants removed. Monks can heal all mobile unit types. |
| S1-05 | **Update scoring formula** — Replace hand-tuned UNIT_VALUE with `ceil(COST / SCORING_SCALAR)` where SCORING_SCALAR=20. Add SCORING_SCALAR constant. | Balance Data GDD | 2h | S1-01 | UNIT_VALUE matches formula for all 11 types. Existing scoring tests updated. |
| S1-06 | **Run full parity test suite** — Verify SimGame and Unity produce identical hashes after all changes. Fix any divergences. | Sim Parity GDD | 4h | S1-01 through S1-05 | All existing parity tests pass. No hash divergences on any opponent matchup. |

### Should Have

| ID | Task | Source | Est. Hours | Dependencies | Acceptance Criteria |
|----|------|--------|-----------|-------------|-------------------|
| S1-07 | **Add `GetFailedCommands()` to IGameState** — Implement FailedCommand struct, add method to IGameState, populate during Phase 1, clear each tick. | ADR-0002, Tick Engine GDD | 4h | S1-06 | Method returns failed commands. Parity tests verify identical lists in both engines. |
| S1-08 | **Add gold refund on failed commands** — When Build/Train passes validation but action fails, refund gold before Phase 2. | ADR-0002, Tick Engine GDD | 3h | S1-07 | Gold refunded for failed Build/Train. No gold lost to timing conflicts. Parity tests pass. |
| S1-09 | **Expand CommandResult enum** — Add all status codes from Command Validation GDD (InvalidUnit, NotOwned, IncapableUnit, etc.). | Command Validation GDD | 2h | — | All 13 CommandResult codes exist. Queue-time validation returns correct codes. |

### Nice to Have

| ID | Task | Source | Est. Hours | Dependencies | Acceptance Criteria |
|----|------|--------|-----------|-------------|-------------------|
| S1-10 | **Add DIAGONAL_COST constant** — Pin diagonal movement cost as `1.41421356f` fixed constant in GameConstants rather than computing at runtime. | Tick Engine GDD | 0.5h | — | Constant defined. All diagonal cost references use it. |
| S1-11 | **Add startup validation** — Verify all GameConstants dictionaries cover all UnitType enum values at game start. Log error if missing. | Balance Data GDD | 1h | S1-01 | Missing dictionary entries detected at startup. |
| S1-12 | **Clean up Godot/Unreal reference docs** — Remove leftover engine reference docs for engines we're not using. | Gate check | 0.5h | — | Only `docs/engine-reference/unity/` remains. |

## Carryover from Previous Sprint
N/A — first sprint.

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Constants elimination breaks Unity compilation | Medium | High | Incremental changes with compile check after each dictionary replacement |
| Parity tests reveal pre-existing divergences | High | Medium | Fix divergences found, but don't block sprint on edge cases — log as bugs for Sprint 2 |
| Healing rebalance changes opponent AI behavior | Medium | Low | Expected — opponent AIs read from GameConstants. Behavior change is intentional. Re-run opponent tests. |

## Dependencies on External Factors
- Access to Unity project at `C:/Git/Warcrap/UnityRTS/RTS/`
- Ability to build and run the existing test suite (`build-all.sh`)

## Estimated Hours
- Must Have: 17h
- Should Have: 9h
- Nice to Have: 2h
- **Total committed (Must + Should): 26h across 4 available days (~6.5h/day)**

## Definition of Done for this Sprint
- [ ] All Must Have tasks completed
- [ ] All tasks pass acceptance criteria
- [ ] Full parity test suite passes with zero divergences
- [ ] No compile errors in any project (AgentSDK, SimGame, PlanningAgent.Tests, Unity)
- [ ] Design documents updated if any implementation deviates from GDD
- [ ] Changes committed with descriptive commit messages
