# Sprint 2 — 2026-03-28 to 2026-04-04

## Sprint Goal
Verify all Tier 1 gameplay systems against their GDD acceptance criteria,
close ADR-0002 validation gaps, and re-export the Unity parity snapshot.

## Capacity
- Total days: 5 (1 week, solo developer)
- Buffer (20%): 1 day reserved for unplanned work
- Available: 4 days

## Tasks

### Must Have (Critical Path)

| ID | Task | Source | Est. Hours | Dependencies | Acceptance Criteria |
|----|------|--------|-----------|-------------|-------------------|
| S2-01 | **Re-export Unity parity snapshot** — Run the game in Unity with ParityExporter enabled to generate fresh ParityCommands/ParityState CSVs that reflect Sprint 1 balance changes (MANA_COST=10, HEAL_AMOUNT=100, computed UNIT_VALUE). | Sim Parity GDD | 1h | Unity editor | `EngineExport_ReplayMatchesSnapshots` test passes. 0 divergences. |
| S2-02 | **ADR-0002 parity verification** — Write a parity scenario where a Build command fails after queue-time (area becomes blocked by another build in sort order). Verify `GetFailedCommands()` returns identical results in SimGame record vs replay. Verify gold is refunded. | ADR-0002 | 3h | S2-01 | New parity scenario `BuildConflictRefund` passes. FailedCommand list matches. Gold restored. |
| S2-03 | **Verify combat formula accuracy** — Write targeted tests that assert exact damage-per-tick values for each R-P-S matchup at GameSpeed 20. Verify `DamagePerTick = BASE_DAMAGE × ScalarDamage × Multiplier × TickDuration`. Test retargeting (target dies → closest enemy or IDLE). | Combat GDD | 3h | — | All 9 R-P-S multiplier combinations verified. Retarget-on-death tested. Damage values match GDD examples. |
| S2-04 | **Verify healing acceptance criteria** — Write tests for: flat HEAL_AMOUNT heal, eligibility rejection when <100 HP missing, overheal capping, mana deduction, mana regen rate at GameSpeed 20, monk cannot sustain a unit vs single attacker. | Healing GDD | 3h | — | All 6 healing acceptance criteria from GDD verified with automated tests. |
| S2-05 | **Verify economy acceptance criteria** — Write tests for: gold deduction on Build/Train, mine depletion (health decreases per gather trip), pawn carries min(MINING_CAPACITY, mine.Health), mine removal at 0 health, pawn goes IDLE on depletion. | Economy GDD | 3h | — | All gold flow paths tested. Mine depletion verified. Parity scenario covers full gather→deplete cycle. |
| S2-06 | **Verify scoring/timeout acceptance criteria** — Write test that runs a match to timeout, verifies `UNIT_VALUE = ceil(COST/20)` produces correct winner, and gold tiebreaker works. | Balance Data GDD, Match Flow GDD | 2h | — | Timeout produces correct winner. Tiebreaker resolves ties. Scoring matches formula for all 11 types. |

### Should Have

| ID | Task | Source | Est. Hours | Dependencies | Acceptance Criteria |
|----|------|--------|-----------|-------------|-------------------|
| S2-07 | **Verify production chain acceptance criteria** — Test: unbuilt buildings can't train, building rejects Train while busy, dependency checking, spawn position selection for trained units. | Production GDD | 2h | — | All production edge cases tested. BuildingBusy, dependency, and spawn tests pass. |
| S2-08 | **Verify match flow lifecycle** — Test full best-of-3 progression: InitializeMatch → (InitializeRound → Update loop → Learn) × 3 → final winner. Verify round resets (gold, units, map). Test mutual elimination (draw). | Match Flow GDD | 3h | — | 3-round match runs correctly. Round resets verified. Draw handled. Agent lifecycle order correct. |
| S2-09 | **Verify map generation fairness** — Run symmetry, connectivity, and path-balance tests across all 3 templates (OpenField, Forest, Maze) for map sizes 15×15, 30×30, 60×60. Verify same seed → identical map. | Procedural Map Gen GDD | 2h | — | All templates pass fairness. Spawn-to-mine path ±1 cell. Deterministic across runs. |
| S2-10 | **Agent exception resilience** — Test that agent throwing in Update(), InitializeMatch(), InitializeRound(), and Learn() doesn't crash the match. Agent should be skipped for that tick, not terminated. | Tick Engine GDD, Agent SDK GDD | 2h | — | Exception in each lifecycle method is non-fatal. Match continues. Other agent unaffected. |

### Nice to Have

| ID | Task | Source | Est. Hours | Dependencies | Acceptance Criteria |
|----|------|--------|-----------|-------------|-------------------|
| S2-11 | **Update ConstantsUnitValueTests for new scoring** — The Unity EditMode tests in `ConstantsUnitValueTests.cs` assert old hardcoded UNIT_VALUE values. Update to match `ceil(COST/20)` formula. | Balance Data GDD | 1h | — | All Unity EditMode UNIT_VALUE tests pass with new computed values. |
| S2-12 | **Design decision: Agent Update() timeout** — Tick Engine GDD edge case #6 flags timeout enforcement as unresolved. Draft ADR-0003 with options (50ms hard kill, 50ms soft warning, no timeout until tournament). | Tick Engine GDD | 1h | — | ADR-0003 written with decision and rationale. Tick Engine GDD updated. |

## Carryover from Previous Sprint

| Task | Reason | New Estimate |
|------|--------|-------------|
| Unity-side `GetFailedCommands()` population | Plumbing in place (Agent.RecordFailedCommand → AgentBridge → GameStateAdapter), needs verification via S2-01 re-export | Covered by S2-01 + S2-02 |
| `EngineExport_ReplayMatchesSnapshots` failure | Stale CSV snapshots from pre-Sprint-1 balance values | Covered by S2-01 |

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Unity parity snapshot reveals new divergences from Sprint 1 changes | Medium | High | Fix divergences immediately — they block all downstream verification. Budget 2h for unexpected fixes in S2-01. |
| Combat/healing formula tests reveal SimGame implementation doesn't match GDD | Medium | Medium | The GDD is authoritative. If code diverges, fix code. Track as bug, fix inline. |
| Unity EditMode tests fail after UNIT_VALUE formula change | High | Low | Known — S2-11 fixes this. Non-blocking since Unity tests run separately. |
| Map generation fairness tests fail on some templates | Low | Medium | OpenField is low risk. Forest/Maze may need tuning. Defer template fixes to Sprint 3 if >2h. |

## Dependencies on External Factors
- Unity editor access for S2-01 (parity re-export) and S2-11 (EditMode tests)
- All opponent DLLs must be rebuilt after Sprint 1 changes (done in Sprint 1)

## Estimated Hours
- Must Have: 15h
- Should Have: 9h
- Nice to Have: 2h
- **Total committed (Must + Should): 24h across 4 available days (~6h/day)**

## Definition of Done for this Sprint
- [ ] All Must Have tasks completed
- [ ] `EngineExport_ReplayMatchesSnapshots` passes (no stale snapshot)
- [ ] ADR-0002 validation criteria all checked off
- [ ] All new tests pass in both `build-all.sh` and `dotnet test`
- [ ] No S1 or S2 bugs in delivered features
- [ ] Design documents updated for any deviations discovered during testing
- [ ] GDD acceptance criteria checked off as verified
