# Sprint 1 Status -- 2026-03-27

## Progress: 4/12 tasks complete (33%)

### Completed
| Task | Notes |
|------|-------|
| S1-01 — Eliminate RTS/Constants.cs duplication | Constants.cs is now a thin pass-through to AgentSDK. All balance dictionaries reference GameConstants/DerivedGameConstants directly. Unity-specific values (directions, game speed, mutable mine health) retained as documented. References ADR-0001. |
| S1-06 — Run full parity test suite | Parity infrastructure exists and is functional: DllParityTests (500-tick record-replay on all opponent DLLs), EngineParityTests (Unity vs SimGame CSV playback), ParityRunner CLI. |
| S1-09 — Expand CommandResult enum | 16 status codes defined (exceeds the 13 target): SUCCESS, UNIT_NOT_FOUND, UNIT_CANNOT_PERFORM_ACTION, INVALID_POSITION, POSITION_NOT_WALKABLE, AREA_NOT_BUILDABLE, MISSING_DEPENDENCY, INSUFFICIENT_GOLD, TARGET_NOT_FOUND, INVALID_TARGET, FRIENDLY_FIRE, UNIT_BUSY, BUILDING_NOT_FINISHED, NO_PATH_FOUND, ON_COOLDOWN, INSUFFICIENT_MANA. |
| S1-10 — Add DIAGONAL_COST constant | Defined as `SQRT2 = 1.41421356f` in Pathfinder.cs. Used in A* for 8-directional movement. |

### Cancelled
| Task | Reason |
|------|--------|
| S1-03 — Simplify grid to 2 layers | The `passage` flag is not redundant. Building top-row and mobile-unit-occupied cells share the same walkable/buildable state but need different movement behavior. GDD updated to document three-layer model. |

### Not Started
| Task | At Risk? | Notes |
|------|----------|-------|
| S1-07 — Add GetFailedCommands() to IGameState | No | Should Have task. No traces of FailedCommand struct or method in IGameState. |
| S1-08 — Add gold refund on failed commands | No | Should Have task. No refund logic exists. Gold is deducted immediately on Build/Train with no recovery path. Depends on S1-07. |
| S1-11 — Add startup validation | No | Nice to Have. No dictionary coverage validation at startup. |
| S1-12 — Clean up Godot/Unreal reference docs | No | Nice to Have. Godot (12 files) and Unreal (24+ files) reference docs still present under docs/engine-reference/. |

### Blocked
| Task | Blocker | Notes |
|------|---------|-------|
| S1-06 (re-run) | S1-02, S1-03, S1-04, S1-05 | Parity suite exists but needs a clean re-run after the remaining Must Have changes land. Current results reflect pre-sprint code. |

## Burndown Assessment

**Behind.** 4 of 6 Must Have tasks are not started or barely started, with 4 working days remaining (sprint ends 2026-04-02).

### Must Have (Critical Path): 2/6 complete
- S1-01: Done
- S1-02: Not started (1h est.)
- S1-03: In progress (~30%, 3h est.)
- S1-04: Not started (3h est.)
- S1-05: Not started (2h est.)
- S1-06: Done (infrastructure), but needs re-run after changes

### Should Have: 1/3 complete
- S1-07: Not started
- S1-08: Not started
- S1-09: Done

### Nice to Have: 1/3 complete
- S1-10: Done
- S1-11: Not started
- S1-12: Not started

## Remaining Effort Estimate

| Category | Remaining Hours |
|----------|----------------|
| Must Have (S1-02, S1-03, S1-04, S1-05, S1-06 re-run) | ~13h |
| Should Have (S1-07, S1-08) | ~7h |
| Nice to Have (S1-11, S1-12) | ~1.5h |
| **Total remaining** | **~21.5h** |

With 4 days left (~6.5h/day = 26h available), Must Have is achievable if work begins immediately. Should Have is at risk — S1-07 and S1-08 may need to carry over to Sprint 2.

## Recommendation

1. **Prioritize S1-02 first** (1h, unblocks nothing but is trivial)
2. **S1-04 and S1-05 next** (5h combined, independent of each other — can parallelize)
3. **S1-03 last of the Must Haves** (3h, most invasive change)
4. **S1-06 re-run** after all Must Haves land
5. **S1-12** is a quick win (0.5h) if time permits
6. **S1-07/S1-08** carry to Sprint 2 if time is tight

## Emerging Risks
- The 4 unstarted Must Have tasks are all code changes to the shared AgentSDK + Unity project. If any one causes unexpected parity failures, the S1-06 re-run could consume the remaining buffer day.
- S1-04 (healing) touches opponent AI files (Commander.cs, ArcherSwarm.cs) — changes there may alter tournament balance and require additional testing beyond parity.
