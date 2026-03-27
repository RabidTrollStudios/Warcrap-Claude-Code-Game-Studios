# Unity 6.0 — Deprecated APIs

**Last verified:** 2026-03-26
**Applies to:** Unity 6.0.3 (6000.0.35f2)

Quick lookup table for deprecated APIs and their replacements.
Format: **Don't use X** → **Use Y instead**

---

## Object Discovery

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `Object.FindObjectsOfType<T>()` | `Object.FindObjectsByType<T>(FindObjectsSortMode.None)` | `None` = unsorted (faster) |
| `Object.FindObjectOfType<T>()` | `Object.FindFirstObjectByType<T>()` | Deterministic |
| `Object.FindObjectOfType<T>()` | `Object.FindAnyObjectByType<T>()` | Non-deterministic but faster |

---

## Graphics Formats (Compile Error)

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `GraphicsFormat.DepthAuto` | `GraphicsFormat.None` | Check depth with `depthStencilFormat != GraphicsFormat.None` |
| `GraphicsFormat.ShadowAuto` | `GraphicsFormat.None` + `shadowSamplingMode = CompareDepths` | Set on `RenderTextureDescriptor` |
| `GraphicsFormat.VideoAuto` | `GraphicsFormat.None` | |

---

## Lightmapper Properties

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `filteringGaussRadiusAO` (int) | `filteringGaussianRadiusAO` (float) | Note spelling: Gauss**ian** |
| `filteringGaussRadiusDirect` (int) | `filteringGaussianRadiusDirect` (float) | |
| `filteringGaussRadiusIndirect` (int) | `filteringGaussianRadiusIndirect` (float) | |

---

## Render Pipeline

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `SetupRenderPasses` | `AddRenderPasses` (render graph) | Rewrite Scriptable Renderer Features |
| `ScriptableRenderPass.Execute()` | `RecordRenderGraph()` | Use RenderGraph API |
| `CustomEditorForRenderPipelineAttribute` | `CustomEditor` + `SupportedOnRenderPipelineAttribute` | Combine attributes |
| `VolumeComponentMenuForRenderPipelineAttribute` | `VolumeComponentMenu` + `SupportedOnRenderPipelineAttribute` | |
| `FetchFirstCompatibleType...Extension` | `GetDerivedTypesSupportedOnCurrentPipeline` | Returns all types |

---

## UI Toolkit

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `ExecuteDefaultAction()` | `HandleEventBubbleUp()` | |
| `ExecuteDefaultActionAtTarget()` | `HandleEventBubbleUp()` | AtTarget phase removed |
| `PreventDefault()` | `StopPropagation()` | |
| `UxmlTraits` / `UxmlFactory` | `[UxmlElement]` / `[UxmlAttribute]` | Source generator approach |

---

## Input

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `Input.GetKey()` | `Keyboard.current[Key.X].isPressed` | Input System package |
| `Input.GetKeyDown()` | `Keyboard.current[Key.X].wasPressedThisFrame` | Input System package |
| `Input.GetMouseButton()` | `Mouse.current.leftButton.isPressed` | Input System package |
| `Input.GetAxis()` | `InputAction` callbacks | Input System package |
| `Input.mousePosition` | `Mouse.current.position.ReadValue()` | Input System package |

---

## Lighting

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| Enlighten Baked GI | Progressive Lightmapper | Auto-substituted on upgrade |
| Auto Generate lighting | `Lightmapping.Bake()` / `BakeAsync()` | Manual trigger required |

---

## UI (UGUI)

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `Canvas` (UGUI) | `UIDocument` (UI Toolkit) | UGUI still works, UI Toolkit recommended for new |
| `Text` component | `TextMeshPro` or UI Toolkit `Label` | |

---

## DOTS/Entities

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `ComponentSystem` | `ISystem` (unmanaged) | Entities 1.0+ |
| `JobComponentSystem` | `ISystem` with `IJobEntity` | Burst-compatible |
| `GameObjectEntity` | Pure ECS workflow | |
| `ComponentDataFromEntity<T>` | `ComponentLookup<T>` | Entities 1.0+ rename |
| `Entities.ForEach` | `IJobEntity` / `SystemAPI.Query` | Marked obsolete |

---

## Asset Loading

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `Resources.Load()` | Addressables | Better memory control, async |
| `WWW` class | `UnityWebRequest` | Modern async networking |

---

## Social & Platform

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `Social` API (ISocialPlatform) | Platform-specific SDKs | Deprecated in Unity 6 |

---

## Android

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `UnityPlayer` (Java class) | `UnityPlayerForActivityOrService` | |
| `UnityPlayer` as `FrameLayout` | `unityPlayer.getFrameLayout()` | No longer extends FrameLayout |

---

## Scene Management

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `Application.LoadLevel()` | `SceneManager.LoadScene()` | Long deprecated |

---

**Sources:**
- https://docs.unity3d.com/6000.0/Documentation/Manual/UpgradeGuideUnity6.html
- https://docs.unity3d.com/6000.0/Documentation/Manual/WhatsNewUnity6.html
