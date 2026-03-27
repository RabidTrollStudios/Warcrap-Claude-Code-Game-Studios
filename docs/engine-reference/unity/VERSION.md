# Unity Engine — Version Reference

| Field | Value |
|-------|-------|
| **Engine Version** | Unity 6.0.3 (6000.0.35f2) |
| **Release Date** | Early 2025 |
| **Project Pinned** | 2026-03-26 |
| **Last Docs Verified** | 2026-03-26 |
| **LLM Knowledge Cutoff** | May 2025 |

## Knowledge Gap Warning

The LLM's training data likely covers Unity up to early Unity 6.0 releases.
Unity 6.0.3 is near the edge of training data — most core APIs are known,
but patch-level changes and newer package versions may diverge.

**Risk Level: MEDIUM** — Core APIs are covered, but cross-reference this
directory before suggesting unfamiliar Unity 6 APIs.

## Post-Cutoff Version Timeline

| Version | Release | Risk Level | Key Theme |
|---------|---------|------------|-----------|
| 6.0 (6000.0) | Oct 2024 | LOW | Unity 6 rebrand, GPU Resident Drawer, Build Profiles, Sentis AI |
| 6.0.3 (6000.0.35f2) | Early 2025 | MEDIUM | **Current** — Patch fixes, stability improvements |
| 6.1 | Mid 2025 | MEDIUM | Performance enhancements |
| 6.2 | Late 2025 | HIGH | Shader API deprecations, new render graph APIs |
| 6.3 LTS | Early 2026 | HIGH | Long-term support branch |
| 6.4 | March 2026 | HIGH | Unity Studio, latest supported release |

## Major Changes from 2022 LTS to Unity 6.0

### Breaking Changes
- **Object Discovery**: `FindObjectsOfType` → `FindObjectsByType` (with sort mode)
- **Lighting**: Enlighten Baked GI removed, auto-generate lighting removed
- **Graphics Formats**: `DepthAuto`/`ShadowAuto`/`VideoAuto` obsoleted
- **UI Toolkit**: `ExecuteDefaultAction` deprecated, `UxmlTraits`/`UxmlFactory` replaced
- **Build System**: Build Settings replaced by Build Profiles
- **Android**: `UnityPlayer` class replaced, Gradle/AGP/NDK versions updated

### New Features
- **GPU Resident Drawer**: Automatic GPU instancing for compatible GameObjects
- **Build Profiles**: Per-platform build configurations
- **Sentis**: Runtime ML inference engine
- **STP Upscaling**: Built-in spatial-temporal post-processing upscaler
- **Render Graph**: New recommended approach for custom render passes (URP/HDRP)
- **visionOS**: Full Apple Vision Pro support
- **Multiplayer Center**: Guided multiplayer setup workflow

## Verified Sources

- Official upgrade guide: https://docs.unity3d.com/6000.0/Documentation/Manual/UpgradeGuideUnity6.html
- What's new: https://docs.unity3d.com/6000.0/Documentation/Manual/WhatsNewUnity6.html
- Release notes: https://unity.com/releases/editor/whats-new/6000.0.0f1
- Unity 6 releases: https://unity.com/releases/unity-6/support
- C# API reference: https://docs.unity3d.com/6000.0/Documentation/ScriptReference/index.html
