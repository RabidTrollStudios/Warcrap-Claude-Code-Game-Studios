# Sprint 2 Status — 2026-03-28

## Progress: 0/12 tasks complete (0%) — Day 1

### S2-01 Finding: Pre-existing Cross-Engine Parity Divergence

The `EngineExport_ReplayMatchesSnapshots` test fails with a **pre-existing** divergence
between Unity and SimGame. This is NOT caused by Sprint 1 changes — the old export
(2026-03-26) shows the same map/seed configuration and the same test was already failing.

**Symptoms (tick 50):**
- Unity has 14 units, SimGame has 12 (2 extra PAWNs in Unity)
- Agent 1 gold: Unity=0, SimGame=50 (Unity trained 1 more pawn)
- Mine health diverging (gathering started in Unity, not in SimGame)
- Unit positions diverging downstream

**Root cause hypothesis:** Training completion timing or command processing order
differs between Unity and SimGame. Unity successfully trains more pawns from the
same command stream, suggesting a timing difference in when TRAIN completes and
when the trained unit becomes available for subsequent commands.

**Impact:** This blocks S2-01 (re-export) and S2-02 (ADR-0002 parity verification).
All other Sprint 2 tasks (S2-03 through S2-12) are **not blocked** — they verify
SimGame gameplay logic against GDD acceptance criteria, not cross-engine parity.

**Next steps:**
1. File as a tracked parity bug
2. Investigate the TRAIN completion timing difference between engines
3. Continue with S2-03 through S2-06 (gameplay formula verification) in parallel
