# Systems Index: Agents of Empires

> **Status**: Approved
> **Created**: 2026-03-26
> **Last Updated**: 2026-03-26
> **Source Concept**: design/gdd/game-concept.md

---

## Overview

Agents of Empires is an AI competition framework built as an RTS spectator game.
The systems decomposition reflects its dual nature: a deterministic simulation engine
that runs identically in headless (SimGame) and visual (Unity) modes, and a student-facing
platform for building, testing, and competing with AI agents.

The core loop is: students write agents → agents compete in matches → matches are
spectated and scored → rankings are updated. This requires simulation systems (tick
engine, combat, economy), an agent framework (SDK, loader, security), competition
infrastructure (tournament, ELO, recording), and presentation (spectator UI, juice, audio).

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | Tick Engine | Sim Core | Tier 1 | Designed | design/gdd/tick-engine.md | — |
| 2 | Balance Data | Sim Core | Tier 1 | Designed | design/gdd/balance-data.md | — |
| 3 | Map System | Sim Core | Tier 1 | Designed | design/gdd/map-system.md | — |
| 4 | Unit Lifecycle | Gameplay | Tier 1 | Designed | design/gdd/unit-lifecycle.md | Tick Engine, Balance Data |
| 5 | Pathfinding | Gameplay | Tier 1 | Designed | design/gdd/pathfinding.md | Map System |
| 6 | Command Validation | Gameplay | Tier 1 | Designed | design/gdd/command-validation.md | Tick Engine, Balance Data |
| 7 | Combat | Gameplay | Tier 1 | Designed | design/gdd/combat.md | Unit Lifecycle, Command Validation, Pathfinding, Balance Data |
| 8 | Economy | Gameplay | Tier 1 | Designed | design/gdd/economy.md | Unit Lifecycle, Command Validation, Pathfinding, Balance Data |
| 9 | Production | Gameplay | Tier 1 | Designed | design/gdd/production.md | Unit Lifecycle, Command Validation, Economy, Balance Data |
| 10 | Healing | Gameplay | Tier 1 | Designed | design/gdd/healing.md | Unit Lifecycle, Command Validation, Pathfinding, Balance Data |
| 11 | Match Flow | Gameplay | Tier 1 | Designed | design/gdd/match-flow.md | Unit Lifecycle, Combat, Economy, Tick Engine |
| 12 | Agent SDK | Agent Framework | Tier 1 | Designed | design/gdd/agent-sdk.md | Balance Data, Map System |
| 13 | Sim Parity | Agent Framework | Tier 1 | Designed | design/gdd/sim-parity.md | Tick Engine, Combat, Economy, Production, Healing, Pathfinding |
| 14 | Procedural Map Gen | Map & World | Tier 1 | Designed | design/gdd/procedural-map-gen.md | Map System |
| 15 | Agent Loader | Agent Framework | Tier 2 | Not Started | — | Agent SDK |
| 16 | Agent Security | Agent Framework | Tier 2 | Not Started | — | Agent Loader, Agent SDK |
| 17 | Build Pipeline | Student Experience | Tier 2 | Not Started | — | Agent SDK, Agent Loader |
| 18 | Opponent Library | Agent Framework | Tier 2 | Not Started | — | Agent SDK, Combat, Economy, Production |
| 19 | Student Docs | Student Experience | Tier 2 | Not Started | — | Agent SDK, Build Pipeline, Opponent Library |
| 20 | Match Recording | Tournament | Tier 3 | Not Started | — | Tick Engine, Command Validation, Match Flow |
| 21 | Analytics | Tournament | Tier 3 | Not Started | — | Match Flow, Agent SDK |
| 22 | Tournament Runner | Tournament | Tier 3 | Not Started | — | Match Flow, Agent Loader, Sim Parity, Match Recording |
| 23 | ELO / Rating | Tournament | Tier 3 | Not Started | — | Tournament Runner, Match Flow |
| 24 | Camera | Presentation | Tier 4 | Not Started | — | Map System |
| 25 | Spectator UI | Presentation | Tier 4 | Not Started | — | Match Flow, Economy, Camera |
| 26 | Debug Overlays | Presentation | Tier 4 | Not Started | — | Map System, Pathfinding, Combat, Camera |
| 27 | Unit Animation | Presentation | Tier 4 | Not Started | — | Unit Lifecycle, Combat, Economy, Production, Healing |
| 28 | Micro-interactions | Presentation | Tier 4 | Not Started | — | Unit Animation, Combat, Healing, Production, Economy |
| 29 | Audio | Presentation | Tier 4 | Not Started | — | Combat, Economy, Production, Healing, Match Flow |

---

## Categories

| Category | Description |
|----------|-------------|
| **Sim Core** | Foundation systems the deterministic simulation depends on |
| **Gameplay** | Game rules: combat, economy, production, healing, match logic |
| **Agent Framework** | Student-facing SDK, agent loading, security, and AI opponents |
| **Map & World** | Grid structure, procedural generation, map variety |
| **Tournament** | Competition infrastructure: brackets, ratings, recording, analytics |
| **Presentation** | Visual and audio: camera, UI, overlays, animation, juice, sound |
| **Student Experience** | Build pipeline, documentation, onboarding |

---

## Priority Tiers

| Tier | Definition | Target Milestone |
|------|------------|------------------|
| **Tier 1: Core** | Simulation runs correctly, all gameplay works, parity verified | Current + Parity Fix |
| **Tier 2: Student-Ready** | Students can securely build, load, and test agents | Student SDK release |
| **Tier 3: Tournament-Ready** | Automated competitions with rankings and replay | First tournament |
| **Tier 4: Polish** | Visual juice, audio, spectator features | Ongoing polish |

---

## Dependency Map

### Layer 0 — Foundation (no dependencies)

1. **Tick Engine** — Deterministic tick loop; the simulation contract everything conforms to
2. **Balance Data** — Centralized tuning values all gameplay systems reference
3. **Map System** — Grid structure that pathfinding, camera, and overlays depend on

### Layer 1 — Core (depends on Foundation)

1. **Pathfinding** — depends on: Map System
2. **Command Validation** — depends on: Tick Engine, Balance Data
3. **Unit Lifecycle** — depends on: Tick Engine, Balance Data
4. **Agent SDK** — depends on: Balance Data, Map System
5. **Camera** — depends on: Map System

### Layer 2 — Gameplay (depends on Core)

1. **Combat** — depends on: Unit Lifecycle, Command Validation, Pathfinding, Balance Data
2. **Economy** — depends on: Unit Lifecycle, Command Validation, Pathfinding, Balance Data
3. **Production** — depends on: Unit Lifecycle, Command Validation, Economy, Balance Data
4. **Healing** — depends on: Unit Lifecycle, Command Validation, Pathfinding, Balance Data
5. **Match Flow** — depends on: Unit Lifecycle, Combat, Economy, Tick Engine
6. **Procedural Map Gen** — depends on: Map System

### Layer 3 — Agent Framework (depends on Gameplay)

1. **Agent Loader** — depends on: Agent SDK
2. **Agent Security** — depends on: Agent Loader, Agent SDK
3. **Sim Parity** — depends on: Tick Engine, Combat, Economy, Production, Healing, Pathfinding
4. **Opponent Library** — depends on: Agent SDK, Combat, Economy, Production
5. **Build Pipeline** — depends on: Agent SDK, Agent Loader

### Layer 4 — Presentation (wraps Gameplay)

1. **Spectator UI** — depends on: Match Flow, Economy, Camera
2. **Debug Overlays** — depends on: Map System, Pathfinding, Combat, Camera
3. **Unit Animation** — depends on: Unit Lifecycle, Combat, Economy, Production, Healing
4. **Micro-interactions** — depends on: Unit Animation, Combat, Healing, Production, Economy
5. **Audio** — depends on: Combat, Economy, Production, Healing, Match Flow

### Layer 5 — Tournament & Meta (depends on Agent Framework)

1. **Match Recording** — depends on: Tick Engine, Command Validation, Match Flow
2. **Analytics** — depends on: Match Flow, Agent SDK
3. **Tournament Runner** — depends on: Match Flow, Agent Loader, Sim Parity, Match Recording
4. **ELO / Rating** — depends on: Tournament Runner, Match Flow
5. **Student Docs** — depends on: Agent SDK, Build Pipeline, Opponent Library

---

## Recommended Design Order

| Order | System | Priority | Layer | Lead Agent(s) | Est. Effort |
|-------|--------|----------|-------|---------------|-------------|
| 1 | Tick Engine | Tier 1 | L0 | engine-programmer, systems-designer | M |
| 2 | Balance Data | Tier 1 | L0 | systems-designer, economy-designer | M |
| 3 | Map System | Tier 1 | L0 | systems-designer, level-designer | S |
| 4 | Unit Lifecycle | Tier 1 | L1 | systems-designer, gameplay-programmer | M |
| 5 | Pathfinding | Tier 1 | L1 | ai-programmer, engine-programmer | M |
| 6 | Command Validation | Tier 1 | L1 | systems-designer, gameplay-programmer | S |
| 7 | Combat | Tier 1 | L2 | game-designer, systems-designer | L |
| 8 | Economy | Tier 1 | L2 | economy-designer, systems-designer | M |
| 9 | Production | Tier 1 | L2 | systems-designer, gameplay-programmer | M |
| 10 | Healing | Tier 1 | L2 | systems-designer, game-designer | S |
| 11 | Match Flow | Tier 1 | L2 | game-designer, systems-designer | M |
| 12 | Agent SDK | Tier 1 | L1 | lead-programmer, systems-designer | L |
| 13 | Sim Parity | Tier 1 | L3 | lead-programmer, qa-lead | L |
| 14 | Procedural Map Gen | Tier 1 | L2 | level-designer, gameplay-programmer | M |
| 15 | Agent Loader | Tier 2 | L3 | engine-programmer, security-engineer | M |
| 16 | Agent Security | Tier 2 | L3 | security-engineer, lead-programmer | L |
| 17 | Build Pipeline | Tier 2 | L3 | devops-engineer, tools-programmer | M |
| 18 | Opponent Library | Tier 2 | L3 | ai-programmer, game-designer | L |
| 19 | Student Docs | Tier 2 | L5 | community-manager, game-designer | M |
| 20 | Match Recording | Tier 3 | L5 | gameplay-programmer, tools-programmer | M |
| 21 | Analytics | Tier 3 | L5 | analytics-engineer, tools-programmer | M |
| 22 | Tournament Runner | Tier 3 | L5 | tools-programmer, devops-engineer | L |
| 23 | ELO / Rating | Tier 3 | L5 | systems-designer, analytics-engineer | S |
| 24 | Camera | Tier 4 | L1 | gameplay-programmer | S |
| 25 | Spectator UI | Tier 4 | L4 | ux-designer, ui-programmer | M |
| 26 | Debug Overlays | Tier 4 | L4 | tools-programmer, ui-programmer | S |
| 27 | Unit Animation | Tier 4 | L4 | technical-artist, gameplay-programmer | M |
| 28 | Micro-interactions | Tier 4 | L4 | technical-artist, sound-designer | L |
| 29 | Audio | Tier 4 | L4 | audio-director, sound-designer | M |

Effort: **S** = 1 session, **M** = 2-3 sessions, **L** = 4+ sessions

---

## Circular Dependencies

None found. The dependency graph is a clean DAG.

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| Sim Parity | Technical | SimGame and Unity must produce identical results tick-for-tick; any divergence breaks fairness | Automated parity tests on every code change; shared GameConstants; command-level replay comparison |
| Agent Security | Technical | Students may attempt reflection, file I/O, threading, or timing attacks to gain advantage | Sandboxed AppDomain or process isolation; whitelist allowed assemblies; resource limits; code review of SDK surface |
| Balance Data | Design | 11 units with R-P-S triangle + economy creates complex interaction space; small changes cascade | Formal balance model with spreadsheet; automated balance tests; run tournament data to detect dominant strategies |
| Tournament Runner | Scope | Full bracket system with ELO, replay, and dashboard is substantial infrastructure | Start with simple round-robin CLI; add ELO and UI incrementally |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 29 |
| Design docs started | 14 |
| Design docs reviewed | 0 |
| Design docs approved | 0 |
| Tier 1 systems designed | 14 / 14 |
| Tier 2 systems designed | 0 / 5 |
| Tier 3 systems designed | 0 / 4 |
| Tier 4 systems designed | 0 / 6 |

---

## Next Steps

- [ ] Design Tier 1 systems (use `/design-system [system-name]`)
- [ ] Run `/design-review` on each completed GDD
- [ ] Run `/balance-check` after Balance Data and Combat GDDs are written
- [ ] Run `/gate-check pre-production` when Tier 1 systems are designed
- [ ] Prototype Sim Parity testing framework early (highest risk)
