# MotionForge — Architecture Overview

> **Status of the 0.2 build.** Implemented: sections 1–4 for the layer compositor, video decode, expressions (6.7), the displace + chromatic aberration pass (6.3), MP4 export (8). The Core ML bridge (`NeuralStage.swift`) is written but not connected, and no models are bundled. Everything else in sections 6 and 7 is design only. See `README.md` for the exact list.

Native iOS / iPadOS motion-graphics app. SwiftUI shell, Metal compositor, Core ML neural stages on the Apple Neural Engine (ANE).
Target: iOS 17+, A14 / M1 or newer. Swift 5 language mode (Swift 6 strict concurrency needs annotations on the Core ML hand-off).

---

## 0. Scope, honesty and licensing

| Topic | Position |
|---|---|
| What is in the box | A vertical slice: dark UI, Metal preview with split compare, pinch-zoom timeline, Bézier easing editor, the fused Displace + Chromatic Aberration kernel, and a zero-copy Core ML bridge. Everything else in §6–§7 is specified, not implemented. |
| Not compiled here | The code was written without access to Xcode. Expect a handful of compiler fixes on first build. |
| Full After Effects parity | A multi-year programme. §10 gives a staged order that ships value early. |
| Topaz-class models | Topaz's models are proprietary and are not available to copy. Ship your own Core ML models, trained or converted from open research. Verify each licence yourself before shipping: RIFE (MIT), Real-ESRGAN (BSD-3) and GFPGAN (Apache-2.0) are permissive; CodeFormer's licence is non-commercial. |
| Names | Do not ship features named after Adobe or Topaz products or trademarks. Use your own (`Face Restore`, `Frame Synthesis`, `Detail Upscale`). |
| "Real-time" neural | Not at full resolution. Preview runs neural stages on a half or quarter-res proxy, with cached results. Final render runs them tiled, offline. |
| "No ghosting" | Cannot be guaranteed. Interpolation quality depends on the model. Occlusion handling comes from the model's fusion/mask output, with a fall-back to frame blending where flow confidence is low. |
| MLX | MLX runs on the GPU, not the ANE. Use Core ML for ANE. MLX is useful for prototyping models on a Mac. |

---

## 1. Layer map

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ SwiftUI layer (main actor)                                                   │
│  RootView ─ TopBar ─ PreviewStage ─ PropertyDrawer ─ TimelineDock ─ Sheets   │
│  @Observable ProjectModel  (document truth: layers, keyframes, fx, neural)   │
└───────────────┬──────────────────────────────────────────────────────────────┘
                │ value-type snapshot per frame:  RenderGraph (immutable, Sendable)
┌───────────────▼──────────────────────────────────────────────────────────────┐
│ Engine layer (render queue, off main)                                        │
│  Evaluator   : keyframes -> easing LUT -> expressions (JavaScriptCore) -> values │
│  Scheduler   : decides cache hit / re-render / neural ticket per frame       │
│  Compositor  : walks the graph, encodes Metal passes into MTLCommandBuffers  │
└───────┬─────────────────────────────┬────────────────────────────────────────┘
        │ MTLTexture (rgba16Float)    │ IOSurface-backed CVPixelBuffer (BGRA8)
┌───────▼───────────┐        ┌────────▼─────────────┐        ┌─────────────────┐
│ GPU: Metal        │◄──────►│ ANE: Core ML         │        │ Media I/O       │
│ compute + render  │ shared │ Face / Interp / Up   │        │ AVFoundation,   │
│ MTLHeap pools     │ memory │ .cpuAndNeuralEngine  │        │ VideoToolbox    │
└───────────────────┘        └──────────────────────┘        └─────────────────┘
```

Rule: **SwiftUI never touches a texture and the engine never touches a View.** They communicate through `ProjectModel` (down) and a small `PreviewState` (frame-ready callback + scopes, up).

---

## 2. SwiftUI layer tree

```
RootView
├─ TopBar                          .ultraThinMaterial, 40 pt icon buttons, PressableStyle (spring on press)
├─ GeometryReader
│  ├─ iPad regular width:   HStack { PreviewStage | PropertyDrawer(330 pt) }
│  └─ iPhone / compact:     ZStack { PreviewStage ; PropertyDrawer as bottom sheet (≤ 60 % height) }
│       PreviewStage
│       ├─ MetalPreview (UIViewRepresentable -> MTKView, on-demand draw)
│       └─ CompareDivider  (drag; sends splitX uniform to the present shader)
├─ TimelineDock                    detents 68 / 290 / 470 pt, spring-snapped, VoiceOver adjustable
│  ├─ TransportBar                 play (CADisplayLink clock), frame step, timecode, add-layer menu
│  └─ TrackArea                    pinned labels | horizontal ScrollView { Ruler, ClipViews, Playhead }
│                                  MagnifyGesture zoom, highPriority scrub on the ruler
└─ .sheet EasingSheet              BezierGraphEditor, detents 600 / large
```

Motion language (all `interpolatingSpring`, defined once in `Animation.motionPanel` and `.motionSnap`):

| Interaction | Animation |
|---|---|
| Drawer open/close, section expand | `motionPanel` (mass 1, k 260, c 24), slight settle |
| Press, toggle, handle grab | `motionSnap` (mass 0.8, k 420, c 28), no visible bounce |
| Timeline detent release | `motionPanel` keyed on `detent`, finger-tracking during drag |
| Playback | **Never** SwiftUI animation. `CADisplayLink` advances `project.time`; Metal redraws. |

Accessibility baseline in the code: 44 pt hit targets, labels on icon buttons, adjustable actions on the ruler, timeline resize and Bézier handles, Reduce Motion respected in the easing preview.

**Redraw strategy.** `MTKView` runs `isPaused = true` with `enableSetNeedsDisplay`. `updateUIView` reads `time`, `fx` and `neural`, which registers an Observation dependency, then calls `setNeedsDisplay()`. During playback `isPaused = false`. Idle app = 0 GPU work.

---

## 3. Per-frame data flow

1. `PlaybackClock` (or a scrub gesture) writes `project.time` on the main actor.
2. Main actor builds `RenderGraph` (value copy: layer order, transforms, effect stacks, matte links, camera, lights, easing LUTs). Copy cost is O(layers), not O(pixels).
3. Engine queue evaluates properties at `t`:
   `value = lerp(kf[i], kf[i+1], easingLUT[(t-t_i)/(t_{i+1}-t_i)])`, then runs expressions (§6.7).
4. Compositor allocates offscreen targets from a pool (`MTLHeap`, rgba16Float, `.private`).
5. For each layer in z-order (3D layers sorted by view-space depth):
   source texture → effect chain (compute passes) → matte multiply → blend into the accumulation target.
6. Optional neural stage (§7) inserted at a defined point in the chain; result returned through the shared event.
7. `present_fragment` converts linear → display and draws the compare split.
8. The completed handler releases the in-flight semaphore (3 frames in flight).

---

## 4. Memory and synchronisation on Apple Silicon

* **Unified memory.** CPU, GPU and ANE use the same DRAM. "Zero-copy" means passing the same physical pages, not copying across a bus.
* **Shared surfaces.** A `CVPixelBuffer` created with `kCVPixelBufferIOSurfacePropertiesKey` and `kCVPixelBufferMetalCompatibilityKey`. A `CVMetalTextureCache` wraps it as an `MTLTexture`; Core ML reads it via `MLFeatureValue(pixelBuffer:)` and writes its result into a second buffer via `MLPredictionOptions.outputBackings`.
* **Pixel formats.** Compositor works in linear-light `rgba16Float` (scene-referred, Display P3 primaries). Core ML image I/O is 8-bit BGRA, so the bridge adds two tiny kernels: `encodeForML` (linear → sRGB/Display-P3 OETF → BGRA8) and `decodeFromML` (the inverse). Cost: banding in deep gradients and loss of HDR headroom. Upgrade path: models taking `MLMultiArray` fp16 backed by a `OneComponent16Half` pixel buffer (iOS 16+), or tone-map-then-restore HDR with a gain map.
* **Ordering.** The ANE is not a Metal queue, so command buffers cannot wait on it directly:
  1. GPU command buffer A encodes `encodeForML` → `addCompletedHandler`.
  2. Handler hops to a detached task → `await model.prediction(...)` → sets `MTLSharedEvent.signaledValue = ticket`.
  3. GPU command buffer B calls `encodeWaitForEvent(event, ticket)` and then `decodeFromML`.
  *Always signal, even on failure*, or queue B stalls forever.
* **Do not stall the preview.** Because B blocks the serial queue until the ANE finishes, preview uses **latency hiding**: show the result for frame N−k, and keep un-neural frames flowing on a second `MTLCommandQueue`. Export uses a strict pipeline with no latency hiding.
* **Residency.** `MTLHeap` with `.private` storage for intermediates, aliasing for non-overlapping passes, `makeAliasable()` between passes. Static textures (displacement maps, LUTs) use `.shared` storage and are uploaded once.
* **Thermal.** Observe `ProcessInfo.thermalState`; at `.serious` drop preview to half-res and pause neural preview.

---

## 5. Engine data model

```swift
struct RenderGraph: Sendable {
    var time: Double
    var comp: CompSettings                      // size, fps, color space
    var layers: [LayerSnapshot]                 // bottom -> top
    var camera: CameraSnapshot?                 // nil = flat 2D
    var lights: [LightSnapshot]
}
struct LayerSnapshot: Sendable {
    var id: UUID; var kind: LayerKind
    var transform: simd_float4x4                // already evaluated at `time`
    var opacity: Float; var blend: BlendMode
    var matte: (source: UUID, mode: MatteMode)?
    var effects: [EffectInstance]               // typed param blocks, already evaluated
    var is3D: Bool; var castsShadow: Bool; var acceptsLights: Bool
}
```

Properties are stored as `[Keyframe<T>]` with an index into a shared easing-LUT table (256 floats per curve; exported by `CubicBezier.lut()`), so interpolation is identical on CPU (preview scrub) and GPU (batched particle/expression-free paths).

---

## 6. After Effects feature suite

### 6.1 3D camera and lights

```swift
func perspective(fovY: Float, aspect: Float, near: Float, far: Float) -> simd_float4x4 {
    let y = 1 / tan(fovY * 0.5), x = y / aspect
    let z = far / (near - far)                     // Metal clip space: z in [0, 1], right-handed view
    return simd_float4x4(columns: (SIMD4(x, 0, 0, 0), SIMD4(0, y, 0, 0),
                                   SIMD4(0, 0, z, -1), SIMD4(0, 0, z * near, 0)))
}
```
* AE camera "Zoom" in pixels maps to FOV: `fovY = 2·atan(compHeight / (2·zoom))`.
* Layers become textured quads transformed by `P · V · M`. Depth is written, so intersecting planes sort correctly; transparent layers sort back-to-front.
* **Lights:** ambient, parallel (directional), point, spot (cone angle + feather). Per-layer material options (accepts lights, casts shadows, ambient/diffuse/specular) evaluated in the layer fragment shader.
* **Shadows:** one depth map per shadow-casting light (`depth32Float`, 2048²), PCF 3×3 filtering, slope-scaled bias. Point lights use a cube map (6 passes; limit to 2 at a time on phones).
* **Depth of field:** thin-lens circle of confusion
  `CoC = A · f · |d − s| / (d · (s − f))` (A aperture, f focal length, s focus distance, d fragment depth).
  Implement as a gather blur on the composited image with the depth buffer: bokeh disc of 16–32 golden-angle taps scaled by CoC, with a near-field/far-field split to avoid foreground bleeding.

### 6.2 Track mattes and blend modes

* **Mattes:** the matte layer renders to its own target; the target layer multiplies by the matte factor before blending.

  | Mode | Factor |
  |---|---|
  | Alpha | `m.a` |
  | Alpha Inverted | `1 − m.a` |
  | Luma | `luma(m.rgb/m.a) · m.a` |
  | Luma Inverted | `1 − luma(m.rgb/m.a) · m.a` |

* **Blend modes (27):** Normal, Dissolve, Darken, Multiply, Color Burn, Linear Burn, Darker Color, Add, Lighten, Screen, Color Dodge, Lighter Color, Overlay, Soft Light, Hard Light, Linear Light, Vivid Light, Pin Light, Hard Mix, Difference, Exclusion, Subtract, Divide, Hue, Saturation, Color, Luminosity.
  Implemented as one `blendLayer` compute kernel specialised with a Metal **function constant** per mode (27 tiny pipelines, no divergent branching), operating on un-premultiplied colour in linear light. Dissolve uses a per-pixel hash against opacity.

### 6.3 Effects engine (compute shaders)

All effects share one contract: `(input, optional aux textures, params block) → output`, same size, rgba16Float, premultiplied.

| Effect | Technique |
|---|---|
| Displacement Map, Turbulent Displace, Chromatic Aberration | **Implemented**: `displaceChromaticKernel` (fused: one read per spectral tap, one write). |
| Wave Warp | UV offset by `A·sin(2π(x·cosθ + y·sinθ)/λ + φ)`; sine, square, triangle, sawtooth variants. |
| Glow / Bloom | Threshold → Kawase down/up chain (5 levels, offsets 0,1,2,2,3), additive composite, per-channel tint. |
| Saber-style edge glow | Sobel edge map → thin core + two Kawase halos → additive; animated "pulse" by modulating threshold. |
| Gaussian / Directional / Radial blur | Separable passes, `MPSImageGaussianBlur` where available, custom kernels for radial and directional. |
| Color: Curves, Levels, Lumetri-style grade | 3D LUT (33³) generated from parameters; one trilinear lookup per pixel. |

### 6.4 Voronoi shatter

* Seed N points (blue-noise or radial-weighted around an impact point).
* **GPU:** Jump Flooding Algorithm produces a cell-ID texture in O(log₂ size) passes; each cell becomes a "shard" with a centroid and mask.
* **Animation:** each shard is a quad sampling the source through its cell mask, with rigid-body state (position, rotation, velocity, angular velocity) integrated in a compute kernel (gravity, impulse from the impact, damping). Extrusion/thickness for 3D shatter by instancing a prism mesh built from the cell polygon (CPU Fortune's algorithm for the polygon, run once per shatter setup).

### 6.5 Particles

* Structure-of-arrays in `MTLBuffer`s: `position`, `velocity`, `age/life`, `size`, `colour`, `seed`.
* **Update kernel** per frame: `v += (gravity + wind + turbulence(p, t)) · dt`, drag, collision planes, age; dead particles recycled via an atomic free-list.
* **Emission** counts come from the evaluator (animatable). **Draw** with indirect command buffers (instance count written by the GPU), additive or alpha-blended point sprites, optional soft-particle depth fade.
* Determinism: seeded by `(emitterID, particleIndex)`, so scrubbing backwards matches playing forwards. Cache a keyframe of particle state every 1–2 s and re-simulate forward from the nearest.

### 6.6 Puppet pin mesh warp (ARAP)

1. Triangulate the layer's alpha silhouette (constrained Delaunay, CPU, once per pin-set change) with ~1–3k vertices.
2. User pins are positional constraints on vertices.
3. **As-Rigid-As-Possible** local/global solve per frame:
   * *Local:* per-triangle best-fit rotation via 2×2 SVD (closed form).
   * *Global:* solve the sparse Laplacian system with the pinned vertices eliminated. The matrix depends only on topology and pin set, so **pre-factorise once** (sparse Cholesky, Accelerate `SparseFactor`) and only back-substitute per frame (2 solves × 3–5 iterations).
4. Render the deformed mesh with the original UVs; overlap ordering from pin "depth" values.
5. Preview at ≈30 fps for ≤ 2k vertices on A15 and newer is realistic; cache for export.

### 6.7 Expression engine

Use **JavaScriptCore** (system framework, JIT-free on iOS, no extra binary), not V8.

```swift
import JavaScriptCore

final class ExpressionEngine {                   // one instance per worker thread; JSContext is not thread-safe
    private let js = JSContext()!
    init() {
        js.exceptionHandler = { _, e in print("expression error:", e?.toString() ?? "?") }
        js.evaluateScript(Self.prelude)
    }
    func evaluate(_ source: String, time: Double, index: Int, value: Double, seed: Int) -> Double? {
        js.setObject(time,  forKeyedSubscript: "time"  as NSString)
        js.setObject(index, forKeyedSubscript: "index" as NSString)
        js.setObject(value, forKeyedSubscript: "value" as NSString)
        js.setObject(seed,  forKeyedSubscript: "__seed" as NSString)
        let r = js.evaluateScript(source)        // result is the last expression, as in AE
        return (r?.isNumber == true) ? r?.toDouble() : nil
    }
    static let prelude = """
    function __h(n){ n = Math.sin(n*127.1 + __seed*311.7)*43758.5453; return n - Math.floor(n); }
    function __noise(x){ var i=Math.floor(x), f=x-i; f=f*f*(3-2*f); return (__h(i)*(1-f)+__h(i+1)*f)*2-1; }
    function wiggle(freq, amp, oct, mult, t){
      oct = oct||1; mult = (mult===undefined)?0.5:mult; t = (t===undefined)?time:t;
      var s=0, a=amp, f=freq;
      for (var o=0; o<oct; o++){ s += a*__noise(t*f + o*17.3); f*=2; a*=mult; }
      return value + s;
    }
    function linear(t,a,b,c,d){ var u=Math.min(Math.max((t-a)/(b-a),0),1); return c+(d-c)*u; }
    """
}
```
* Property linking (`thisComp.layer("A").transform.position`) is a **proxy object graph** rebuilt per evaluation from the `RenderGraph`, read-only.
* Dependency graph: parse identifiers once, topologically sort property evaluation, detect cycles and show an error badge rather than hanging.
* Evaluations are cached per `(property, frame)` and invalidated by dependency edits. Time limit per evaluation (≈2 ms) enforced by running on a watchdog queue.

---

## 7. Neural module (ANE)

| Stage | Model class | Input → output | Placement |
|---|---|---|---|
| Face Restore | Small face-restoration network (GFPGAN-class) | Detected face crops 256² → restored crops | After the source layer, before effects. Vision `VNDetectFaceRectanglesRequest` finds crops; blend back with a feathered mask and `strength`. |
| Frame Synthesis | Flow-based interpolation (RIFE-class) | frames (t₀, t₁) + timestep τ → frame at τ | Time-remap layer. N× slow motion = N−1 evaluations per source pair; in preview, cache and run at proxy resolution. |
| Detail Upscale | Real-ESRGAN-class (compact variant) | tile 256² (+overlap 16 px) → 1024² | Final render only. Tiled with overlap-and-feather, run through the same zero-copy buffers. |

**Making models ANE-friendly (coremltools):**
* Convert to ML Program (`.mlpackage`), fp16 weights; palettise or int8-quantise weights (A16/M2+ gains most). Verify accuracy against fp32 on a held-out clip set.
* Fixed or *enumerated* input shapes (e.g. 256², 512²) so the ANE compiler can plan memory. Flexible shapes can fall back to the GPU.
* Avoid ops the ANE lacks (custom grid_sample for warping is the classic offender): do the flow warp in a **Metal kernel**, not in the model; keep the model to convolution/activation blocks.
* Inspect placement with Xcode's Core ML Performance Report and Instruments (Core ML template). Anything silently on the CPU is a regression.
* Simulator has no ANE; test on device.

**Scheduler rules**
1. Neural results are cached on disk by `(clipID, frame, modelVersion, params hash)`.
2. Scrubbing shows the last good result, never blocks.
3. When idle, a background task pre-computes frames around the playhead (low QoS).
4. Respect Low Power Mode and thermal state: disable neural preview, keep export available.

---

## 8. Export

`AVAssetWriter` (HEVC / H.264 hardware via VideoToolbox, ProRes on supported devices). Render loop: evaluate → composite → (neural + upscale) → `CVPixelBuffer` from a `CVPixelBufferPool` → `AVAssetWriterInputPixelBufferAdaptor`. Offline render is deterministic and not tied to the display link. Audio mixes through `AVAudioEngine` offline mode into a second writer input. Provide a progress/ETA UI and background-task assertion for short renders.

---

## 9. Test and quality gates

* Golden-image tests for each shader (fixed seed, tolerance on ΔE).
* Property-based tests for `CubicBezier.value(at:)` (monotonic x, endpoints, parity with CSS reference values).
* Frame-time budget: 16.6 ms at 1080p preview with ≤ 8 layers + 3 effects on A15; Metal System Trace in CI on a device farm.
* Memory: `MTLHeap` high-water mark logged per project; fail the build if a reference project exceeds its budget.

---

## 10. Suggested delivery order

1. **Now (this repo):** UI shell, timeline, easing editor, displace + CA, present/compare.
2. Real media: `AVPlayerItemVideoOutput` → `CVMetalTextureCache`, layer transforms, keyframes evaluated through the easing LUTs, 27 blend modes, mattes.
3. Text and shape layers, Glow/Kawase, blur family, Wave Warp, colour tools. Undo/redo, project save, export.
4. Expressions (JavaScriptCore), parenting, precomps.
5. 3D camera + lights, shadows, DoF.
6. Particles, Voronoi shatter, Puppet pins.
7. Neural: Face Restore → Detail Upscale (export) → Frame Synthesis.

---

## 11. Building the `.ipa`

Building requires a Mac with Xcode 15+; an `.ipa` cannot be produced on Windows or Linux.

1. Use `project.yml` with XcodeGen (`xcodegen generate`), or create an iOS App in Xcode (SwiftUI, iOS 17) and delete its generated `@main` file.
2. Add everything in `MotionForge/` to the target, including `Shaders.metal` and `Assets.xcassets`.
3. Add Core ML models as `.mlpackage` once you have them. Add `NSPhotoLibraryUsageDescription` / camera / microphone strings as you add import and capture.
4. Run on a physical device (Metal and the ANE behave differently from the Simulator).
5. Product → Archive → Distribute App:
   * *Development / Ad Hoc* → `.ipa` for registered devices.
   * *App Store Connect* → TestFlight and release.
   A paid Apple Developer Program membership is required for Ad Hoc, TestFlight and App Store distribution.
