---
status: reverse-documented
source: C:/Git/Warcrap/UnityRTS/
date: 2026-03-26
verified-by: Dana
---

# Agents of Empires — Game Concept

> **Note**: This document was reverse-engineered from the existing implementation
> at `C:/Git/Warcrap/UnityRTS/` and clarified through discussion with the creator.
> Some sections capture future vision beyond what is currently implemented.

---

## Overview

Agents of Empires is an AI competition framework built as a medieval RTS spectator
game. Students write C# planning agents that gather resources, build armies, and
wage war against each other in automated matches. The game runs AI-vs-AI with full
spectator controls, debug visualization, and tournament infrastructure. Built in
Unity 6.0.3 with the Tiny Swords pixel art style.

## Player Fantasy

**"I built an AI that outsmarts everyone else's."**

The "player" is a student writing code, not a human clicking units. The fantasy is
the satisfaction of watching your creation execute a strategy you designed — and the
thrill of seeing it adapt and win against opponents you've never seen before.

The secondary audience is spectators watching tournament matches, who should find
the battles visually engaging and the strategic decisions legible.

## Core Pillars

### 1. Fair Competition
Deterministic simulation, cheat-proof SDK, balanced units, and reproducible results.
Every match outcome must be explainable by the agents' decisions, never by engine
quirks or timing artifacts. SimGame (headless) and Unity must produce identical
results tick-for-tick.

### 2. Easy Onboarding
A student with C# fundamentals should be able to write a functioning agent in under
an hour. The API is small (`IGameState` for queries, `IAgentActions` for commands),
the build process is simple, and example opponents provide graduated learning targets.

### 3. Strategic Depth
The rock-paper-scissors combat triangle (Warrior > Archer > Lancer > Warrior), economic
trade-offs (invest in workers vs. rush military), and army composition choices (specialist
vs. balanced) create a rich decision space. No single dominant strategy should exist — the
meta should reward adaptation and scouting.

### 4. Spectator Appeal
Matches should be visually engaging and fun to watch. Micro-interactions (hit sparks,
death effects, gold carry animations, healing particles, building construction sequences,
projectile trails) make the action legible and satisfying. Debug overlays, speed control,
and camera tools let spectators understand what's happening at any level of detail.

### 5. Spectacle & Juice
Beyond functional correctness, the visual presentation should have polish that makes
tournament matches exciting to watch. Screen shake on big hits, particle bursts on
unit deaths, smooth interpolated movement, archer volleys arcing through the air,
monks channeling heal beams — these micro-interactions transform a simulation into
a spectacle.

## Genre & Format

- **Genre**: AI Competition Framework / RTS Spectator
- **Platform**: PC (Windows primary)
- **Engine**: Unity 6.0.3 (6000.0.35f2)
- **Language**: C# (.NET Standard 2.1 for AgentSDK, .NET 8.0 for tools)
- **Art Style**: Tiny Swords pixel art (medieval fantasy) — FINAL
- **Target Audience**: CS/Game Development students (GDD3400 and similar courses)
- **Session Length**: Individual matches ~1-5 minutes (speed-dependent); tournaments ~1 hour

## Game Mechanics

### Units (11 Types)

| Unit | Role | Cost | HP | Damage | Speed | Range | Special |
|------|------|------|----|--------|-------|-------|---------|
| Mine | Resource | — | 10,000 | — | — | — | Gold source, depletes |
| Pawn | Worker | 50g | 200 | — | 1.0x | — | Gathers, builds, repairs |
| Warrior | Melee | 100g | 1,500 | 50 dps | 2.1x | 1.0 | Strong vs Archer (1.25x) |
| Archer | Ranged | 80g | 600 | 38 dps | 3.0x | 9.0 | Strong vs Lancer (1.25x) |
| Lancer | Cavalry | 70g | 900 | 42 dps | 3.45x | 2.5 | Strong vs Warrior (1.25x) |
| Monk | Healer | 90g | 400 | — | 0.85x | 4.0 (heal) | Heals 50% HP, mana-gated |
| Base | HQ | 500g | 8,000 | — | — | — | Trains Pawns |
| Barracks | Factory | 400g | 4,000 | — | — | — | Trains Warriors |
| Archery | Factory | 350g | 4,000 | — | — | — | Trains Archers |
| Tower | Factory | 300g | 3,000 | — | — | — | Trains Lancers |
| Monastery | Factory | 350g | 3,500 | — | — | — | Trains Monks |

### Combat Triangle

```
    Warrior (1.25x)
     ↗         ↘
Lancer ←(1.25x)← Archer
     ↖         ↙
      (1.25x)
```

- Strong matchup: 1.25x damage dealt, 0.75x damage received
- Neutral matchups: 1.0x
- All units deal neutral damage to buildings and monks

### Economy

- **Starting gold**: 1,000 per player
- **Mine capacity**: 10,000 gold per mine
- **Mines per map**: 2 (configurable)
- **Mining**: Pawns carry 50 gold per trip, deposit at Base
- **Mining speed**: 20 gold/sec × game speed

### Match Structure

- **Rounds**: Best-of-3 (configurable)
- **Time limit**: 300 seconds per round
- **Win condition**: Destroy all enemy buildings OR highest unit value at timeout
- **Game speed**: Adjustable 1-30x during spectating
- **Tick rate**: 20 ticks/second

### Production Chain

```
Base (500g) ─── trains ──→ Pawn (50g)
  │
  ├── Barracks (400g) ──→ Warrior (100g)
  ├── Archery (350g) ───→ Archer (80g)
  ├── Tower (300g) ─────→ Lancer (70g)
  └── Monastery (350g) ─→ Monk (90g)
```

All military buildings require a built Base as prerequisite.

### Agent API

Students implement `IPlanningAgent` with four methods:

```csharp
void InitializeMatch();           // Once per match
void InitializeRound();           // Once per round
void Update(IGameState, IAgentActions);  // Every tick — main decision loop
void Learn(IGameState);           // Post-round reflection
```

**IGameState** provides: gold, unit lists, unit info, pathfinding, buildable positions.
**IAgentActions** provides: Move, Attack, Train, Build, Gather, Repair, Heal, Log.

### Unit Actions

| Action | Who | Description |
|--------|-----|-------------|
| IDLE | All | No activity |
| MOVE | Mobile units | Navigate to position via A* |
| TRAIN | Buildings | Produce a unit (costs gold, takes time) |
| BUILD | Pawn | Walk to site, construct building |
| GATHER | Pawn | Mine → carry gold → deposit at Base → repeat |
| ATTACK | Warrior, Archer, Lancer | Move into range, deal DPS |
| REPAIR | Pawn | Walk to building, restore HP at 110% build rate |
| HEAL | Monk | Walk into range, restore 50% of target's max HP (costs 25 mana) |

## Architecture Overview

### Project Structure

```
UnityRTS/
├── AgentSDK/            # Shared game rules & interfaces (netstandard2.1)
│   ├── GameConstants.cs     # All balance values (read-only)
│   ├── IGameState.cs        # Query interface for agents
│   ├── IAgentActions.cs     # Command interface for agents
│   ├── IPlanningAgent.cs    # Agent lifecycle interface
│   └── PlanningAgentBase.cs # Convenience base class
├── AgentTestHarness/    # Headless game simulator (netstandard2.1)
│   ├── SimGame.cs           # Tick-based game loop
│   └── SimGameState.cs      # IGameState implementation
├── PlanningAgent/       # Default/student agent (netstandard2.1)
├── PlanningAgent.Tests/ # NUnit parity & mechanic tests
├── ParityRunner/        # CLI test runner (net8.0)
├── Opponents/           # 15+ pre-built AI strategies
├── EnemyAgents/         # Compiled opponent DLLs
└── RTS/                 # Unity project (spectator UI, visualization)
```

### Dual-Engine Design

The game logic exists in two implementations that must stay in sync:

1. **SimGame** (AgentTestHarness) — Headless C# simulator for fast testing, parity
   verification, and tournament execution. No Unity dependency.
2. **Unity** (RTS project) — Visual frontend for spectating matches, debugging agents,
   and demonstrating gameplay.

Both consume the same `AgentSDK` and must produce identical results for identical
inputs. Parity testing validates this.

## Current State (as of 2026-03-26)

### What's Working
- Complete game loop with all 11 unit types
- Combat triangle with damage multipliers
- Resource gathering, building, training, healing
- 15+ opponent AIs (Idle through ArcherSwarm)
- SimGame headless simulator
- Parity testing framework (NUnit)
- Spectator UI with debug overlays (influence map, paths, targets, ranges)
- Camera pan/zoom with speed control
- Command logging and analytics CSV export
- Tiny Swords art integration with directional animations

### What's Missing / Needs Work
- **Sim Parity**: SimGame and Unity don't perfectly match yet — active priority
- **Tournament System**: No bracket runner, ELO, or match recording
- **Student SDK Security**: No sandboxing, reflection restrictions, or resource limits
- **Balance Tuning**: Numbers exist but no formal balance pass with goals/metrics
- **Micro-interactions**: Limited visual juice — needs hit effects, death animations,
  heal beams, construction sequences, screen shake, particle polish
- **Student Documentation**: No onboarding guide, API reference, or example walkthroughs
- **Audio**: No sound effects or music

## Priorities (Ordered)

1. **Sim ↔ Unity Parity** — Ensure headless sim matches Unity tick-for-tick
2. **Student SDK** — Secure, sandboxed, well-documented, easy to compile
3. **Unit Balance** — Formal balance pass with goals, metrics, and testing
4. **Micro-interactions & Juice** — Visual polish for spectator appeal
5. **Tournament Runner** — Bracket system, ELO ratings, match recording/replay
6. **Opponent Library** — More AIs at graduated difficulty levels
7. **Audio** — Sound effects and music for spectator experience
8. **Student Documentation** — Onboarding guide, API docs, example agents

## Scope Tiers

### Tier 1: Core (Current + Parity Fix)
- All 11 units functional with correct balance
- SimGame ↔ Unity parity verified
- 15+ opponent AIs working
- Basic spectator UI

### Tier 2: Student-Ready
- Secured/sandboxed agent SDK
- Student onboarding documentation
- API reference with examples
- Easy build pipeline (one command)
- Graduated opponent difficulty

### Tier 3: Tournament-Ready
- Bracket tournament runner
- ELO rating system
- Match recording and replay
- Results dashboard
- Anti-cheat enforcement

### Tier 4: Polish
- Full micro-interaction suite (hit FX, death, heal, build, projectiles)
- Audio (SFX + music)
- Map variety (procedural + hand-crafted tournament maps)
- Advanced spectator features (slow-mo replay, unit tracking, commentary overlay)

## Competitive Landscape

This is an educational tool, not a commercial game. Comparable projects:
- **Halite** (Two Sigma) — AI competition, simpler mechanics, web-based
- **Battlecode** (MIT) — Java AI competition, annual tournament
- **StarCraft II AI** (DeepMind) — Research-focused, complex API

Agents of Empires differentiates by:
- Accessible C# API (not Java or Python)
- Visual spectator appeal (Tiny Swords art, not ASCII/minimal)
- RTS mechanics familiar to gamers (Warcraft-inspired)
- Runs locally (no cloud dependency)
- Unity-based (students can extend the visual frontend)

---

**Sources:**
- `C:/Git/Warcrap/UnityRTS/AgentSDK/GameConstants.cs` — All balance values
- `C:/Git/Warcrap/UnityRTS/AgentSDK/IGameState.cs` — Agent query API
- `C:/Git/Warcrap/UnityRTS/AgentSDK/IAgentActions.cs` — Agent command API
- `C:/Git/Warcrap/UnityRTS/AgentTestHarness/SimGame.cs` — Game loop implementation
- `C:/Git/Warcrap/UnityRTS/Opponents/` — AI strategy examples
