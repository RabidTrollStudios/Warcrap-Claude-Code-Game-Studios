# Unity 6.0 — Breaking Changes

**Last verified:** 2026-03-26
**Applies to:** Unity 6.0.3 (6000.0.35f2)

This document tracks breaking API changes and behavioral differences between
Unity 2022 LTS (likely in model training) and Unity 6.0 (current version).

---

## HIGH RISK — Will Break Existing Code

### Object Discovery API Removed

```csharp
// OLD (compile warning, will be removed)
var enemies = Object.FindObjectsOfType<Enemy>();
var player = Object.FindObjectOfType<Player>();

// NEW (Unity 6.0+)
var enemies = Object.FindObjectsByType<Enemy>(FindObjectsSortMode.None);
var player = Object.FindFirstObjectByType<Player>();
// Or for non-deterministic (faster):
var player = Object.FindAnyObjectByType<Player>();
```

**Migration:** Replace all `FindObjectsOfType`/`FindObjectOfType` calls.
Use `FindObjectsSortMode.None` for best performance when order doesn't matter.

---

### Graphics Format Enums Obsoleted

```csharp
// OLD (compile error in Unity 6)
var desc = new RenderTextureDescriptor { graphicsFormat = GraphicsFormat.DepthAuto };

// NEW
var desc = new RenderTextureDescriptor {
    graphicsFormat = GraphicsFormat.None,
    depthStencilFormat = GraphicsFormat.D32_SFloat
};
// For shadows: desc.shadowSamplingMode = ShadowSamplingMode.CompareDepths;
```

**Affected:** `GraphicsFormat.DepthAuto`, `ShadowAuto`, `VideoAuto` — all produce compile errors.

---

### URP Render Pass API Changed

```csharp
// OLD: ScriptableRenderPass.Execute
public override void Execute(ScriptableRenderContext context, ref RenderingData data)

// NEW: RenderGraph API
public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData)
```

**Migration:** Rewrite custom render passes to use RenderGraph API.
`SetupRenderPasses` is deprecated — use `AddRenderPasses` instead.

---

### Enlighten Baked GI Removed

Unity auto-substitutes Progressive Lightmapper (GPU on Apple Silicon, CPU elsewhere).
Enlighten Realtime GI remains supported through Unity 6.

---

### Auto Generate Lighting Removed

```csharp
// OLD: Auto-generate in Lighting window (removed)

// NEW: Manual trigger
Lightmapping.Bake();        // Synchronous
Lightmapping.BakeAsync();   // Asynchronous
// Or use "Generate Lighting" button in Lighting window
```

---

### Input System — Legacy Input Deprecated

```csharp
// OLD (deprecated)
if (Input.GetKeyDown(KeyCode.Space)) { }

// NEW (Input System package)
using UnityEngine.InputSystem;
if (Keyboard.current.spaceKey.wasPressedThisFrame) { }
```

**Migration:** Install `com.unity.inputsystem` package, replace all `Input.*` calls.

---

### DOTS/Entities API Overhaul

```csharp
// OLD (pre-Unity 6)
public class DamageSystem : ComponentSystem {
    protected override void OnUpdate() { }
}

// NEW (Entities 1.0+)
public partial struct DamageSystem : ISystem {
    public void OnCreate(ref SystemState state) { }
    public void OnUpdate(ref SystemState state) {
        foreach (var (health, dot) in
            SystemAPI.Query<RefRW<Health>, RefRO<DamageOverTime>>()) {
            health.ValueRW.Value -= dot.ValueRO.DPS * SystemAPI.Time.DeltaTime;
        }
    }
}
```

| Old API | New API |
|---------|---------|
| `ComponentSystem` | `ISystem` (unmanaged) |
| `JobComponentSystem` | `ISystem` with `IJobEntity` |
| `GameObjectEntity` | Pure ECS workflow |
| `ComponentDataFromEntity<T>` | `ComponentLookup<T>` |
| `Entities.ForEach` | `IJobEntity` or `SystemAPI.Query` |

---

## MEDIUM RISK — Behavioral Changes

### Lighting Brightness

Light probe brightness increased to match lightmap brightness (previously 94%).
If you need old behavior, multiply coefficient 0 of each `SphericalHarmonicsL2`
instance by 16/17.

---

### Ambient Probe No Longer Auto-Bakes

Ambient probe and skybox Reflection Probe no longer bake by default.
Select "Generate Lighting" in the Lighting window when environment settings change.

---

### Mipmap Limits Now Opt-In

For runtime-created 2D textures, mipmap limits are opt-in (was opt-out).
Use `Texture2D` constructors with `MipmapLimitDescriptor` to enable.

---

### Gaussian Filter Radius Type Change

Properties changed from `int` to `float`:
- `filteringGaussRadiusAO` → `filteringGaussianRadiusAO`
- `filteringGaussRadiusDirect` → `filteringGaussianRadiusDirect`
- `filteringGaussRadiusIndirect` → `filteringGaussianRadiusIndirect`

---

### Metal Shader Buffer Layout

Buffer layout changed for Metal shaders using min16float, half, or real types.
Verify generated MSL matches C# buffer data if writing directly to buffers.

---

## LOW RISK — Deprecations (Still Functional)

### UGUI (Legacy UI)
**Status:** Deprecated but supported. UI Toolkit recommended for new projects.

### Legacy Particle System
**Status:** Deprecated. Visual Effect Graph (VFX Graph) recommended.

### Build Settings Window
**Status:** Replaced by Build Profiles. Old workflow still accessible but not default.

---

## Platform-Specific Breaking Changes

### Android
- `UnityPlayer` Java class → `UnityPlayerForActivityOrService` / `UnityPlayerForGameActivity`
- `UnityPlayer` no longer extends `FrameLayout` — use `getFrameLayout()`
- Default tools: Gradle 8.11, AGP 8.7.2, NDK r27c, JDK 17
- GameActivity is default entry point

### iOS
- `.xcframework` plugin support added
- 64-bit ARM + x86_64 simulator support

### Web
- WebAssembly 2023: 4GB heap, native exceptions, SIMD

---

## UI Toolkit Event Handling

```csharp
// OLD
protected override void ExecuteDefaultAction(EventBase evt) { }
protected override void ExecuteDefaultActionAtTarget(EventBase evt) { }
evt.PreventDefault();

// NEW
protected override void HandleEventBubbleUp(EventBase evt) { }
// AtTarget phase removed entirely
evt.StopPropagation();  // replaces PreventDefault
```

Custom controls: Replace `UxmlTraits`/`UxmlFactory` with `[UxmlElement]`/`[UxmlAttribute]`.

---

## Editor Changes

- **Build Profiles** replace Build Settings — create per-platform configs
- **Assets/Create menu** reorganized — verify `ExecuteMenuItem` paths
- **ScriptTemplates** renamed — update custom template overrides
- **ProBuilder** moved to tool context system (no standalone window)

---

**Sources:**
- https://docs.unity3d.com/6000.0/Documentation/Manual/UpgradeGuideUnity6.html
- https://docs.unity3d.com/6000.0/Documentation/Manual/WhatsNewUnity6.html
