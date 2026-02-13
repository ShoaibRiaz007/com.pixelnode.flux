# com.pixelnode.flux
Unity's lightweight, zero-GC tween animation framework built around a node-based architecture. Designed for production use where performance matters. No garbage collection spikes, O(1) operations, and a fluent API that stays out of your way. Inspired by GameCreator2
---

## Key Features

- **Zero-GC runner** -- Slot-array architecture with generation-safe handles. No dictionaries in the hot path. Pre-allocated, pooled, cache-friendly. Profiler-instrumented.
- **Node-based** -- Every animation is a composable `AnimationNode`. Sequence them, run them in parallel, loop them, or gate them with conditions.
- **Fluent API** -- Chain animations with `.Then()` and `.Join()`, or use `DoX()` one-liners for quick fire-and-forget tweens.
- **Relative tweens** -- `DoMoveBy()`, `DoScaleBy()`, `DoRotateBy()` -- animate relative to current values without computing start/end manually.
- **Async/await** -- `await handle.ToAwaitable()` with zero heap allocation. No Task, no TaskCompletionSource.
- **Lifetime safety** -- `handle.LinkLifetime(gameObject)` auto-stops animations when the owning object is disabled or destroyed.
- **Unscaled time** -- Per-animation option via `Play(node, useUnscaledTime: true)`. Pause menus and gameplay animations run simultaneously.
- **Multiple runners** -- `AnimationRunner.Create()` or `AnimationRunner.CreateFor(canvas)` for per-panel isolation.
- **Editor integration** -- Custom inspector with hierarchical type picker, inline timeline preview, copy/paste, and per-node play/stop buttons.
- **Condition system** -- Built-in condition evaluators (iteration count, active state, destroyed, float comparison, logical AND/OR/NOT) that plug into control flow nodes.
- **Custom curves** -- Select `CustomCurve` easing and assign an `AnimationCurve` directly in the inspector.
- **Extensible** -- Add new animation types or condition types by subclassing and adding attributes. The editor and runner discover them automatically.

---

## Architecture

```
AnimationNode (base class)
  |-- Apply(t)          -- override to animate a property
  |-- Timing            -- duration, delay, easing, custom curve
  +-- State machine     -- Idle -> Playing -> Completed/Stopped/...

AnimationRunner (MonoBehaviour)
  |-- Slot[] array      -- pre-allocated, pooled per-type
  |-- Dense active list -- cache-friendly tick loop
  |-- Generation handles -- stale-handle safe
  |-- Per-slot callbacks -- OnComplete / OnUpdate
  |-- ProfilerMarkers   -- Tick, Play, Stop visible in Unity Profiler
  +-- Factory methods   -- Create(), CreateFor(parent)

AnimationHandle (struct)
  +-- Packed int: slot index + generation (zero-alloc)
```

---

## Quick Start

### From Code (fluent API)

```csharp
// Simple fade
canvasGroup.DoFadeIn(duration: 0.3f);

// Anchored position tween
rectTransform.DoAnchoredPosition(
    start: new Vector2(0, -100),
    end: Vector2.zero,
    duration: 0.5f,
    ease: Ease.Type.EaseOut
);

// Compose a sequence
var handle = rectTransform
    .AnchoredPositionNode(startPos, endPos, 0.3f)
    .Join(canvasGroup.FadeNode(true, 0.25f))        // parallel
    .Then(rectTransform.ScaleNode(                   // then
        Vector3.one * 0.9f, Vector3.one, 0.15f))
    .Play(
        onComplete: () => Debug.Log("Done!")
    );

// Control it
handle.Pause();
handle.Resume();
handle.Stop();

// Query
if (handle.IsPlaying()) { ... }
float progress = handle.GetProgress();
```

### Relative Tweens

```csharp
// Move 100px to the right from current position
rectTransform.DoMoveBy(new Vector2(100, 0), 0.3f);

// Scale up 20% from current scale
transform.DoScaleBy(Vector3.one * 1.2f, 0.2f);

// Rotate 90 degrees from current rotation
transform.DoRotateBy(new Vector3(0, 0, 90), 0.4f);

// Fade down by 0.3 from current alpha
canvasGroup.DoFadeBy(-0.3f, 0.25f);
```

### Async/Await

```csharp
async void ShowCard()
{
    // Slide in and wait for completion -- zero heap allocation
    await rectTransform.DoMoveBy(new Vector2(0, 200), 0.3f).ToAwaitable();

    // Chain sequential awaits
    await canvasGroup.DoFadeIn(0.2f).ToAwaitable();
    await transform.DoScaleBy(Vector3.one * 1.1f, 0.15f).ToAwaitable();

    Debug.Log("All animations done");
}
```

### Lifetime Safety

```csharp
// Auto-stop when the GameObject is disabled or destroyed
var handle = canvasGroup.DoFadeIn(0.5f);
handle.LinkLifetime(gameObject);

// Or chain it fluently
canvasGroup.DoFadeIn(0.5f).LinkLifetime(gameObject);
```

### Unscaled Time (pause menus)

```csharp
// This animation runs even when Time.timeScale = 0
var handle = node.Play(useUnscaledTime: true);
```

### Multiple Runners

```csharp
// Create a runner scoped to a specific Canvas -- destroyed with it
IAnimationRunner panelRunner = AnimationRunner.CreateFor(myCanvas.transform);
var handle = panelRunner.Play(myNode);

// Or create a standalone runner
IAnimationRunner standalone = AnimationRunner.Create("MyRunner");
```

### From the Inspector

1. Add a `[SerializeReference] IAnimationNode` field to any MonoBehaviour.
2. In the inspector, click the type selector to pick an animation node (or composite).
3. Configure properties, hit play to preview.

```csharp
public class MyUI : MonoBehaviour
{
    [SerializeReference] IAnimationNode showAnimation;
    AnimationHandle _handle;

    void OnEnable()
    {
        _handle = showAnimation.Play();
        _handle.LinkLifetime(gameObject);
    }

    void OnDisable()
    {
        _handle.Stop();
    }
}
```

---

## Built-in Animation Nodes

| Category | Node | Description |
|---|---|---|
| **UI / RectTransform** | RectAnchoredPosition | Tween anchored position |
| | RectAnchorMin / Max | Tween anchor boundaries |
| | RectPivot | Tween pivot |
| | RectSizeDelta | Tween size delta |
| **UI / Canvas Group** | CanvasGroupAlpha | Fade in / out with interactable control |
| **UI / Graphic** | GraphicColor | Tween color on any Graphic (Image, Text) |
| | GraphicAlpha | Tween alpha on any Graphic |
| **Transform** | ScaleAnimation | Tween local scale |
| | RotationAnimation | Tween local euler rotation |
| | PositionAnimation | Tween local position |
| **GameObject** | SetActive | Activate / deactivate at end of duration |
| **Sequence** | SequentialAnimation | Play children one after another |
| | ParallelAnimation | Play children simultaneously |
| **Control Flow** | WaitNode | Wait for a duration (no side effects) |
| | RepeatUntilNode | Loop an animation until a condition is met |
| | ConditionalNode | Play animation only if condition is true |
| | WaitForConditionNode | Wait until a condition becomes true |

---

## Condition System

Conditions are used by control flow nodes to make decisions at runtime.

| Condition | Description |
|---|---|
| IterationCountCondition | True while iteration < max |
| IsActiveCondition | Checks GameObject active state |
| IsDestroyedCondition | Checks if GameObject is destroyed |
| CompareFloatCondition | Float comparison (< > ==) |
| LogicalCondition | Combines conditions with AND / OR / NOT |

Conditions are selected in the inspector via the same hierarchical popup as animation nodes.

---

## Easing

Built-in easing curves via `Ease.Type`:

Linear, EaseIn, EaseOut, EaseInOut, QuadIn/Out/InOut, CubicIn/Out/InOut, ExpoIn/Out/InOut, CustomCurve.

When `CustomCurve` is selected, an `AnimationCurve` field appears in the inspector for full manual control.

---

## Runner Details

The `AnimationRunner` is a zero-GC slot-based system:

- **Play** -- Acquires a slot (from per-type pool or free list), reinitializes the node, returns a packed handle. O(1).
- **Stop** -- Validates generation, fires callback, swap-removes from dense array, returns slot to pool. O(1).
- **Tick** -- Iterates the dense active array. No dictionary lookups. Per-animation unscaled time support.
- **Handles** -- `AnimationHandle` is a readonly struct packing slot index + generation into a single int. Stale handles silently no-op.
- **Pooling** -- Nodes are pooled per-type in their slots. After warmup, Play/Stop never allocate.
- **Profiling** -- `ProfilerMarker` on Tick, Play, and Stop. Shows up in the Unity Profiler with zero overhead when not profiling.
- **Factory** -- `AnimationRunner.Create()` for standalone runners, `CreateFor(parent)` for scoped runners destroyed with their parent.

Access via `AnimationRunner.DefaultRunner` (auto-creates, survives scene loads, recreates if destroyed).

---

## Extending

### Add a new animation node

```csharp
[Title("My Tween")]
[Description("Tweens something custom.")]
[Category("Custom")]
[Serializable]
public class MyTween : AnimationNode
{
    [SerializeField] private Transform target;
    [SerializeField] private float startValue;
    [SerializeField] private float endValue;

    public override string Title => "My Tween";

    public MyTween() : base() { }

    protected override void Apply(float t)
    {
        if (target == null) return;
        // Apply your interpolation here
    }

    protected override void CopyFrom(AnimationNode source)
    {
        var other = source as MyTween;
        if (other == null) return;
        target = other.target;
        startValue = other.startValue;
        endValue = other.endValue;
    }
}
```

Register it in `AnimationNodeRegistration.RegisterBuiltInTypes()` or let the auto-discovery find it at startup.

### Add a new condition

```csharp
[Title("My Condition")]
[Description("Checks something custom.")]
[Category("Condition")]
[Serializable]
public class MyCondition : ConditionEvaluator
{
    public override string Title => "My Condition";
    public override string Description => "Custom check";
    public override bool Evaluate() => /* your logic */;
}
```

Both will appear automatically in the hierarchical type picker.

---

## API Reference (Extension Methods)

### Absolute Tweens (DoX)

| Method | Target | Parameters |
|---|---|---|
| `DoAnchoredPosition` | RectTransform | start, end |
| `DoAnchorMin` / `DoAnchorMax` | RectTransform | start, end |
| `DoSizeDelta` | RectTransform | start, end |
| `DoPivot` | RectTransform | start, end |
| `DoFadeIn` / `DoFadeOut` | CanvasGroup | -- |
| `DoScale` | Transform | start, end |
| `DoRotation` | Transform | startEuler, endEuler |
| `DoPosition` | Transform | start, end |
| `DoColor` | Graphic | startColor, endColor |
| `DoAlpha` | Graphic | startAlpha, endAlpha |

### Relative Tweens (DoXBy)

| Method | Target | Parameters |
|---|---|---|
| `DoMoveBy` | RectTransform | offset |
| `DoScaleBy` | Transform | multiplier |
| `DoRotateBy` | Transform | eulerDelta |
| `DoPositionBy` | Transform | offset |
| `DoFadeBy` | CanvasGroup | alphaDelta |
| `DoColorBy` | Graphic | colorDelta |

### Handle Operations

| Method | Returns | Description |
|---|---|---|
| `handle.Stop()` | void | Stop and return to pool |
| `handle.Pause()` | handle | Pause playback |
| `handle.Resume()` | handle | Resume playback |
| `handle.IsPlaying()` | bool | Check if still active |
| `handle.GetProgress()` | float | Normalized progress 0-1 |
| `handle.OnComplete(cb)` | handle | Register completion callback |
| `handle.OnUpdate(cb)` | handle | Register per-frame callback |
| `handle.LinkLifetime(go)` | handle | Auto-stop on disable/destroy |
| `handle.ToAwaitable()` | awaitable | Zero-alloc async/await |

### Composition

| Method | Description |
|---|---|
| `node.Then(next)` | Sequential: play first, then next |
| `node.Join(other)` | Parallel: play both simultaneously |
| `node.Play()` | Fire on default runner |

---

## Project Structure

```
com.pixelnode.flux/
|-- Editor/                        -- Inspector, type picker, drawers
|-- Runtime/
|   |-- Animation Handler/         -- Runner, interface, awaiter, extensions
|   |-- Animations/                -- Built-in node types
|   |   |-- Control Flow/          -- Wait, Repeat, Conditional
|   |   |-- Sequence/              -- Sequential, Parallel
|   |   |-- Transform/             -- Scale, Rotation, Position
|   |   |-- UI/                    -- RectTransform, CanvasGroup, Graphic
|   |   +-- Extension/             -- Fluent API, composition, relative tweens
|   |-- Attribute/                 -- Title, Description, Category, Button
|   |-- base/                      -- AnimationNode, Handle, State, Interface
|   |-- Condition/                 -- ConditionEvaluator + built-in conditions
|   +-- Utility/                   -- Easing, Registry, Clone, JSON, Cleanup
+-- Sample/                        -- Example scripts
```
