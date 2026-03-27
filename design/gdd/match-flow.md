# Match Flow

> **Status**: In Design
> **Author**: Dana + Claude Game Studios
> **Last Updated**: 2026-03-26
> **Implements Pillar**: Fair Competition (structured matches, clear outcomes)

## Overview

Match Flow governs the high-level structure of a game: how matches are organized
into rounds, how rounds start and end, what constitutes a win, and how ties are
resolved. It sits above the Tick Engine — while the Tick Engine runs the simulation
frame by frame, Match Flow decides when to start ticking, when to stop, who won,
and what happens next.

Match Flow owns the match state machine, round initialization (placing starting
units), win condition evaluation, timeout scoring, and the agent lifecycle calls
for InitializeMatch, InitializeRound, and Learn.

## Player Fantasy

**"Every match is a fair contest with a clear winner."**

Matches have defined structure — best-of-3 rounds, time limits, deterministic
scoring. No match ends ambiguously. Agents know the rules before they play, and
spectators always know who's winning. This serves **Fair Competition**: the format
is transparent and reproducible.

## Detailed Design

### Core Rules

#### 1. Match Structure

```
Match
├── Round 1
│   ├── Setup (place starting units)
│   ├── Play (tick loop runs until win condition or timeout)
│   └── Result (winner determined, Learn() called)
├── Round 2
│   └── ...
├── Round 3 (if needed)
│   └── ...
└── Match Result (best-of-N winner)
```

- **Rounds per match**: Configurable (default 3, best-of-3)
- **Time limit per round**: Configurable (default 300 seconds)
- **Match winner**: Agent with most round wins. If tied after all rounds, match
  is a draw.

#### 2. Round Setup

Before the first tick of each round:

1. Reset game state (clear all units, reset gold, regenerate or reload map)
2. Place starting units for each player:
   - 1 Base at spawn position
   - 2 Pawns adjacent to base
   - 1 Mine per player at mine positions (neutral, belongs to no agent)
3. Set starting gold (default 1,000 per agent)
4. Call `agent.InitializeRound()` for each agent
5. Begin tick loop

Starting units are placed symmetrically per the Map System's mirror rules.

#### 3. Win Conditions (checked after Phase 4 each tick)

Win conditions are evaluated in priority order:

| Priority | Condition | Winner |
|----------|-----------|--------|
| 1 | **Elimination**: One agent has zero buildings (all buildings destroyed) | Agent with buildings remaining |
| 2 | **Mutual elimination**: Both agents lose all buildings in same tick | Draw (round has no winner) |
| 3 | **Timeout**: ElapsedTime >= TimeLimit | Agent with higher unit value score |
| 4 | **Timeout tie**: Equal unit value scores | Agent with more gold |
| 5 | **Timeout double tie**: Equal scores AND equal gold | Draw (round has no winner) |

Win conditions are checked AFTER Phase 4 (Remove Dead Units) — units killed this
tick are already removed before the check runs.

#### 4. Timeout Scoring

When time limit is reached:

```
Score[agent] = Σ (count[type] × UNIT_VALUE[type]) for all owned units
UNIT_VALUE[type] = ceil(COST[type] / SCORING_SCALAR)
SCORING_SCALAR = 20
```

See Balance Data GDD for the full UNIT_VALUE table. Tiebreaker: agent with more
gold wins.

#### 5. Agent Lifecycle Calls

| Method | When Called | Purpose |
|--------|-----------|---------|
| `InitializeMatch()` | Before first round | One-time setup (load strategy, allocate data) |
| `InitializeRound()` | Before each round, after setup | Reset per-round state |
| `Update(IGameState, IAgentActions)` | Phase 5 of every tick | Main decision loop |
| `Learn(IGameState)` | After round ends, before next round setup | Post-round analysis (who won, final state) |

#### 6. Game Speed Control

Game speed can be changed during play by the spectator (UI control). Speed changes
take effect at the next tick boundary. Match Flow does not own the speed value —
it is a spectator/presentation concern — but it must ensure elapsed time is
computed correctly regardless of speed changes.

```
ElapsedTime = CurrentTick × TickDuration
```

ElapsedTime is independent of GameSpeed — GameSpeed affects how fast things happen
within each tick, not how many ticks occur per wall-clock second.

### States and Transitions

```
MATCH_START → ROUND_SETUP → PLAYING → ROUND_END → [ROUND_SETUP or MATCH_END]
```

| State | Description | Transition |
|-------|------------|------------|
| **MATCH_START** | Call InitializeMatch() for all agents | → ROUND_SETUP |
| **ROUND_SETUP** | Reset state, place starting units, call InitializeRound() | → PLAYING |
| **PLAYING** | Tick loop runs. Win conditions checked after each Phase 4 | Win/timeout → ROUND_END |
| **ROUND_END** | Record round result. Call Learn() for all agents | More rounds needed → ROUND_SETUP; else → MATCH_END |
| **MATCH_END** | Determine match winner from round results | Terminal state |

### Interactions with Other Systems

| System | Interface | Data Flow |
|--------|-----------|-----------|
| **Tick Engine** | Match Flow starts/stops the tick loop | Control flow |
| **Unit Lifecycle** | Round setup places starting units; win check reads unit census | Spawn and query |
| **Economy** | Round setup sets starting gold; timeout tiebreaker reads gold | Gold initialization and query |
| **Balance Data** | UNIT_VALUE for timeout scoring | Scoring parameters |
| **Map System** | Round setup initializes grid (or reloads map) | Map initialization |
| **Agent SDK** | Agent lifecycle calls (InitializeMatch, InitializeRound, Update, Learn) | Agent communication |
| **Sim Parity** | Match results must be identical in both engines | Result verification |
| **Tournament Runner** | Consumes match results (winner, scores, round details) | Result reporting |
| **Match Recording** | Match start/end events for replay metadata | Metadata |

#### Ownership Boundary

Match Flow owns:
- Match state machine (start → rounds → end)
- Round setup (starting units, gold, map reset)
- Win condition evaluation (elimination, timeout, scoring)
- Agent lifecycle orchestration (InitializeMatch, InitializeRound, Learn)
- Round and match result recording

Match Flow does NOT own:
- The tick loop itself (Tick Engine)
- Game speed (spectator/UI concern)
- How scoring values are defined (Balance Data)
- How agents make decisions (Agent SDK)

## Formulas

### Elapsed Time

```
ElapsedTime = CurrentTick × TickDuration
TimeLimit = SimConfig.MaxNbrOfSeconds (default 300)
IsTimeout = ElapsedTime >= TimeLimit
```

### Timeout Score

```
Score[agent] = Σ (count[type] × UNIT_VALUE[type])
UNIT_VALUE[type] = ceil(COST[type] / 20)
```

### Match Winner

```
RoundWins[agent] = count of rounds won
MatchWinner = agent with max(RoundWins)
If RoundWins equal: Match is a draw
```

### Maximum Possible Ticks Per Round

```
MaxTicks = TimeLimit / TickDuration
Default: 300 / 0.05 = 6,000 ticks per round
```

## Edge Cases

#### 1. Both agents lose all buildings in the same tick
Round is a draw — neither agent wins the round. This counts as a played round
toward the best-of-N total.

#### 2. Agent has buildings but zero mobile units
Not a loss — buildings are still alive. The agent can still train pawns (if Base
exists) and rebuild. Win condition only checks building count.

#### 3. Timeout with identical scores and identical gold
Round is a draw. In best-of-3, this can lead to a match draw if enough rounds
draw.

#### 4. Agent crashes during InitializeMatch
The match cannot proceed. Agent forfeits the entire match. Logged as a loss with
reason "agent crash."

#### 5. Agent crashes during InitializeRound
Agent forfeits that round (opponent wins). Match continues with remaining rounds.

#### 6. Agent crashes during Learn
Learning is optional — the crash is logged but does not affect the round result
(already determined). Match continues normally.

#### 7. Round ends on tick where agent queued commands
Commands queued during the final tick's Phase 5 are never processed — the tick
loop stops after the win condition is met in Phase 4. This is deterministic.

#### 8. Time limit reached mid-Phase-2
Time limit is checked AFTER Phase 4, not during Phase 2. The full tick completes
before timeout is evaluated. This ensures the final game state is consistent.

#### 9. Three-round match where all rounds draw
Match result is a draw (0-0-0). Tournament Runner decides how to handle draws
(replay, coin flip, etc.).

## Dependencies

### Upstream

| System | What It Provides |
|--------|-----------------|
| **Unit Lifecycle** | Unit census for win conditions, starting unit placement |
| **Combat** | Combat outcomes that trigger elimination |
| **Economy** | Starting gold, gold tiebreaker |
| **Tick Engine** | Tick loop execution |

### Downstream

| System | Dependency Type | What It Needs |
|--------|----------------|---------------|
| **Agent SDK** | Hard | Lifecycle calls (InitializeMatch, InitializeRound, Learn) |
| **Tournament Runner** | Hard | Match results |
| **Match Recording** | Hard | Match metadata (rounds, results, timing) |
| **Analytics** | Soft | Per-round and per-match statistics |
| **Spectator UI** | Soft | Score display, round indicators, game over screen |

## Tuning Knobs

| Knob | Current | Safe Range | Too Low | Too High | Affects |
|------|---------|------------|---------|----------|---------|
| **TotalRounds** | 3 | 1–7 | Single round = high variance, lucky wins | Long matches, fatigue | Match reliability, tournament duration |
| **TimeLimit** | 300s | 60–600s | Games end before decisive combat | Stalemates drag on | Match pacing, timeout frequency |
| **StartingGold** | 1,000 | 200–5,000 | Can't build anything, stalled | Skip economy phase entirely | Opening strategy diversity |
| **StartingMineGold** | 10,000 | 2,000–50,000 | Very short games, resource scarcity | Effectively infinite resources | Game length, economic pressure |
| **StartingPawns** | 2 | 1–5 | Slow economy start | Fast economy, rush favored | Early game tempo |
| **MinesPerPlayer** | 1 | 1–4 | Single economic bottleneck | Abundant resources | Map control importance |
| **SCORING_SCALAR** | 20 | 5–100 | Buildings dominate scoring | All units ≈1 point | Timeout outcome granularity |

## Acceptance Criteria

#### Match Structure
- [ ] Best-of-N rounds execute correctly (default 3)
- [ ] Round setup places starting units symmetrically
- [ ] Starting gold is set correctly from config
- [ ] Agent lifecycle methods called in correct order

#### Win Conditions
- [ ] Elimination detected when an agent has zero buildings after Phase 4
- [ ] Mutual elimination correctly results in a draw
- [ ] Timeout triggers at exactly ElapsedTime >= TimeLimit
- [ ] Timeout scoring uses UNIT_VALUE formula correctly
- [ ] Gold tiebreaker resolves scoring ties

#### Agent Lifecycle
- [ ] InitializeMatch called once before first round
- [ ] InitializeRound called before each round
- [ ] Learn called after each round with final game state
- [ ] Agent exception during lifecycle does not crash the match

#### Parity
- [ ] Round results are identical in SimGame and Unity for the same agents and map
- [ ] Win condition evaluation produces identical results in both engines
- [ ] Timeout scoring produces identical scores in both engines