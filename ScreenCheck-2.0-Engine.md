# Screen Check 2.0: realtime engine rework

**Version:** 2.0 (build 2) · **Minimum OS:** iOS / iPadOS 26 (A13 Bionic or newer) · **Last verified:** 25 September 2026

This note records what changed between the 1.0 prototype and 2.0, why, and how to test it on a device. The 1.0 master document (`ScreenCheck-Complete-Project.md`) still describes the product, planner and data model; where the two disagree on the live pipeline, this file wins.

## 1. Why the preview was slow in 1.0

A line-by-line review of the 1.0 engine found these causes, in rough order of impact:

| Cause | Effect in 1.0 | Fix in 2.0 |
|---|---|---|
| Frame gate `timestamp - last >= 1/fps` on a 60 Hz camera | "24 fps" actually ran at 20 fps (41.7 ms always rounded up to 50 ms); 30 fps jittered between 30 and 20 | `FramePacer` accumulates deadlines, so 24 / 25 / 30 / 60 average exactly (unit-tested) |
| Background colour controls and blur recomputed every frame | A radius-20 blur on a 2048 px plate on every camera frame | `BackgroundPlateCache` renders the graded plate once into an owned buffer; per frame only an affine or perspective sample |
| Core Image render encoded on the main thread inside `MTKView.draw()` | Kernel compile and encode competed with SwiftUI | `LivePreviewSurface` renders on its own queue straight into a `CAMetalLayer` drawable, two frames in flight |
| Masks upscaled to full resolution before morphology, blur and dissolve | 4 to 5 full-resolution passes on 256 px sensor data | Masks are built, stabilised and stored at sensor resolution (~256 px), then upscaled once |
| `latestSourceForAnalysis`, mask history and lazy graphs held ARKit buffers | ARKit's small buffer pool could starve and drop frames | Analysis requests are queued and run on the next frame; masks are rendered into app-owned pixel buffers |
| 3D colour-cube rebuild (35 ms debounce) on every slider change | Laggy sliders; brief unkeyed frames on Green/Blue switch | Chroma maths runs directly in a Metal kernel: instant slider response, no LUT |
| ~12 separate Core Image filters per frame | Many intermediate textures | One fused `scComposite` kernel: key, despill, mask routing, light wrap, colour match and composite in a single pass |
| A main-thread hop on every frame to re-publish "running" | Needless SwiftUI invalidation | Published once per session start |

## 2. New pipeline

```
ARFrame (1920x1440 @ 60 Hz, format chosen for high-res still capture)
  ├─ FramePacer (24 / 30 / 60 target, thermal aware)
  ├─ camera -> working image (960 / 1280 / 1440 px x adaptive scale)
  ├─ scMaskBuild @ ~256 px  <- person seg · LiDAR depth + confidence · AI object mask
  │     r = person/object, g = LiDAR near, b = LiDAR far (behind screen plane)
  ├─ scTemporal  <- previous owned mask (motion-adaptive smoothing)
  ├─ render into owned BGRA buffer (breaks ARKit + graph retention)
  ├─ scGuidedUpsample (joint bilateral, guided by camera luma) -> working res
  ├─ background: cached plate -> Fixed | Tracked 3D (perspective) | 360 (scPanorama)
  └─ scComposite (one pass) -> CAMetalLayer on the render queue
```

Kernels live in `ScreenCheckKernels.metal` and are compiled as Core Image kernels (`MTL_COMPILER_FLAGS = -fcikernel`, `MTLLINKER_FLAGS = -cikernel`). The Swift reference `ChromaKeyMath` mirrors the kernel's key and despill, and `testCompositeKernelMatchesTheCPUReference` checks the GPU result against it.

## 3. New capabilities

### Performance
- **60 fps monitoring** in Quality mode on A15 or newer (or M-series / 6 GB A14). Balanced is now a true 30 fps and Efficiency a true 24 fps.
- **Dynamic resolution:** if displayed fps falls below 88% of target, the working size drops in 10% steps down to 60%; it recovers in 5% steps after three healthy seconds. The header shows "FPS · 80%" while scaled.
- **Device tier** detection from Metal GPU families, never a model list.

### LiDAR (Pro devices)
- **Screen plane lock.** Calibrate now fits the physical screen as a plane with RANSAC over the LiDAR point cloud, rejecting floors, ceilings and the performer, and locks it in world space. The depth cutoff follows the real screen as the camera moves and stays correct across a screen that is angled to the lens. The tilt is shown and written into the key-package manifest.
- **Extend screen.** An optional garbage matte removes everything at or behind the screen plane, so light stands, walls and ceiling beyond a small indie screen disappear without extra fabric. People are never removed by it.
- **Smarter protection.** LiDAR "near" now only rescues pixels the chroma key is unsure about. A green floor in front of the screen is still keyed, while spill-heavy edges and props survive.

### On-device AI (Apple's built-in models, no bundled weights)
- **AI object assist** in Smart mode on non-LiDAR devices uses Vision's foreground-instance model (the "lift subject" model in Photos) a few times per second on the Neural Engine, so props, pets and objects stay with the performer. It pauses automatically when LiDAR is already keeping props.
- **Edge-aware mask upsampling** snaps low-resolution person and depth masks to real image edges.
- Legacy Vision person fallback now uses `.balanced` quality and runs only where ARKit segmentation is missing.

### Compositing
- **Despill on every pixel** with partial luminance restore (1.0 only despilled semi-transparent pixels, so opaque spill on skin stayed green).
- **Light wrap** and **Colour match** sliders in a new Blend section.

### Background plates
- **Tracked 3D:** the plate is anchored in the room at a chosen distance (2 to 60 m) and re-projected every frame with ARKit's camera, so pans and tilts behave like a real backdrop and walking gives correct parallax. Falls back to the bounded 2D motion if projection fails.
- **360°:** choose any 2:1 equirectangular photo (auto-detected, decoded up to 4096 px). The background follows the camera in every direction. Horizontal / Vertical re-aim, Scale zooms, "Re-centre on current view" resets.

### High Quality Key
- Uses ARKit's **high-resolution still capture** for the package's `Input` frame where the device supports it (manifest `inputSource: "high-resolution-still"`), with the live alpha hint scaled to match. Manifest is now `formatVersion: 3`.
- Fixed the README preview-path wording flagged in section 15 of the master document.

## 4. Compatibility

- Deployment target raised from iOS 17 to **iOS 26**. iOS 26 and 27 both require A13 or newer, so this drops only iPhone XS / XR-era hardware and guarantees a 3rd-generation Neural Engine and a GPU that holds the new pipeline.
- 1.x preferences decode with defaults for the new fields (unit-tested); nothing is lost on upgrade.
- Building requires Xcode's **Metal Toolchain** component (Xcode › Settings › Components, or `xcodebuild -downloadComponent MetalToolchain`).

## 5. Verification

- App, Metal Core Image kernels and tests compile for generic arm64 iOS with Xcode 27.0 (27A266a) and Metal Toolchain 32023.921 on the creator's Mac.
- The Interactive Demo renders through the full GPU pipeline in the simulator (`screencheck-2.0-demo-green.png`).
- 48 unit tests (all passing on the iPhone 17 Pro simulator, Xcode 27.0), including frame pacing, dynamic resolution, camera-ray maths, plane fitting on a synthetic angled screen with floor and performer outliers, world/camera plane round trips, world-anchored plate projection, legacy preference decoding, routing, despill and GPU-vs-CPU keyer parity.
- Not yet verified: live camera, LiDAR, thermals and real frame rates. These need the device pass below.

## 6. Device test checklist

Run a **Release** build, unplugged from Xcode, for each item.

1. **Frame rate.** Balanced should read 30 FPS steady; Quality on an A15+ device should read close to 60. Note if the "· %" scale indicator appears and at what setting.
2. **Green / Blue switch** is instant with no unkeyed flash; slider drags respond without lag.
3. **Despill:** a performer close to the screen should lose green on skin and hair without going grey.
4. **LiDAR plane lock** (Pro): angle the phone about 20° to the screen, tap Calibrate, confirm the tilt readout, then walk sideways. The cutoff should stay on the fabric.
5. **Extend screen** (Pro): frame wider than the screen so a stand or wall is visible beside it. With Extend on, those areas should be replaced. The performer must never be cut.
6. **AI object assist** (non-Pro, Smart mode): hold a prop; it should appear within a fraction of a second and stay.
7. **Tracked 3D:** pan and walk; near plate distances should show more parallax than far ones.
8. **360°:** pick a 2:1 panorama; look all around, including up and down.
9. **High Quality Key:** export a package and check `manifest.json` for `inputSource` and the Input PNG's resolution.
10. **Thermals:** 15 minutes in Quality; the thermometer should appear rather than frame drops or a crash.
