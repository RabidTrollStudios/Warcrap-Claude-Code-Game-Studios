# Technical Preferences

<!-- Populated by /setup-engine. Updated as the user makes decisions throughout development. -->
<!-- All agents reference this file for project-specific standards and conventions. -->

## Engine & Language

- **Engine**: Unity 6.0.3 (6000.0.35f2)
- **Language**: C# (.NET Standard 2.1 for shared SDK, .NET 8.0 for tools)
- **Rendering**: URP (Universal Render Pipeline)
- **Physics**: Unity 2D Physics

## Source Project

- **Location**: `C:/Git/Warcrap/UnityRTS/`
- **Unity Project**: `C:/Git/Warcrap/UnityRTS/RTS/`
- **Shared SDK**: `C:/Git/Warcrap/UnityRTS/AgentSDK/` (netstandard2.1)
- **Test Harness**: `C:/Git/Warcrap/UnityRTS/AgentTestHarness/` (netstandard2.1)
- **Parity Runner**: `C:/Git/Warcrap/UnityRTS/ParityRunner/` (net8.0)
- **Opponent AIs**: `C:/Git/Warcrap/UnityRTS/Opponents/`

## Naming Conventions

- **Classes**: PascalCase (e.g., `PlayerController`)
- **Public fields/properties**: PascalCase (e.g., `MoveSpeed`)
- **Private fields**: _camelCase (e.g., `_moveSpeed`)
- **Methods**: PascalCase (e.g., `TakeDamage()`)
- **Signals/Events**: PascalCase with `On` prefix (e.g., `OnHealthChanged`)
- **Files**: PascalCase matching class (e.g., `PlayerController.cs`)
- **Scenes/Prefabs**: PascalCase (e.g., `MainScene.unity`)
- **Constants**: PascalCase or UPPER_SNAKE_CASE (e.g., `MaxHealth` or `MAX_HEALTH`)

## Performance Budgets

- **Target Framerate**: 60fps (Unity spectator frontend)
- **Frame Budget**: 16.6ms total, of which:
  - Tick simulation: < 2ms (excluding agent Update time)
  - Agent Update: < 50ms per agent per tick (tournament enforcement, see Agent Security)
  - Pathfinding: < 2ms per call, < 10ms total per tick (50 concurrent requests)
  - State hash: < 0.5ms per tick
  - Rendering: remainder (~12ms)
- **Max Units Per Side**: 100 (200 total on field)
- **Max Map Size**: 100×100 (pathfinding budget scales with map area)
- **Memory Ceiling**: 512MB (Unity process total, including rendering)

## Testing

- **Framework**: NUnit (Unity PlayMode + EditMode tests, standalone parity tests)
- **Parity Testing**: SimGame harness mirrors Unity engine for deterministic replay
- **Minimum Coverage**: [TO BE CONFIGURED]
- **Required Tests**: Balance formulas, gameplay systems, agent parity (SimGame vs Unity)

## Forbidden Patterns

<!-- Add patterns that should never appear in this project's codebase -->
- [None configured yet — add as architectural decisions are made]

## Allowed Libraries / Addons

<!-- Add approved third-party dependencies here -->
- Tiny Swords sprite pack (fantasy medieval art assets)
- TextMesh Pro (UI text rendering)
- Unity Input System (new input handling)

## Architecture Decisions Log

<!-- Quick reference linking to full ADRs in docs/architecture/ -->
- [No ADRs yet — use /architecture-decision to create one]
