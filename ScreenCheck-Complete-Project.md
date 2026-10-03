# Screen Check — Complete Project Documentation

**Product:** Screen Check  
**Platform:** iPhone and iPad  
**App version:** 1.0 (build 1)  
**Minimum OS:** iOS 17.0  
**UI framework:** SwiftUI  
**Language mode:** Swift 5  
**Bundle identifier:** `com.kamal.screencheck`  
**External dependencies:** None  
**Last verified:** 1 September 2026  
**Document role:** Master product, design, engineering, testing, and handoff reference  

---

## 0. Product intention and origin

### The original intention

Screen Check was created to answer a practical production question before a crew commits time, equipment, performers, and money to a chroma-screen setup:

> **What will this shot look like after the screen is replaced, and is the physical setup good enough to produce a usable result?**

Traditional on-set camera previews show the green or blue background but do not show the intended environment. This makes it harder to judge framing, performer placement, screen coverage, spill, lighting evenness, edge quality, and whether important props will survive the key. Professional virtual-production and compositing systems can solve parts of this problem, but they are often expensive, fixed to a workstation, or too complex for a quick scout, rehearsal, student production, independent shoot, or small crew.

The intention of Screen Check is to turn the iPhone or iPad already carried by the crew into a fast, understandable previsualization and planning instrument. It is not intended to replace a final compositor. It is intended to reveal likely problems early, communicate the visual intention clearly, and help the crew make better decisions while changes are still inexpensive.

### The product promise

Screen Check combines three jobs in one Apple-native application:

1. **See the intended shot:** remove a real green or blue screen—or isolate a real person without one—and preview a replacement background live.
2. **Use the best device capability automatically:** fuse LiDAR depth and person segmentation on supported Pro hardware, then fall back gracefully to semantic or Vision person mattes on other compatible devices.
3. **Plan before building:** estimate camera coverage, performer separation, chroma spill, and screen-lighting evenness before the set is finalized.

### Why iPhone and iPad

- They are portable, battery-powered, and already familiar to most crews.
- Their cameras, GPU, Neural Engine, motion tracking, and—on supported Pro models—LiDAR can cooperate in one low-latency pipeline.
- SwiftUI, ARKit, Vision, Core Image, PhotosUI, and system accessibility provide a private, dependency-free, on-device implementation.
- iPad offers a larger monitoring surface, while iPhone provides the most portable scouting and rehearsal experience.
- Runtime capability detection lets the same project offer the best available workflow without maintaining a fragile list of device names.

### Intended users

- Directors, cinematographers, VFX supervisors, and virtual-production teams.
- Independent filmmakers, small crews, students, educators, and content creators.
- Producers and clients who need to understand the planned composite before post-production.
- Location scouts and technicians checking whether a room can support the proposed screen, lens, distance, and lighting arrangement.

### Core product principles

- **Previsualization first:** prioritize immediate understanding over final-pixel finishing.
- **Truthful capability labels:** distinguish hardware availability from a feature that is actually enabled.
- **Progressive enhancement:** LiDAR improves the experience but never becomes a requirement for basic Green, Blue, or Plan workflows.
- **Safe failure:** if a subject mask is unavailable, show the camera rather than an unrelated or destructive matte.
- **Real-set semantics:** clearly distinguish physical depth from virtual imagery displayed on a television or monitor.
- **Apple-native interaction:** use familiar controls, materials, SF Symbols, haptics, accessibility, safe areas, and responsive layouts.
- **Bounded realtime work:** drop stale frames, adapt quality, and protect thermals instead of allowing latency or memory to grow without limit.
- **On-device privacy:** keep camera processing and settings local, with explicit user-driven import and export.

### Definition of success

The app succeeds when a user can open it on a supported iPhone or iPad, understand which processing is available and active, choose the correct matte workflow, preview the intended background, recognize setup problems, and communicate the planned shot without needing a workstation. A successful preview does not guarantee a final-quality key; it gives the crew evidence for what to fix or preserve before recording.

### Reading guide

- Sections 1–2 define the product and its requirements.
- Sections 3–5 explain workflows, interface, and device behavior.
- Sections 6–8 document realtime processing, finishing handoff, and planner mathematics.
- Sections 9–12 cover design, architecture, state, privacy, and permissions.
- Sections 13–14 contain build and verification instructions.
- Sections 15–21 record limitations, roadmap, release readiness, glossary, artifacts, and final handoff status.

---

## 1. Project summary

Screen Check is a native iPhone and iPad production-previsualization tool for green-screen, blue-screen, and screen-free shoots. It gives a crew two connected workspaces:

1. **Previs** — a real-time camera composite showing the intended replacement background.
2. **Plan** — a setup calculator for screen coverage, camera placement, performer separation, chroma spill, and screen-lighting evenness.

The live system automatically selects the best processing available on the current Apple device:

- LiDAR depth plus person segmentation on supported Pro hardware.
- ARKit person segmentation on supported non-LiDAR hardware.
- Vision person segmentation as the legacy world-tracking fallback.
- GPU chroma keying remains available for green and blue screens.

The app is designed to stay useful across the device range. LiDAR improves subject and nearby-prop isolation, but it is not required for the main green/blue workflow or the planning tools.

---

## 2. Product goals

### Primary goals

- Show a live approximation of the final composite before shooting.
- Switch reliably between green, blue, and screen-free Smart mattes.
- Let users select a built-in environment or a custom image from Photos.
- Use LiDAR when available without making it mandatory.
- Give non-Pro iPhones and iPads the best practical person-matte fallback.
- Identify screen coverage, spill, and lighting problems before the set is committed.
- Preserve an Apple-native visual language and interaction model.
- Keep processing on the device and avoid third-party runtime dependencies.

### Current non-goals

- The app is not a full video recorder or editor.
- Smart Background on non-LiDAR devices is person-only; it is not a universal object segmentation system.
- The live key is intended for previs, not final cinema-quality finishing.
- No external desktop keying engine or model weights are embedded in the iOS app.
- The planner is a production estimate, not a physically complete ray tracer.

### Requirement-to-implementation summary

| Product requirement | Implemented response |
|---|---|
| Apple-designed appearance | Native SwiftUI, semantic materials, SF Symbols, system controls, haptics, an adaptive light/dark planner, a dark production monitor, and accessibility labels |
| Reliable Green and Blue switching | Always-visible segmented mode control, explicit matte state, revision-protected asynchronous color-cube rebuilding, and safe unkeyed transition frames |
| Fast real-set key setup | Tap-to-sample eyedropper, Auto Key, exposure-normalized dominance, live Matte view while adjusting sliders, and measured bare-screen evenness |
| Background preview | Built-in environments, Photos import, fixed or ARKit-tracked plate motion, transform, blur, brightness, and saturation controls |
| Best Pro-device experience | LiDAR scene-depth fusion, confidence rejection, nearby-prop retention, physical-screen depth calibration, and thermal status |
| Best non-Pro alternative | ARKit person segmentation, with Vision person segmentation as the older world-tracking fallback |
| Screen-free option | Smart Background person isolation with optional LiDAR depth fusion |
| Setup planning | Real sensor formats or a custom gate, free focal length, metric/imperial units, direct camera distance, movement envelope, screen exposure, coverage, spill, clean-key stand-off, and lighting-evenness analysis |
| Plan-to-set validation | Shared Plan/Previs state, expected-versus-measured distance, coverage/FPS/evenness feedback, and one-tap use of measured LiDAR distance |
| Reusable handoff | Named setups, shareable text setup reports, saved composite stills, and a clean mirrored output mode |
| No-device evaluation | Interactive generated demo frame processed through the real key and composite pipeline |
| Small and large Apple displays | Safe-area-aware responsive overlays, bounded iPhone controls, internal scrolling, and an iPad width cap |
| App opening and usage performance | Lazy heavy resources, quality-bounded processing, frame backpressure, independent Vision work, debounced persistence, and thermal reduction |
| Reference and support details | Device Processing links to a standard About screen with bundle-derived build details, method limits, and CRIT Studio contacts |
| Higher-quality finishing option | Optional `.screencheckkey` still package for compatible desktop keying or compositing tools |
| Complete handoff | Buildable Xcode project, source, tests, screenshots, README, this master document, and packaged ZIP |

---

## 3. Main user flows

### 3.1 Live green- or blue-screen previs

1. Open the **Previs** tab.
2. Grant camera permission.
3. Choose **Green** or **Blue** in the always-visible segmented control.
4. Open **Matte Controls** and use **Sample** on uncovered fabric, or run **Auto Key**.
5. Adjust Strength, Edge, and Despill; the preview temporarily switches to Matte while a slider is being dragged.
6. Before the performer enters, fill the frame with bare screen and run **Measure Screen Evenness**.
7. Optionally enable Protect people and LiDAR depth assist.
8. Select Soundstage, Dusk, Night, or a background from Photos.
9. Review the truthful session status, composite, frame rate, screen coverage and evenness result.

### 3.2 Smart Background without a physical screen

1. Open Matte Controls.
2. Select **Smart**.
3. Choose a built-in or Photos background.
4. On LiDAR hardware, enable depth assist and set the Foreground range.
5. Frame the subject and review the live isolation.

Smart behavior depends on the device:

- **LiDAR device:** person matte and scene depth are merged. Nearby props can be retained when they fall inside the chosen foreground range.
- **Non-LiDAR device with ARKit person segmentation:** the performer is isolated, but non-person props are not guaranteed.
- **Older world-tracking device:** Vision periodically updates a person-only matte.

For reliable non-person props on non-LiDAR devices, use a physical green or blue screen.

### 3.3 Replace the background

- Tap **Soundstage** to select Soundstage, Dusk, or Night.
- Tap **Photo** to choose any still image through Apple’s Photos picker.
- Open **Background & Preview** from the background menu to adjust scale, horizontal/vertical framing, rotation, blur, brightness, and saturation.
- Select **Fixed** to lock the plate to the display or **Tracked** to add restrained ARKit camera-motion parallax.
- Tracked plates work on every ARKit world-tracking device; LiDAR improves foreground occlusion but is not required for plate tracking.
- Custom images are decoded away from the interface thread and downsampled to a safe 2048-pixel maximum before Core Image processing, then use aspect-fill scaling.
- Selecting a built-in preset clears the custom image.

### 3.4 Plan a shoot

1. Open the **Plan** tab.
2. Choose units, sensor format, focal length and framing; optionally enter camera distance directly.
3. Enter screen size and material.
4. Enter performer height, distance from screen and movement width.
5. Set the number and distance of screen-lighting units, plus screen exposure relative to the subject key.
6. Review the verdict, spill map, through-lens view, evenness map, recommended light distance and numerical readouts.
7. Open Previs; expected screen distance is shared automatically.
8. Return to Plan to compare coverage, LiDAR distance, sustained frame rate and measured screen evenness. A measured distance can be applied back to the plan.
9. Save a named setup or share the text setup report.

### 3.5 High-quality still handoff

1. Tap the wand button in the Previs header.
2. Capture the current frame.
3. Export a `.screencheckkey` package.
4. Process it with a compatible desktop keying or compositing tool.
5. Export an RGBA PNG from that tool.
6. Import the finished PNG back into Screen Check for review.

### 3.6 Quick still and clean monitor output

- **Save Still** writes the current composite to Photos after requesting add-only permission.
- **Clean Feed** hides operational controls and the tab bar so an iPhone or iPad display can be mirrored over a cable or AirPlay as a simple client monitor. The exit control fades after two seconds and one tap reveals it.
- Clean Feed is display mirroring, not a custom low-latency network video protocol.

### 3.7 Interactive demo without camera hardware

When world tracking or camera input is unavailable, Previs creates a reference subject and screen, then runs it through the same chroma cube, background and inspection controls as live footage. The status is explicitly **DEMO FRAME** so generated input is never presented as a live camera measurement.

### 3.8 Real-device testing and correct LiDAR use

The app must be evaluated differently depending on whether the camera sees a real set or footage playing on another display.

#### Physical green or blue screen

1. Light the screen as evenly as possible and avoid deep shadows or reflected highlights.
2. Place the real performer or props in front of the physical screen with meaningful depth separation.
3. Select Green or Blue to match the fabric.
4. Enable Protect people.
5. On LiDAR hardware, enable LiDAR depth assist and Include nearby props only when real foreground depth should be preserved.
6. Aim the center of the camera at an uncovered portion of the physical fabric and tap **Calibrate Screen Depth**.
7. Return to the intended framing and refine Strength, Edge, and Despill.

Green or Blue chroma removes the screen. LiDAR does not remove a color; it prevents real foreground surfaces nearer than the calibrated screen plane from being lost.

#### Footage playing on a television or monitor

- Use Green or Blue mode and leave LiDAR depth assist off.
- LiDAR sees the television glass as one physical plane. It cannot know that the displayed actor is intended to be nearer than the displayed background.
- A filmed display can introduce moiré, pixel structure, refresh artifacts, glare, crushed shadows, a cyan/teal shift, and uneven brightness. These make the recaptured green less consistent than the source footage.
- A dark room mainly harms the chroma image through exposure, noise, reflection, and color inconsistency. LiDAR uses active sensing, but it still cannot create virtual depth inside the display.
- Screen coverage is the estimated portion removed from the camera frame, not a universal key-quality score.

#### Performance interpretation

- Efficiency, Balanced, and Quality target approximately 20, 24, and 30 displayed FPS. The header now measures Metal frames that completed, so the reading reflects visible output rather than work merely scheduled.
- A temporary low reading during camera startup, a sheet transition, or thermal change can recover after a few seconds.
- Persistently single-digit FPS on recent Pro hardware should be reproduced in a Release build disconnected from Xcode, using the same quality mode and controls so the comparison is meaningful.
- Live camera, person matte, thermal behavior, and LiDAR calibration require a physical device; simulator validation covers the interactive demo, interface and deterministic model logic.

---

## 4. App navigation and interface

The root interface is a two-tab SwiftUI `TabView`:

| Tab | Purpose | Main icon |
|---|---|---|
| Previs | Live camera, matte controls, and background preview | `camera.viewfinder` |
| Plan | Screen geometry, spill, coverage, and lighting calculator | `ruler` |

Previs stays dark for a consistent production-monitor aesthetic. Plan uses adaptive semantic colors so it remains legible in both light and dark system appearances.

### 4.1 Previs header

The floating header shows:

- App name: **SCREEN CHECK**.
- Best processing mode detected for the device.
- Measured preview frame rate when available.
- High Quality Key workflow button.
- Clean Feed button for uncluttered display mirroring.
- Device-processing information button.
- The Device Processing sheet retains live capability guidance and links to **About Screen Check** for version/build, methodology, support, licensing, website and creator credit.

### 4.2 Live status badge

Green/Blue modes display:

- `FINDING SCREEN`
- `SCREEN LOCKED`

Smart mode displays:

- `FINDING SUBJECT`
- `SUBJECT ISOLATED`

The badge also displays measured matte coverage when available. It reports camera permission, startup, interruption, failure and **DEMO FRAME** states directly; it does not claim to be finding a screen unless the live camera session is actually running.

### 4.3 Matte Controls

The expanded control panel begins with an explicit segmented selector:

| Mode | Purpose |
|---|---|
| Green | Removes a dominant green screen |
| Blue | Removes a dominant blue screen |
| Smart | Isolates the subject using person and optional depth masks |

Green and Blue modes expose:

- **Sample:** tap a bare screen area to key the actual measured chromaticity.
- **Auto Key:** identifies the dominant green or blue screen family and suggests initial Strength, Edge and Despill values.
- **Strength:** `0.25...1.00`, default `0.76`.
- **Edge:** `0.025...0.30`, default `0.12`.
- **Despill:** `0.00...1.00`, default `0.82`.
- **Protect people:** enabled by default.
- **Measure Screen Evenness:** analyzes bare screen, returns minimum/maximum code values, code-value spread, stop variation and hotspot position, and visualizes the sampled grid. The practical target shown in the app is a spread of 32 code values or less.

Dragging any key slider temporarily presents the Matte inspection view, then restores the previous inspection mode when the drag ends.

LiDAR devices additionally expose:

- **LiDAR depth assist:** enabled by default.
- **Include nearby props:** enabled by default and confidence-filtered.
- **Screen distance** in Green/Blue modes.
- **Foreground range** in Smart mode.
- **Calibrate Screen Depth** in Green/Blue modes. Aim the frame center at uncovered physical screen fabric and tap to measure it with LiDAR.
- Range: `0.5...8 m`, default `4 m`.

The live header and Device Processing sheet distinguish **LiDAR Available**, **LiDAR Assist On**, and a temporary thermal pause. Hardware availability no longer implies that depth assist is enabled. Green and Blue chroma perform screen removal; LiDAR protects foreground depth.

Smart mode replaces chroma sliders with a device-specific explanation so the interface only shows controls relevant to the selected matte. Switching matte mode also produces a light native selection haptic.

### 4.4 Bottom control dock

| Control | Action |
|---|---|
| Soundstage | Opens the built-in background menu |
| Photo | Opens Apple’s Photos picker |
| Save Still | Writes the current composite to Photos |
| Sliders | Expands or collapses Matte Controls |
| Green / Blue / Smart | Always-visible segmented matte-mode selector |

The background menu also opens **Background & Preview**, which contains plate movement, transform, look, preview-quality, and matte/depth inspection controls.

### 4.5 Responsive layout

- Horizontal overlay margin: `14 pt` per side.
- Maximum overlay width on iPad: `732 pt`.
- Effective overlay width: `min(viewport width - 28, 732)`.
- The camera and overlays are explicitly constrained and clipped to the viewport.
- Slider labels and values are placed above full-width sliders to prevent wrapping on iPhone.
- Expanded matte controls scroll inside a viewport-relative height, preventing the dock from leaving the screen in landscape or larger text configurations.
- The layout was visually verified at iPhone 15 Plus dimensions.

---

## 5. Device capability ladder

Capabilities are detected at runtime through `ARWorldTrackingConfiguration`. There is no hard-coded device-name list.

| Priority | Runtime capability | Display name | What it provides |
|---:|---|---|---|
| 1 | Smoothed scene depth | LiDAR Fusion | Depth protection, nearby props, and person matte |
| 2 | Person segmentation | Neural Matte | ARKit person protection or isolation |
| 3 | World tracking | Vision Assist | Periodic Vision person mask plus continuous chroma |
| 4 | Unsupported/simulator | Interactive Demo | Generated reference source through the real key/composite controls; no live measurements |

### Recommended use by hardware

#### LiDAR-equipped iPhone Pro and iPad Pro

- Use Smart mode for screen-free person and nearby-prop previews.
- Adjust Foreground range to avoid retaining distant walls or scenery.
- Use Green/Blue mode when the chroma screen should remain the primary matte.
- Depth and person segmentation can protect difficult foreground edges from the chroma key.

#### Modern non-Pro iPhone and iPad

- Use Smart mode for people-only background replacement.
- Use Green/Blue mode when props, furniture, or non-person objects must remain.
- ARKit person segmentation protects performers, hair, and clothing from false chroma removal.

#### Older compatible devices

- Vision generates a person mask at a reduced update frequency.
- Green/Blue remains the most dependable live mode.
- Slower subject movement improves the apparent Smart matte.

---

## 6. Live processing architecture

### 6.1 Frameworks

- **ARKit:** camera frames, world tracking, person segmentation, and scene depth.
- **AVFoundation:** camera authorization state.
- **Core Image:** color-cube chroma key, masks, compositing, gradients, and direct Metal-texture rendering.
- **MetalKit:** a GPU-backed live preview surface that avoids per-frame CPU image readback and SwiftUI image publication.
- **Vision:** legacy person-segmentation fallback.
- **PhotosUI:** custom background selection.
- **SwiftUI:** interface, state, sheets, menus, picker, and file workflows.
- **Uniform Type Identifiers:** `.screencheckkey` package registration.

### 6.2 AR session configuration

The app creates an `ARWorldTrackingConfiguration` with:

- Gravity world alignment.
- Environment texturing disabled.
- `.smoothedSceneDepth` only when LiDAR assist and nearby-prop retention currently require it.
- `.personSegmentationWithDepth` when the active controls require people and depth and the device supports it.
- Otherwise `.personSegmentation` when active people protection or Smart mode requires it.

A single per-frame processing plan resolves those choices from the runtime capabilities and active controls. On a non-LiDAR device, a persisted Depth Assist value cannot request scene depth, trigger a substitute Vision pass, or alter foreground blending. On LiDAR hardware, depth activates only when both Depth Assist and nearby-prop retention are enabled and the device is not thermally constrained.

The session resets tracking and removes existing anchors when first started or recovering from an interruption. Repeated SwiftUI lifecycle start requests are coalesced, and control changes update frame semantics without unnecessarily resetting world tracking.

### 6.3 Per-frame pipeline

The live pipeline targets 20, 24, or 30 frames per second depending on the selected quality, and automatically reduces work when iOS reports serious thermal pressure:

1. Receive an `ARFrame`.
2. Accept it only when the previous frame has completed; stale camera frames are dropped instead of queued.
3. Orient its captured pixel buffer from the active `UIWindowScene` interface orientation.
4. Preserve the original camera image for a requested high-quality still, then create a quality-bounded working image before expensive effects.
5. Apply the active 24³, 32³, or 48³ chroma color cube for Efficiency, Balanced, or Quality.
6. Acquire an ARKit person mask when present.
7. If necessary, asynchronously update and reuse a quality-dependent Vision person mask without blocking the render queue.
8. On LiDAR hardware, create a feathered foreground-depth mask and reject low-confidence samples at the depth sensor's native resolution, then scale the finished mask once to the preview extent.
9. Merge enabled person and nearby-prop depth masks with maximum compositing.
10. Stabilize the combined mask against one previous raw mask, expand it slightly, and soften its edge. The filtered result is never retained as history, preventing a recursively growing lazy Core Image graph.
11. Select the foreground strategy:
   - Green/Blue: chroma foreground, optionally blended with the protected original subject.
   - Smart: original camera foreground isolated by the combined subject mask.
12. Create, grade, transform, and optionally camera-track the selected background plate.
13. Composite foreground over background with source-over compositing.
14. Select Composite, Matte, Camera, or Depth inspection output.
15. Hand the newest lazy Core Image graph to a Metal-backed preview surface. Core Image renders directly into the current drawable texture in sRGB, with at most two GPU frames in flight; stale display work is skipped rather than queued.
16. Count completed Metal command buffers for the displayed FPS value. Measure coverage with a separate 1×1 area-average render on a utility queue, then publish only the small status values on the main thread. High-quality still handoff remains an intentional on-demand readback, retaining the full source image dimensions and upscaling its alpha hint from the realtime result.

### 6.4 Smart Background behavior

Smart mode always requests a subject mask.

- The person mask preserves the performer.
- The depth mask preserves pixels nearer than the configured foreground cutoff.
- The two masks are combined so either can retain a pixel.
- The combined mask is temporally stabilized to reduce edge flicker.
- The original camera image is blended over transparency using the combined mask.
- A valid subject must occupy a plausible portion of the frame before the UI reports `SUBJECT ISOLATED`.
- If a subject mask is unavailable, Smart mode fails safely to the unkeyed camera image. It never applies an unrelated green key.

The LiDAR cutoff uses a soft transition around the configured range to avoid an unnaturally hard depth edge.

#### Physical-screen depth calibration

- Calibration enables the required depth settings and waits for ARKit smoothed scene depth.
- It samples a centered patch rather than trusting one noisy pixel.
- Low-confidence samples, non-finite depths, and values outside `0.5...8 m` are rejected.
- The median of at least nine reliable samples becomes Screen distance, rounded to `0.1 m`.
- If no reliable plane appears within three seconds, the UI explains how to aim at bare physical screen fabric and retry.
- Calibration is intentionally unavailable in Smart mode, where the control is a foreground range rather than a screen plane.
- A TV or monitor is one physical plane: LiDAR cannot interpret the actor and background displayed inside it as different depths. The app explicitly recommends leaving LiDAR assist off for that test and using Green/Blue chroma instead.

### 6.5 Vision fallback

- Uses `VNGeneratePersonSegmentationRequest`.
- Quality is set to `.fast` for live responsiveness.
- Output is a one-component 8-bit mask.
- The request refreshes every third, fifth, or eighth processed frame for Quality, Balanced, or Efficiency mode; thermal limiting doubles that interval.
- At most one Vision request can be in flight, and the request runs on its own user-initiated queue so it cannot stall Core Image rendering.
- The latest successful Vision mask is reused between refreshes.

### 6.6 Green/blue chroma algorithm

For each color-cube sample:

1. Select green or blue as the key channel.
2. Find the maximum competing channel.
3. Divide channel dominance by peak RGB exposure: `(key - competing) / max(R, G, B)`. This keeps the same fabric hue much more consistent between bright and shadowed areas.
4. Compute threshold: `0.24 - strength × 0.20`.
5. Use smoothstep between threshold and threshold + softness.
6. Set alpha to `1 - keyed amount`.
7. Limit the key channel relative to the competing channels for despill.
8. Store premultiplied RGBA in the color cube.

This makes pure green transparent only in Green mode and pure blue transparent only in Blue mode. Automated tests protect this behavior.

When the user samples the screen, the cube instead compares normalized RGB chromaticity against the sampled color. Auto Key analyzes the current source frame, scores green and blue dominance, averages candidate screen pixels, and returns a sampled color plus conservative slider starting values. Both paths run on demand, away from the per-frame render loop.

### 6.7 Reliable switching and performance

- Matte mode is explicit state, not an ambiguous color toggle.
- When Green and Blue are switched, the old color cube is never used for the new mode; the app briefly shows the unkeyed source until the requested cube is ready.
- The engine is lightweight at creation: the Core Image context, Vision request and first color cube are initialized lazily or asynchronously after launch.
- Color-cube generation runs on a separate user-initiated queue and uses a quality-appropriate 24³, 32³, or 48³ table.
- Rebuild requests are debounced by 35 milliseconds.
- A revision number prevents an older calculation from replacing newer settings.
- Non-chroma changes do not rebuild the color cube once its configuration already matches.
- ARKit delivery and rendering use separate queues. A one-frame gate drops stale input whenever rendering falls behind, preventing latency and memory from growing without bound.
- The realtime preview is rendered directly from Core Image into an `MTKView` drawable. It does not create a full-resolution `CGImage` or publish a new SwiftUI `Image` for every camera frame.
- The preview surface keeps only the latest lazy image graph, coalesces main-thread draw requests, and allows at most two Metal command buffers in flight so display work cannot build an old-frame backlog.
- The Metal drawable's longest dimension never exceeds the processed source image. Balanced 1280-pixel processing is therefore not wastefully upscaled into a full native Retina texture before display.
- Temporal subject stabilization stores exactly one previous raw mask. Retaining the already-filtered result would recursively retain every older camera, person and depth buffer, causing progressive slowdown and an eventual `EXC_RESOURCE (RESOURCE_TYPE_MEMORY)` termination.
- Vision has a separate queue and a one-request gate.
- Core Image intermediate caching is disabled to reduce unnecessary retained frame data.
- The coverage meter's tiny GPU readback runs on a utility queue and permits only one request at a time, keeping diagnostics out of the live processing queue.
- LiDAR depth thresholding, confidence rejection and mask multiplication occur before enlargement. This avoids full-preview intermediate textures while preserving all information available in the native depth map.
- Expensive masks, keying, plate effects and compositing are capped to 720, 1280, or 1920 pixels on the longest side before processing, not merely at final readback.
- Efficiency, Balanced and Quality target 20, 24 and 30 FPS respectively. The header reports completed Metal renders, not scheduled Core Image graphs, so it represents the preview the user actually received.
- ARKit person/depth semantics are enabled only when the active matte controls require them, and duplicate session-start requests are ignored.
- Capability routing is shared by ARKit configuration, the frame loop and final foreground blend. This prevents hidden LiDAR preferences from activating unnecessary Vision work on devices such as iPhone 15 Plus.
- Custom Photos backgrounds are decoded off the main actor and capped at 2048 pixels to avoid a large transient memory spike.
- Live and planner preferences use a 300 ms trailing save so dragging a slider does not encode and write on every tick; lifecycle transitions flush the latest value.
- Planner analysis is calculated once per SwiftUI body update and reused by all panels.
- Serious/critical thermal pressure reduces the frame target and suppresses the optional depth pass until the device cools; an orange thermometer appears in the header.
- Camera stop and iOS memory-warning paths discard transient sources, mask history and the pending preview, then clear reusable Core Image caches.

### 6.8 Background generation

Built-in backgrounds are Core Image vertical gradients:

- **Soundstage:** dark navy to cool production blue.
- **Dusk:** deep violet to warm orange.
- **Night:** near-black to dark teal.

Custom images are safely downsampled, scaled with aspect-fill, and centered in an expanded working extent. Every background supports:

- Fixed or ARKit-tracked plate movement.
- Scale, horizontal/vertical offset, and rotation.
- Blur, brightness, and saturation.
- A one-tap background-adjustment reset.

Tracked movement is bounded to prevent sudden camera motion from exposing an image edge or making the plate unusable.

### 6.9 Live meters

- Frame rate is averaged and published once per second.
- Matte/screen coverage is sampled every twelfth processed frame.
- Coverage uses a one-pixel `CIAreaAverage` alpha readback on a separate utility queue with a one-request gate.
- Smart coverage reports isolated-subject percentage; Green/Blue report removed-screen percentage.
- Readiness is mode-specific, so an opaque Smart fallback cannot be mistaken for a successful isolation.
- Bare-screen measurement is an explicit on-demand action. It evaluates only pixels whose selected green/blue channel dominates, publishes min/max/spread/stops/hotspot plus a grid, and does not pretend a performer-filled frame is a lighting measurement.

### 6.10 Interactive demo pipeline

If world tracking is unavailable, the engine creates a synthetic reference source and processes it through the current chroma cube, background plate and inspection mode. Settings changes re-render the demo, and high-quality snapshot capture remains available. Demo mode publishes a distinct session state and never reports its generated metrics as live camera evidence.

---

## 7. High Quality Key package

### 7.1 File type

- Extension: `.screencheckkey`
- Uniform type identifier: `com.kamal.screencheck.key-package`
- Conforms to: `com.apple.package`

### 7.2 Package structure

```text
Example.screencheckkey/
├── Input/
│   └── frame_000001.png
├── AlphaHint/
│   └── frame_000001.png
├── Preview/
│   └── live-preview.png
├── manifest.json
└── README.txt
```

### 7.3 Package contents

- **Input:** original unkeyed sRGB camera frame.
- **AlphaHint:** alpha extracted from the current live foreground.
- **Preview:** current composited preview.
- **Manifest:** image metadata, active matte settings, and processing mode.
- **README:** Mac handoff instructions.

### 7.4 Manifest version 2

The JSON manifest contains:

| Field | Meaning |
|---|---|
| `formatVersion` | Package schema version |
| `createdAt` | ISO-8601 timestamp |
| `inputFile` | Original frame path |
| `alphaHintFile` | Alpha-hint path |
| `previewFile` | Live preview path |
| `inputColorSpace` | Currently `sRGB` |
| `width`, `height` | Original frame dimensions |
| `matteMode` | `green-screen`, `blue-screen`, or `smart-background` |
| `keyColor` | Green or blue chroma fallback |
| `keyStrength` | Active strength value |
| `edgeSoftness` | Active edge-softness value |
| `despill` | Active despill value |
| `processingMode` | Device capability mode |
| `personProtection` | Whether a person matte contributed |
| `depthAssist` | Whether supported LiDAR depth was enabled |
| `screenDistanceMeters` | Depth cutoff/range setting |

### 7.5 External finishing boundary

The iOS app only creates and imports files. No desktop keying application, external runtime or model weights are bundled, downloaded or executed by Screen Check. Users choose and operate compatible finishing tools separately under those tools' own licence and model terms.

---

## 8. Planner model

### 8.1 Default state

| Setting | Default |
|---|---:|
| Units | Metric |
| Sensor format | Super 35 · 28.25 × 18.17 mm |
| Focal length | 32 mm |
| Framing | Full |
| Camera distance | Derived from sensor, lens, performer height and framing |
| Screen width | 3.0 m |
| Screen height | 2.4 m |
| Screen material | Cloth |
| Performer height | 1.75 m |
| Performer separation | 1.5 m |
| Performer movement width | 1.0 m |
| Screen-light units | 2 |
| Light distance | 3.0 m |
| Screen exposure vs subject key | −1.0 stops |

### 8.2 Planner controls and ranges

| Group | Control | Values/range |
|---|---|---|
| Camera | Units | Metric or feet/inches |
| Camera | Sensor format | Full Frame, Super 35, Micro Four Thirds, Super 16, iPhone equivalent, or custom gate |
| Camera | Focal length | 1...600 mm |
| Camera | Custom gate | Width and height, 1...100 mm |
| Camera | Framing | Full, Waist, Close |
| Camera | Direct camera-to-performer distance | Optional, 0.5...30 m |
| Screen | Width | 1.5...14.0 m, 0.1 m step |
| Screen | Height | 1.5...8.0 m, 0.1 m step |
| Screen | Material | Cloth, Paint, Blue |
| Performer | Height | 1.2...2.1 m, 0.01 m step |
| Performer | Distance from screen | 0.2...10.0 m, 0.05 m step |
| Performer | Movement width | 0...8.0 m, 0.1 m step |
| Screen lighting | Units | 1, 2, or 4 |
| Screen lighting | Distance from screen | 0.5...8.0 m, 0.1 m step |
| Screen lighting | Screen vs subject key | −2.5...+0.5 stops, 0.25-stop step |

### 8.3 Lens and framing calculations

The planner uses the selected sensor dimensions. Presets are Full Frame/iPhone-equivalent `36 × 24 mm`, Super 35 `28.25 × 18.17 mm`, Micro Four Thirds `17.3 × 13 mm`, and Super 16 `12.52 × 7.41 mm`; Custom uses the entered gate.

- Horizontal FOV: `2 × atan(sensor width / (2 × focal length))`
- Vertical FOV: `2 × atan(sensor height / (2 × focal length))`
- Camera-to-subject distance is derived from performer height, framing factor, and vertical FOV.
- If direct distance is enabled, the entered camera-to-performer distance replaces that derived value.
- Camera-to-screen distance equals camera-to-subject distance plus performer separation.
- Framed width/height at any distance use the corresponding FOV.

The framing factors are production approximations:

| Framing | Factor |
|---|---:|
| Full | 0.85 |
| Waist | 1.55 |
| Close | 3.10 |

### 8.4 Screen materials

Approximate RGB reflectance values:

| Material | R | G | B |
|---|---:|---:|---:|
| Cloth | 0.11 | 0.46 | 0.14 |
| Paint | 0.09 | 0.52 | 0.12 |
| Blue | 0.10 | 0.16 | 0.42 |

The green channel is evaluated for green materials and the blue channel for blue material.

### 8.5 Spill calculation

- The screen is represented as a rectangular polygon in 3D.
- A point-to-polygon form factor estimates how much of the screen is visible to a point on the performer.
- The polygon is clipped against the receiver’s tangent plane.
- Solid-angle edge integration produces a normalized form factor.
- The material reflectance vector is multiplied by the form factor.
- Reflectance is scaled by `2^(screen exposure stops)` to represent screen brightness relative to the subject key.
- The active screen channel becomes the predicted spill ratio.
- Channel imbalance is retained for diagnostic use.
- Center, left-edge and right-edge performer positions across the entered movement width are evaluated; dashboard and stand-off results use the worst sampled position.

Spill classifications:

| Ratio | Classification | Guidance |
|---:|---|---|
| `< 2%` | Negligible | Clean |
| `< 6%` | Despillable | Standard despill should handle it |
| `< 15%` | Visible edges | Edge contamination will be visible |
| `< 30%` | Edge grade | Expect edge grading |
| `≥ 30%` | Move or flag | Change the setup |

Minimum separation is found by binary search over `0.02...40 m` for clean and workable tolerances.

### 8.6 Screen-lighting evenness

- The screen is sampled on an 18×18 grid for dashboard analysis.
- One, two, or four idealized light positions are generated from screen dimensions.
- Each light contributes according to incidence angle and inverse-square distance.
- Evenness in stops is `log2(maximum illuminance / minimum illuminance)`.
- The interface also recommends a minimum starting light distance of half the screen height and asks the crew to work toward a 45-degree incidence, then verify using the live bare-screen measurement.

Evenness classifications:

| Variation | Tone |
|---:|---|
| `≤ 0.33 stops` | Excellent |
| `≤ 0.50 stops` | Good |
| `≤ 1.00 stop` | Usable |
| `> 1.00 stop` | Poor |

### 8.7 Planner output

The planner provides:

- A prioritized natural-language verdict.
- Overhead spill heat map.
- Camera frustum and screen-coverage view.
- Clean-key distance line.
- Through-the-lens screen coverage.
- Screen-lighting evenness heat map.
- Camera-to-performer and camera-to-screen distances.
- Spare or missing screen width per side.
- Maximum camera distance before the screen runs out.
- Back and profile spill estimates.
- Clean and workable stand-off values.
- Evenness in stops.
- A Plan → Check panel comparing expected screen distance against live coverage, measured LiDAR distance, sustainable FPS and bare-screen evenness.
- Named local setup save/load/delete and a shareable plain-text setup report.

Verdict priority is:

1. Screen does not cover the frame.
2. Clean-key separation conflicts with available screen coverage.
3. Performer is closer than the clean-key estimate.
4. Screen-lighting variation exceeds 0.5 stops.
5. Screen lights are closer than the recommended starting distance.
6. Setup is acceptable.

The UI explicitly warns that spill is a one-bounce floor estimate. Real floors, ceilings, wardrobe, flags, and fixture shape can increase or alter the result.

---

## 9. Visual design system

### 9.1 Design principles

- Dark live production-monitor presentation with an adaptive light/dark planning workspace.
- Apple-native SwiftUI materials and controls.
- Rounded continuous corners.
- SF Symbols rather than custom control glyphs.
- Clear hierarchy with compact uppercase section labels.
- Monospaced numeric readouts.
- Neutral operational controls; color is reserved primarily for measured state and severity.
- Severity follows one ordered warm ramp instead of unrelated hues, and warnings retain text labels so meaning never depends on color alone.

### 9.2 Core planner colors

| Token | Light RGB / Dark RGB | Purpose |
|---|---|---|
| Background | 246,247,249 / 14,17,20 | Main surface |
| Panel | 255,255,255 / 23,27,32 | Cards |
| Raised panel | 237,240,244 / 30,36,42 | Elevated surface |
| Line | 204,210,218 / 54,63,72 | Borders/dividers |
| Ink | 20,24,29 / 235,234,230 | Primary text |
| Dim ink | 69,77,87 / 165,174,184 | Secondary text |
| Faint ink | 76,84,94 / 171,180,190 | Explanatory copy with improved contrast |
| Clean → danger | Muted mauve → bright warm orange | Ordered severity ramp |

### 9.3 Accessibility

- Important visual groups combine their children into coherent VoiceOver elements.
- Buttons and menus have explicit accessibility labels.
- Sliders publish readable values.
- Planner canvases provide descriptive labels and values.
- Status is conveyed through text as well as color.
- Dynamic text is allowed where practical, with fixed sizing reserved for compact production labels.

---

## 10. Project architecture and file map

```text
ScreenCheck/
├── ScreenCheck.xcodeproj/
│   ├── project.pbxproj
│   └── xcshareddata/xcschemes/ScreenCheck.xcscheme
├── ScreenCheck/
│   ├── ScreenCheckApp.swift
│   ├── RootView.swift
│   ├── LivePrevisView.swift
│   ├── LivePrevisEngine.swift
│   ├── HighQualityKeyPackage.swift
│   ├── AboutView.swift
│   ├── ContentView.swift
│   ├── ScreenCheckModel.swift
│   ├── VisualizationViews.swift
│   ├── AppTheme.swift
│   ├── Info.plist
│   └── Assets.xcassets/
├── ScreenCheckTests/
│   └── ScreenCheckModelTests.swift
├── README.md
├── ScreenCheck-Critique-Implementation.md
└── ScreenCheck-Complete-Project.md
```

### Source responsibilities

| File | Responsibility |
|---|---|
| `ScreenCheckApp.swift` | Application entry point |
| `RootView.swift` | Shared session ownership and Previs/Plan tab navigation |
| `LivePrevisView.swift` | Live UI, sampling/measurement actions, still capture, clean feed, sheets, Photos selection, responsive layout |
| `LivePrevisEngine.swift` | AR session, capability detection, matte generation, normalized/sample chroma, demo pipeline, Core Image composite, metrics |
| `HighQualityKeyPackage.swift` | Export type, manifest, package encoding, handoff instructions |
| `AboutView.swift` | Bundle-derived app identity, method transparency, CRIT Studio contacts and creator credit |
| `ContentView.swift` | Planner form, verdict, panels, and readout grid |
| `ScreenCheckModel.swift` | Shared session, named setups/reporting, sensor optics, geometry, exposure-aware spill, evenness, and verdict model |
| `VisualizationViews.swift` | SwiftUI Canvas spill, lens, and evenness diagrams |
| `AppTheme.swift` | Design tokens, severity colors, shared panel/label components |
| `Info.plist` | Permissions, orientations, app metadata, exported package type |
| `ScreenCheckModelTests.swift` | Unit and integration-style model tests |

---

## 11. State and lifecycle

### Live view state

- `RootView` owns one `ScreenCheckSession` as a `@StateObject`; both tabs observe it.
- `LivePrevisEngine` is owned by `LivePrevisView` as a `@StateObject`.
- Planner state, live preferences and observed on-set values live in the shared session. Transient Photos selection and sheet/action state remain local to `LivePrevisView`.
- Every settings change is forwarded to the engine.
- Matte, inspection, preview-quality, background-transform, tracking, and built-in preset choices are encoded to `UserDefaults` and restored on launch.
- Planner geometry and lighting values are likewise persisted as a versioned Codable state.
- Named setups are stored as Codable records. The shared report is generated on demand from the current plan and latest live measurements.
- Plan changes continuously synchronize expected screen distance into Previs. LiDAR distance can be applied back to performer separation only through an explicit user action.
- The selected Photos image itself is not copied into app storage; the user reselects it after relaunch, while all plate adjustments remain saved.
- The engine starts when the view appears or the scene becomes active.
- The engine stops when the view disappears or the scene becomes inactive.

### Session states

- Idle
- Requesting camera permission
- Starting
- Running
- Interrupted
- Denied
- Unavailable
- Interactive demo
- Failed with a message

The placeholder UI translates these states into clear user-facing guidance.

---

## 12. Privacy and permissions

### Requested permissions

| Permission | Reason |
|---|---|
| Camera | Live keyed preview and screen evaluation |
| Photo library selection | User-selected replacement background |
| Photo library add-only | Save an explicitly requested previs still |

### Data behavior

- Live camera processing is local to the device.
- The app has no server component.
- There are no analytics or advertising SDKs.
- No third-party packages are linked.
- A still frame leaves the app only when the user explicitly exports a key package or taps Save Still.
- A background image is accessed only after the user selects it through Photos.

---

## 13. Build and run

### Requirements

- macOS with Xcode 26 or later.
- iOS 17 SDK or later.
- iPhone or iPad for live camera, ARKit segmentation, and LiDAR.
- Apple Developer signing team for installation on physical hardware.

### Xcode workflow

1. Open `ScreenCheck.xcodeproj`.
2. Select the `ScreenCheck` scheme.
3. Select an iPhone or iPad.
4. In Signing & Capabilities, select your development team if required.
5. Build and run.
6. Accept camera permission.
7. Accept Photos permission only when selecting a custom background.

### Simulator behavior

The simulator supports interface and planner verification through **Interactive Demo** mode. The generated reference frame passes through the real key/composite controls, but AR world tracking, camera segmentation and LiDAR measurements still require physical hardware.

### Command-line build

```sh
xcodebuild \
  -project ScreenCheck.xcodeproj \
  -scheme ScreenCheck \
  -destination 'generic/platform=iOS' \
  CODE_SIGNING_ALLOWED=NO \
  build
```

### Debug launch arguments

| Argument | Effect |
|---|---|
| `--expand-key-controls` | Opens Matte Controls at launch |
| `--smart-matte` | Starts in Smart mode |
| `--show-high-quality-key` | Opens the High Quality Key sheet |

These arguments are intended for previews, automated screenshots, and development verification.

---

## 14. Testing and validation

The test target contains 35 tests covering:

1. Known point-to-polygon form-factor result.
2. No spill from a polygon behind the receiver.
3. Spill decreases as the performer moves away.
4. Moving screen lights back improves evenness.
5. LiDAR is preferred when available.
6. Person segmentation is the non-LiDAR fallback.
7. Vision is the legacy world-tracking fallback.
8. Green and Blue modes remove only their selected key color.
9. Smart Background follows the LiDAR → person → Vision ladder.
10. High Quality Key package contains the required folder structure.
11. Live overlays fit standard and zoomed iPhone widths.
12. Live overlays cap their width on iPad.
13. Smart mode does not mistake an opaque fallback for a valid subject matte.
14. Green/Blue coverage reports the removed-screen percentage.
15. Tracked-plate motion remains within bounded translation and scale limits.
16. Complete live preferences survive Codable round-trip encoding.
17. Live preferences save and restore through an isolated defaults domain.
18. Planner state saves and restores through an isolated defaults domain.
19. Working-resolution scale is applied before expensive preview effects.
20. Preview-quality profiles keep FPS, render dimensions, and color-cube sizes bounded.
21. LiDAR calibration uses a reliable median and rejects sparse or invalid samples.
22. Exposure-normalized chroma treats dark and bright areas of the same green consistently.
23. A sampled screen color keys matching chromaticity while preserving neutral subject color.
24. Auto Key detects a blue screen and supplies bounded starting settings.
25. Bare-screen measurement reports code spread, stops, hotspot and grid data.
26. Sensor format changes camera geometry and direct camera distance overrides the derived value.
27. Relative screen exposure changes predicted spill by the correct stop ratio.
28. Metric and imperial measured-distance formatting.
29. New planner fields survive a Codable round trip.
30. Eyedropper taps map through aspect-fill crop to the visible source pixel.
31. The Metal preview surface retains only the latest lazy frame and clears it safely.
32. Subject-mask history replaces the prior raw mask rather than chaining frame graphs.
33. The Metal drawable remains bounded by processed-image resolution instead of native Retina resolution.
34. Non-LiDAR routing ignores saved Depth Assist state and performs no substitute mask work when people protection is off.
35. LiDAR routing activates depth only on capable hardware and suppresses it consistently during thermal pressure.

### Latest verification result

- Production Swift sources: **type-checked successfully for arm64 iOS 17 using the iPhoneOS 26.5 SDK**.
- Xcode device build-for-testing: **app and 35-test XCTest bundle compiled and linked successfully for generic arm64 iOS hardware**.
- Xcode Release device build: **optimized app compiled, linked and validated successfully for generic arm64 iOS hardware**.
- Test execution: **not run in this sandbox because CoreSimulatorService is unavailable**. Run the suite from Xcode on the attached device or any installed compatible simulator for runtime results.

### Test command example

```sh
xcodebuild \
  -project ScreenCheck.xcodeproj \
  -scheme ScreenCheck \
  -destination 'platform=iOS Simulator,name=iPhone 15 Plus' \
  CODE_SIGNING_ALLOWED=NO \
  test
```

Use any installed compatible simulator name if iPhone 15 Plus is unavailable.

---

## 15. Known limitations

- Live camera and matte quality cannot be evaluated in the simulator.
- Smart mode without LiDAR isolates people, not arbitrary foreground objects.
- LiDAR foreground range can retain unwanted nearby background surfaces if set too far.
- Vision fallback updates less frequently than the ARKit camera stream.
- Hair, motion blur, transparency, reflections, and fine fabric can still require final keying work.
- Built-in backgrounds are stylized gradients, not modeled 3D environments.
- Tracked backgrounds use bounded 2D plate parallax; they do not reconstruct a photoreal 3D scene or create geometry-aware perspective changes.
- The app does not currently record keyed video.
- Clean Feed relies on normal wired/AirPlay screen mirroring; it is not a custom synchronized second-device network monitor.
- A user-selected Photos background is intentionally not copied into app storage and must be reselected after relaunch.
- The planner assumes a centered performer and idealized screen/light geometry.
- The planner does not model multiple chroma bounces, colored wardrobe, floor reflection, ceiling reflection, flags, or fixture beam shape.
- High Quality Key currently exports one still frame per package.

---

## 16. Recommended roadmap

### Near term

- Add numerical chroma/depth confidence histograms beyond the current Matte and Depth inspection views.
- Add rename plus portable file import/export for the current named local setups.
- Add shareable visual or PDF reports to complement the implemented text report.

### Medium term

- Record preview-quality composited video.
- Add optical-flow-guided mask stabilization and depth hole filling beyond the current temporal blend/confidence rejection.
- Support separate subject and prop depth zones.

### Advanced

- Upgrade tracked plates to optional reconstructed 3D scenes with calibrated anchors.
- Use Metal kernels for more sophisticated chroma edge treatment.
- Add foreground color correction and light-wrap preview.
- Add multi-person and object-aware segmentation where supported.
- Add authenticated, low-latency synchronized iPhone/iPad monitor mode if user research shows normal display mirroring is insufficient.
- Add ProRes/Log-aware capture and color-management options where hardware permits.

---

## 17. Release checklist

Before TestFlight or App Store distribution:

- Confirm the Apple Developer team and production bundle identifier.
- Add final App Store icons and marketing artwork.
- Add privacy policy and support URLs.
- Confirm camera and Photos purpose strings in every localization.
- Test Green, Blue, and Smart modes on at least one LiDAR and one non-LiDAR device.
- Test all supported orientations on iPhone and iPad.
- Test denied and restricted permission states.
- Test thermal behavior during an extended live session.
- Test memory behavior with large Photos backgrounds.
- Test High Quality Key export/import through Files and AirDrop.
- Confirm tool-neutral external-finishing wording and licensing boundary.
- Add UI automation for mode switching and background selection.
- Run accessibility audit for VoiceOver, contrast, and larger text sizes.
- Archive a signed Release build and validate it through Xcode Organizer.

---

## 18. Current completion status

| Area | Status |
|---|---|
| Native SwiftUI app shell | Complete |
| iPhone and iPad layout | Complete |
| Green-screen live key | Complete |
| Blue-screen live key | Complete |
| Reliable mode switching | Complete |
| Smart Background mode | Complete |
| LiDAR depth fusion | Complete |
| Non-LiDAR person fallback | Complete |
| Vision fallback | Complete |
| Built-in backgrounds | Complete |
| Photos background | Complete |
| Background transform/look editor | Complete |
| Fixed and ARKit-tracked plates | Complete |
| Matte/Camera/Depth inspection | Complete |
| Temporal matte stabilization | Complete |
| Depth confidence rejection | Complete |
| Physical-screen LiDAR calibration | Complete |
| Adaptive preview/thermal behavior | Complete |
| Tap-to-sample and Auto Key | Complete |
| Exposure-normalized chroma | Complete |
| Bare-screen evenness measurement | Complete |
| Interactive no-camera demo | Complete |
| Still capture to Photos | Complete |
| Clean mirrored feed | Complete |
| Sensor formats/free focal length/imperial units | Complete |
| Plan ↔ Previs measured-state loop | Complete |
| Planner calculations | Complete |
| Planner visualizations | Complete |
| High-quality still handoff | Complete |
| Automated model tests | Complete |
| Live keyed video recording | Not implemented |
| Latest settings persistence | Complete |
| Multiple named local setups | Complete |
| Shareable text setup report | Complete |
| Portable setup files / PDF report | Not implemented |
| Custom second-device network monitor | Not implemented; normal wired/AirPlay mirroring supported |
| App Store submission assets | Not finalized |

---

## 19. Glossary

- **Chroma key:** Removal of a selected green or blue color range.
- **Despill:** Reduction of green or blue contamination on foreground edges.
- **Matte:** A grayscale/alpha mask separating foreground from background.
- **Previs:** A real-time approximation of a later visual effect.
- **Person segmentation:** A machine-generated mask identifying people.
- **Scene depth:** Per-pixel distance measured or inferred by the device.
- **LiDAR:** Active depth-sensing hardware on supported Pro devices.
- **Form factor:** A geometric estimate of how much of a radiating surface is visible to a receiver.
- **Evenness:** Brightness variation across the screen, expressed in stops.
- **Alpha hint:** The app’s live foreground alpha exported to guide a higher-quality keyer.

---

## 20. Project artifacts

- `ScreenCheck.xcodeproj` — buildable Xcode project.
- `README.md` — concise setup and feature overview.
- `ScreenCheck-Complete-Project.md` — the authoritative master product, design, engineering, testing, and handoff document.
- `ScreenCheck-Critique-Implementation.md` — recommendation traceability, adjustments, deferred product decisions, and verification record.
- `screencheck-smart-background.png` — current Smart mode layout reference.
- `screencheck-controls-redesigned.png` — current chroma controls layout reference.
- `ScreenCheck-iOS.zip` — packaged source project and documentation.

---

## 21. Final implementation note

Screen Check is currently a complete functional prototype and development-ready iOS project. Its sampled and automatic Green/Blue key, stabilized Smart matte, confidence-filtered LiDAR assist, real-screen evenness measurement, interactive demo, tracked plate editor, still capture, clean mirrored feed, shared Plan/Previs session, named setups, reports, adaptive performance, sensor-aware planner, responsive layout and export package compile for the physical-device target. Production release work should focus on extended physical-device image-quality testing, video and true network-monitor decisions, portable/PDF reports, privacy/support materials, positioning, and final App Store preparation.
