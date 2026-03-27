# Claude Code Game Studios -- Game Studio Agent Architecture

Indie game development managed through 48 coordinated Claude Code subagents.
Each agent owns a specific domain, enforcing separation of concerns and quality.

## Technology Stack

- **Engine**: Unity 6.0.3 (6000.0.35f2)
- **Language**: C# (.NET Standard 2.1 for shared SDK, .NET 8.0 for tools)
- **Version Control**: Git with trunk-based development
- **Build System**: Unity Build Pipeline + MSBuild (AgentSDK/Harness)
- **Asset Pipeline**: Unity Asset Import Pipeline

> **Note**: This project uses Unity engine-specialist agents (unity-specialist,
> unity-shader-specialist, unity-ui-specialist, unity-addressables-specialist,
> unity-dots-specialist). Source project lives at `C:/Git/Warcrap/UnityRTS/`.

## Project Structure

@.claude/docs/directory-structure.md

## Engine Version Reference

@docs/engine-reference/unity/VERSION.md

## Technical Preferences

@.claude/docs/technical-preferences.md

## Coordination Rules

@.claude/docs/coordination-rules.md

## Collaboration Protocol

**User-driven collaboration, not autonomous execution.**
Every task follows: **Question -> Options -> Decision -> Draft -> Approval**

- Agents MUST ask "May I write this to [filepath]?" before using Write/Edit tools
- Agents MUST show drafts or summaries before requesting approval
- Multi-file changes require explicit approval for the full changeset
- No commits without user instruction

See `docs/COLLABORATIVE-DESIGN-PRINCIPLE.md` for full protocol and examples.

> **First session?** If the project has no engine configured and no game concept,
> run `/start` to begin the guided onboarding flow.

## Coding Standards

@.claude/docs/coding-standards.md

## Context Management

@.claude/docs/context-management.md
