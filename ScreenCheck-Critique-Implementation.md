# Screen Check — Critique Implementation Record

This file records how the recommendations in `ScreenCheck-Critique.md` were handled. The critique was treated as review input; product changes were made only where they supported the user's request and the existing Screen Check intention.

## Outcome

The app now has one connected workflow: **plan it → check it on set → feed measurements back into the plan → save or share the result**.

| Critique recommendation | Result | Implementation |
|---|---|---|
| Tap-to-sample and Auto Key | Implemented | Sample mode accepts a tap on uncovered fabric. Auto Key determines green/blue, samples chromaticity and supplies bounded Strength, Edge and Despill starting values. |
| Normalize chroma dominance | Implemented | Default key dominance is divided by peak RGB exposure. Sampled mode compares normalized chromaticity. |
| Demo without camera hardware | Implemented with an equivalent approach | A generated reference scene runs through the real key/composite pipeline. This avoids adding licensed footage and remains interactive on simulator or unsupported hardware. The status says `DEMO FRAME`. |
| Truthful status and less false precision | Implemented | Non-running camera states are shown directly. Planned distances use 0.1 m or 6-inch steps; measured LiDAR values retain finer precision. |
| Sensor format, free lens and imperial units | Implemented | Full Frame, Super 35, Micro Four Thirds, Super 16, iPhone-equivalent and custom gates; 1–600 mm focal entry; metric or feet/inches. |
| Shared Plan/Previs state | Implemented | One `ScreenCheckSession` owns both tabs. Plan preloads screen distance; Previs returns coverage, LiDAR distance, sustained FPS and screen evenness. Measured distance can be applied back explicitly. |
| Still capture and external viewing | Partly implemented | Save Still writes the composite to Photos. Clean Feed hides all app navigation for normal wired or AirPlay mirroring; its exit control fades and reappears on tap. A custom second-device network transport is deferred because it requires pairing, latency, privacy and security architecture beyond a safe UI-only change. |
| Ordered severity, contrast and neutral chrome | Implemented | Planner surfaces are adaptive light/dark, explanatory text contrast is raised, severity uses an ordered warm ramp, explicit decision lines are drawn, and live controls are neutral. |
| Matte while sliders move | Implemented | Dragging a key slider temporarily selects Matte inspection and restores the prior view on release. |
| Measure the real screen | Implemented | The bare-screen action reports min/max code value, spread, stops, hotspot and a grid map, with a visible ≤32-code target. |
| Named setups and report | Implemented | Local named setup save/load/delete and a shareable text report are available from Plan. Portable setup files and PDF/image reports remain roadmap work. |
| Exposure and fixture guidance | Implemented | Screen exposure relative to subject key scales predicted spill. The app recommends an initial light distance from screen height and asks the crew to verify the physical result. |

## Additional improvements made

- Added direct camera-to-performer distance and performer movement width; spill and stand-off calculations evaluate the worst sampled movement position.
- Made Green/Blue/Smart an always-visible segmented selector.
- Added control auto-hide and preserved the translucent overlay layout.
- Added Photos add-only permission copy and idle-timer handling during live monitoring.
- Added test coverage for new chroma, visible-pixel tap mapping, measurement, sensor, unit, exposure and persistence behavior.
- Replaced the per-frame `CGImage` readback and SwiftUI image publication with a direct Core Image-to-Metal preview surface. The surface retains only the newest frame, coalesces draw requests and caps GPU work in flight without reducing key quality, resolution profiles or available masks.
- Moved coverage readback off the live render queue and changed the header FPS metric to count completed Metal frames, making both the pipeline and its performance indicator more reliable.
- Fixed the memory-termination root cause: temporal stabilization had retained its filtered lazy result, recursively chaining every prior camera, person and LiDAR mask. It now stores one raw mask only, preserving adjacent-frame stabilization with bounded memory.
- Capped drawable resolution to the processed preview dimensions, limited the drawable pool to two and added memory-warning/cache cleanup. This removes unnecessary Retina-sized GPU allocation without reducing information present in the processed source.
- Moved LiDAR depth/confidence filtering to the sensor buffers' native resolution and scales only the completed protection mask, removing large intermediate textures without discarding any depth samples.
- Centralized the runtime feature route so ARKit semantics, Vision work, LiDAR filters and foreground protection agree. A hidden saved Depth Assist preference no longer causes neural-mask work on a non-LiDAR iPhone after Protect People is disabled.

## Product decisions intentionally left open

- **Name, App Store subtitle and price:** these are positioning/business decisions and were not silently changed in code.
- **True second-device monitoring:** the implemented Clean Feed is immediately useful with Apple display mirroring, but it is not presented as a synchronized network monitor.
- **Final-key claims:** the live key remains a previs and diagnostic tool. Hair, transparency, motion blur, reflections and production color management still require final compositing judgment.

## Verification

- Production Swift sources type-check for arm64 iOS 17.
- The app and the 35-test XCTest bundle compile and link for a generic arm64 iOS device.
- Simulator test execution was unavailable in the automation environment because CoreSimulatorService was not running; real camera, LiDAR, thermal and Photos behavior still requires a physical-device pass before release.
