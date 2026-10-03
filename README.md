# Screen Check

> **2.0 engine (September 2026):** fused Metal keyer, true 24/30/60 fps pacing, dynamic resolution, LiDAR screen-plane lock and screen extension, on-device AI object assist, Tracked 3D and 360° plates, high-resolution key stills. Requires iOS 26 and Xcode's Metal Toolchain. See [ScreenCheck-2.0-Engine.md](ScreenCheck-2.0-Engine.md); where it disagrees with the notes below about the live pipeline, the 2.0 note is current.

Screen Check is a native SwiftUI production-previsualization app for green-screen, blue-screen and screen-free shoots. It combines a live camera composite with the original setup planner, so a crew can see the intended background and identify problems before recording.

## Live Previs

- Explicit Green, Blue and Smart matte modes with reliable live switching.
- Real-time green- or blue-screen key rendered through Core Image on the GPU, with exposure-normalized color dominance for better consistency across hot and shadowed screen areas.
- Tap **Sample** and touch bare fabric to match the actual screen color, or use **Auto Key** to detect green/blue and choose a practical starting strength, edge and despill.
- Smart Background isolates the subject without a physical screen and fails safely to the original camera when no valid subject mask exists.
- Choose a built-in environment or any still image through Apple’s Photos picker.
- Edit background scale, position, rotation, blur, brightness and saturation.
- Use a fixed plate or bounded ARKit camera-tracked plate on any world-tracking iPhone or iPad.
- Adjustable key strength, edge softness and despill.
- Temporally stabilized person protection keeps performers and difficult edges from being removed with the screen.
- LiDAR depth confidence filtering and an optional nearby-props control reduce unwanted depth retention.
- LiDAR depth and confidence filtering run at the sensor's native resolution, then the finished mask is scaled once; enlarging low-resolution sensor buffers earlier would waste GPU work without creating detail.
- One-tap Screen Depth calibration samples a confidence-filtered LiDAR patch from the center of a real green or blue screen and sets the cutoff automatically.
- Device status now distinguishes LiDAR hardware availability, active depth assist, and a thermal pause; Green/Blue chroma remains responsible for removing the screen.
- Composite, Matte, Camera and Depth inspection views expose what the keyer is doing.
- Efficiency, Balanced and Quality modes bound the whole effects pipeline to 720p, 1280p or 1920p-class working dimensions and 20, 24 or 30 FPS.
- The live image stays on the GPU from Core Image through a Metal-backed preview surface, avoiding a full-resolution CPU image copy and SwiftUI view update for every frame.
- Camera and display backpressure retain only the newest useful frame, cap GPU work in flight, and drop stale frames instead of allowing latency or memory to accumulate. The Metal drawable is capped to the processed image resolution instead of unnecessarily rendering an upscaled copy at the full Retina panel resolution.
- Temporal person/depth stabilization retains one raw mask only. It never feeds a filtered lazy Core Image graph back into the next frame, so older camera, segmentation and LiDAR buffers cannot accumulate. Vision person segmentation runs independently from the live composite.
- Coverage sampling runs on a utility queue, while the displayed FPS value counts completed Metal frames rather than merely counting scheduled work.
- Stopping the camera or receiving an iOS memory warning releases transient frame/mask history and purges Core Image caches.
- Launch-time key-table generation, Photos decoding and frequent settings writes stay off the critical interaction path; duplicate camera starts are ignored.
- Thermal limiting automatically reduces optional processing while the device cools.
- Live screen-coverage and frame-rate indicators.
- Bare-screen measurement reports min/max code value, spread in code values and stops, plus a hotspot map; use it before the performer enters frame.
- Save the current composite directly to Photos and switch to Clean Feed for uncluttered wired or AirPlay display mirroring; the tab bar disappears and the exit control fades until the screen is tapped.
- Interactive Demo mode runs the real composite pipeline on a generated reference scene when camera/world tracking is unavailable.
- Matte, background and planner settings persist between launches.
- Apple-native materials, controls, SF Symbols, accessibility labels and responsive iPhone/iPad layouts.
- The device-processing information sheet links to a consistent About screen with bundle-derived version/build details, transparent calculation limits, CRIT Studio support and licensing contacts.

## Automatic device processing

1. **LiDAR Fusion** — the preferred Smart Background path on LiDAR-equipped iPhone Pro and iPad Pro hardware. ARKit smoothed scene depth keeps the performer and nearby props, while person segmentation refines the subject.
2. **Neural Matte** — the next-best path on supported non-LiDAR iPhones and iPads. ARKit isolates people; use Green or Blue mode when non-person props also need a precise key.
3. **Vision Assist** — the fallback on older compatible hardware. Vision periodically refreshes a person-only mask while Green and Blue modes remain available.

The app detects support at runtime; there is no device-name allowlist to become stale.
One capability-resolved frame plan gates the ARKit semantics, Vision fallback, depth filters and foreground blending together. A saved LiDAR preference therefore cannot start depth-related or substitute neural work on a non-LiDAR device.

## Optional high-quality key bridge

The wand button in Live Previs opens a three-step finishing workflow:

1. Capture the current original camera frame, Screen Check alpha hint and live composite.
2. Export a portable `.screencheckkey` package containing `Input` and `AlphaHint` folders, a preview, manifest and instructions.
3. Open the package in a compatible desktop keying or compositing workflow, export the finished composite as an RGBA PNG and import it into Screen Check for review.

No external keying application, model weights or desktop runtime are bundled or downloaded by Screen Check. The bridge is file-based, tool-neutral and optional, and all live processing remains native to iOS.

## Setup planner

The Plan tab estimates camera coverage, performer spill, clean-key stand-off and screen-lighting evenness before the set is built. It supports Full Frame, Super 35, Micro Four Thirds, Super 16, iPhone-equivalent and custom sensor gates; free focal lengths; metric or imperial units; direct camera distance; movement width; relative screen exposure; recommended light distance; named setup saves; and a shareable text report. Plan values feed Previs, while measured LiDAR distance, live coverage, sustained frame rate and screen evenness return to the Plan tab for comparison.

## Open and run

1. Open `ScreenCheck.xcodeproj` in Xcode 26 or later, with the Metal Toolchain component installed (Xcode › Settings › Components).
2. Select a physical iPhone or iPad for the live camera workflow, or a simulator to use the interactive demo and planner.
3. Run the `ScreenCheck` scheme.

The project targets iOS 26 and later and has no third-party dependencies. Live camera, semantic matte and LiDAR behavior require a physical device; the simulator uses an interactive reference frame through the same chroma/composite controls.

## Complete master documentation

[ScreenCheck-Complete-Project.md](ScreenCheck-Complete-Project.md) is the authoritative handoff document. It records the original product intention, intended users, requirements, workflows, LiDAR and non-LiDAR strategy, real-device testing guidance, UX system, processing architecture, planner mathematics, privacy, build instructions, validation, limitations, roadmap, release checklist, and project artifact map.

## Validation

`ScreenCheckTests` contains 35 tests covering the analytic point-to-polygon form factor, tangent-plane clipping, spill falloff, lighting evenness, normalized and sampled chroma, aspect-fill tap mapping, Auto Key, bare-screen measurement, sensor geometry, units, exposure-aware spill, green/blue switching, Smart Background fallback order/readiness, LiDAR calibration math, bounded plate tracking, persistence, responsive layout, performance profiles, latest-frame preview retention, bounded mask history, drawable-resolution limits, LiDAR/non-LiDAR processing routes and high-quality package structure. The production app and XCTest bundle build successfully for generic arm64 iOS hardware. Live camera behavior should still be profiled on a physical device because ARKit camera and LiDAR input are unavailable in the simulator.
