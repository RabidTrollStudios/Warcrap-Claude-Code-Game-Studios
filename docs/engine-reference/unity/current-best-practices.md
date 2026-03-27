# Unity 6.0 — Current Best Practices

**Last verified:** 2026-03-26
**Applies to:** Unity 6.0.3 (6000.0.35f2)

Modern Unity 6 patterns that may not be in the LLM's training data.

---

## Project Setup

### Use the Right Render Pipeline
- **URP (Universal)**: Mobile, cross-platform, good performance — recommended for most games
- **HDRP (High Definition)**: High-end PC/console, photorealistic
- **Built-in**: Avoid for new projects

### Use Build Profiles
Build Profiles replace the old Build Settings window. Create per-platform
configurations for different deployment scenarios.

---

## Object Discovery

Always use the new API — it's faster and the old one is obsolete:

```csharp
// Fastest: unordered
var enemies = Object.FindObjectsByType<Enemy>(FindObjectsSortMode.None);

// Fastest single: non-deterministic
var player = Object.FindAnyObjectByType<Player>();

// Deterministic single:
var player = Object.FindFirstObjectByType<Player>();
```

---

## Scripting (C# 9+)

Unity 6 supports C# 9 features:

```csharp
// Record types for data
public record PlayerData(string Name, int Level, float Health);

// Init-only properties
public class Config {
    public string GameMode { get; init; }
}

// Pattern matching
var result = enemy switch {
    Boss boss => boss.Enrage(),
    Minion minion => minion.Flee(),
    _ => null
};
```

---

## Input

### Use Input System Package (Not Legacy Input)

```csharp
using UnityEngine.InputSystem;

public class PlayerInput : MonoBehaviour {
    private PlayerControls controls;

    void Awake() {
        controls = new PlayerControls();
        controls.Gameplay.Jump.performed += ctx => Jump();
    }

    void OnEnable() => controls.Enable();
    void OnDisable() => controls.Disable();
}
```

Create Input Actions asset in editor, generate C# class via inspector.

---

## UI

### UI Toolkit for New Runtime UI

```csharp
using UnityEngine.UIElements;

public class MainMenu : MonoBehaviour {
    void OnEnable() {
        var root = GetComponent<UIDocument>().rootVisualElement;
        var playButton = root.Q<Button>("play-button");
        playButton.clicked += StartGame;
    }
}
```

UXML (structure) + USS (styling) = HTML/CSS-like workflow.
UGUI still works for existing projects — no need to migrate working UI.

---

## Rendering (URP)

### Use RenderGraph API for Custom Passes

```csharp
public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData) {
    using (var builder = renderGraph.AddRasterRenderPass<PassData>("My Pass", out var passData)) {
        builder.SetRenderFunc((PassData data, RasterGraphContext context) => {
            // Execute commands
        });
    }
}
```

### GPU Resident Drawer
Automatically batches compatible GameObjects with GPU instancing.
Enable it for scenes with many similar objects to reduce draw calls.

### STP Upscaling
Unity's built-in spatial-temporal post-processing upscaler — use it
before reaching for third-party solutions.

---

## Lighting

- Manually trigger baking: `Lightmapping.Bake()` or `Lightmapping.BakeAsync()`
- Progressive Lightmapper is the only baked GI option
- Generate lighting after any environment setting changes

---

## Asset Management

### Prefer Addressables Over Resources

```csharp
using UnityEngine.AddressableAssets;

public async Task SpawnEnemyAsync(string enemyKey) {
    var handle = Addressables.InstantiateAsync(enemyKey);
    var enemy = await handle.Task;
    Addressables.ReleaseInstance(enemy); // Cleanup
}
```

---

## Performance

### Burst Compiler + Jobs System

```csharp
[BurstCompile]
struct ParticleUpdateJob : IJobParallelFor {
    public NativeArray<float3> Positions;
    public NativeArray<float3> Velocities;
    public float DeltaTime;

    public void Execute(int index) {
        Positions[index] += Velocities[index] * DeltaTime;
    }
}
```

20-100x faster than equivalent managed C#.

### GPU Instancing

```csharp
Graphics.RenderMeshInstanced(
    new RenderParams(material),
    mesh,
    0,
    matrices // NativeArray<Matrix4x4>
);
```

---

## DOTS/ECS (If Using)

### Use ISystem (Not ComponentSystem)

```csharp
public partial struct MovementSystem : ISystem {
    public void OnUpdate(ref SystemState state) {
        foreach (var (transform, speed) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<MoveSpeed>>()) {
            transform.ValueRW.Position += speed.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}
```

### Use IJobEntity for Parallel Jobs

```csharp
[BurstCompile]
public partial struct DamageJob : IJobEntity {
    public float DeltaTime;
    void Execute(ref Health health, in DamageOverTime dot) {
        health.Value -= dot.DamagePerSecond * DeltaTime;
    }
}
```

---

## Multiplayer

- **Multiplayer Center** window: guided setup for networking
- **Netcode for GameObjects 2.0**: Distributed Authority mode (Beta)
- **Multiplayer Services**: consolidated Lobby, Relay, Matchmaker

---

## AI

**Unity Sentis** runs ML models at runtime — use for AI-driven gameplay
features instead of external inference servers.

---

## Profiling

- **Memory Profiler v1.1**: Enhanced tracking for RenderTextures, AudioClips, Shaders
- **Frame Debugger**: Shows shader keywords, individual meshes in SRP batches

---

## Summary: Unity 6 Tech Stack

| Feature | Use This (2025+) | Avoid This (Legacy) |
|---------|-------------------|---------------------|
| **Input** | Input System package | `Input` class |
| **UI** | UI Toolkit (new) / UGUI (existing) | `Text` component |
| **ECS** | ISystem + IJobEntity | ComponentSystem |
| **Rendering** | URP + RenderGraph | Built-in pipeline |
| **Assets** | Addressables | Resources.Load |
| **Jobs** | Burst + IJobParallelFor | Coroutines for heavy work |
| **Multiplayer** | Netcode for GameObjects | UNet |
| **Objects** | FindObjectsByType | FindObjectsOfType |

---

**Sources:**
- https://docs.unity3d.com/6000.0/Documentation/Manual/BestPracticeGuides.html
- https://docs.unity3d.com/6000.0/Documentation/Manual/WhatsNewUnity6.html
