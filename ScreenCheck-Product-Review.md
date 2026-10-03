# Screen Check: product review and roadmap

**Reviewed:** 3 October 2026, against Screen Check 2.0 (build 2)
**Scope:** README, `ScreenCheck-2.0-Engine.md`, `ScreenCheck-Complete-Project.md`, `ScreenCheck-Critique-Implementation.md`, the nine screenshots and `ScreenCheck.xcodeproj.zip`. The review also covers competing apps, Apple platform APIs, matting and keying research, and open-source projects, all checked online in October 2026.

> **Limit of this review.** `ScreenCheck.xcodeproj.zip` contains only `project.pbxproj`, the scheme and an `xcuserdata` folder. The project references 13 Swift files, `ScreenCheckKernels.metal`, `Info.plist`, `Assets.xcassets` and the test file, but none of them are in the repository. Everything below about the engine is based on the design documents, not on reading the code. Section 3.1 explains why this matters beyond the review.

---

## 1. Verdict

Yes, this can become a viable product, but not in its current framing.

Today, Screen Check keys **the iPhone's own camera**. An indie crew usually shoots on something else: an FX3, a Pocket 6K, an R5 C, a Komodo. The footage that will actually be keyed in post has a different lens, field of view, position, exposure, codec, depth of field and motion blur. So right now the app answers *"what would this look like if I shot it on my iPhone?"* The question you started with is *"will the shot from my camera key, and what will it look like?"*

Closing that gap is the single most important change. After it, the clearest place for this product is:

> **The green-screen assistant for small crews: plan the screen, check the key on the real camera feed, record takes with everything post needs, and hand off a clean package.**

That is narrower than "virtual production on a phone", and it should be. Lightcraft Jetset already covers 3D virtual production on iPhone. It costs $20/month for Pro and $80/month for Cine, and it supports Gaussian splats, 3D garbage mattes, external cinema cameras through Accsoon SeeMo, and a full Blender/Unreal/Nuke export pipeline [1]. Screen Check 2.0's *Extend screen*, *Tracked 3D* and *360°* features overlap heavily with Jetset. The planner, the screen-quality measurements and the plan-to-set loop don't. Neither Jetset nor anyone else does those well, and they are what your users will pay for.

## 2. What is already strong (keep it)

- **The planner is unique.** None of the apps reviewed has point-to-polygon form-factor spill estimation with movement envelopes, lighting evenness and a plain-language verdict ("The frame is 2.17 m wider than the cloth here..."). Green Screener and Cine Meter measure a screen you have already built. Screen Check tells you before you build it.
- **The plan → check → plan loop.** Measured LiDAR distance, coverage, FPS and evenness flow back into the plan. This is the start of a real on-set workflow.
- **Serious realtime engineering.** The 2.0 notes show a correct diagnosis of the 1.0 slowdowns. The fixed frame gate was rounding 24 fps down to 20, ARKit's buffer pool was being starved, and the temporal mask was recursively retaining old frames. The fixes are also right: an accumulating `FramePacer`, a fused single-pass composite kernel, sensor-resolution masks with guided upsampling, rendering into owned buffers, and a GPU-vs-CPU parity test.
- **Truthful labelling.** The capability ladder is chosen at runtime with no device allow-list, labels are honest, and the app fails safe to passthrough. Keep this; it is what earns trust on set.
- **The file-based high-quality key bridge.** The `Input/` + `AlphaHint/` layout matches what CorridorKey expects. CorridorKey is the open-source neural keyer from Corridor Digital; it takes a raw frame plus a rough alpha hint and returns straight colour and linear alpha as 16/32-bit EXR [2]. Its licence forbids repackaging and selling it, so keeping the bridge file-based and tool-neutral is the correct call.

## 3. Problems to fix first

These come before any new feature. Each one either makes the app less trustworthy or makes it harder to develop.

### 3.1 The source code is not in the repository

The zip has the project file but no sources. If the Mac holding the code fails, the product is gone. Nobody, including future you, can review, diff or bisect a regression, and you can't set up CI.

Fix:

- Commit the full `ScreenCheck/` and `ScreenCheckTests/` folders unzipped, so git can diff them.
- Add a `.gitignore` for `xcuserdata/`, `*.xcuserstate`, `DerivedData/`, `__MACOSX/` and `.DS_Store`. The current zip ships `UserInterfaceState.xcuserstate` and `__MACOSX` resource forks.
- Add CI: a GitHub Actions macOS runner or Xcode Cloud that runs `xcodebuild test` on a simulator on every push. The 48 tests are already there and deserve to run automatically.
- If the code must stay private, make the repository private. Don't leave the code out.

### 3.2 Nothing in 2.0 has run on a real device yet

The 2.0 document says so plainly: *"Not yet verified: live camera, LiDAR, thermals and real frame rates."* Every 2.0 feature (60 fps, plane lock, Extend screen, AI object assist, Tracked 3D, 360°) is unproven where it matters. Until the 10-item device checklist in `ScreenCheck-2.0-Engine.md` §6 passes on at least one LiDAR iPhone and one non-LiDAR iPhone, treat 2.0 as a branch, not a release.

### 3.3 The documentation contradicts itself

| Topic | README | Complete-Project.md | 2.0 Engine |
|---|---|---|---|
| Minimum iOS | iOS 26 | iOS 17.0 | iOS 26 |
| Version | 2.0 (banner only) | 1.0 (build 1) | 2.0 (build 2) |
| Quality mode FPS | 20 / 24 / 30 | 20 / 24 / 30 | 24 / 30 / 60 |
| Key method | Core Image key table generated at launch | 24³/32³/48³ cube, 35 ms debounce | Metal kernel, no LUT |
| Test count | 35 | 35 | 48 |
| Source package | not mentioned | lists `ScreenCheck-iOS.zip` as the "packaged source project" | not mentioned |

The master document calls itself "authoritative", but on the live pipeline it is out of date. Make one document current and turn the others into short changelogs that point to it.

### 3.4 Screen measurements are taken from a processed iPhone image

Bare-screen evenness ("spread ≤ 32 code values") and Auto Key read from ARKit's video feed. That image has been through the iPhone's processing: auto exposure, auto white balance, tone mapping and noise reduction. Two consequences:

1. Readings drift as AE/AWB react to the performer walking in, so the key and the measurements are not stable over a take.
2. Code values from the iPhone don't transfer to the cinema camera's code values. A screen that reads 28 codes of spread on the iPhone could read differently on an FX3 in S-Log3.

Fix:

- After **Auto Key / Sample**, lock exposure, white balance and focus. ARKit exposes the primary capture device for this through `ARWorldTrackingConfiguration.configurableCaptureDeviceForPrimaryCamera` (iOS 16+) [3].
- Report evenness in **stops relative to the screen median**, not raw code values. Stops carry over between cameras far better than code values do.
- Show the locked shutter, ISO and white balance in the HUD so the user knows the measurement conditions.

### 3.5 Latency has never been measured

For an operator framing a shot, glass-to-glass latency matters as much as frame rate. The pipeline goes ARKit delivery → pacer → mask build → upsample → composite → up to 2 drawables in flight. That could add up to 80–150 ms, but nobody knows. Measure it: film a millisecond timer on a second screen through the app and photograph both together. Then show the figure in a diagnostics overlay.

### 3.6 The UI is portrait-first, but rigged phones run in landscape

All the screenshots are portrait. On a camera cage or a C-stand, the phone or iPad is landscape, like every on-camera monitor. Section 8 covers a landscape rig layout.

### 3.7 The bundled demo undersells the app

The Interactive Demo shows a block figure on a flat colour. App Store reviewers, and any potential user without a green screen, will judge the keyer by that. Ship a real 5–10 second green-screen clip you shot yourself, with hair, motion blur and a prop, and run it through the real pipeline.

---

## 4. Market check (October 2026)

| Product | What it does | Price | What it lacks for this user |
|---|---|---|---|
| Lightcraft Jetset [1] | iPhone virtual production: tracking, live composite, splats, 3D garbage mattes, SeeMo cinema-camera input, Autoshot export to Blender/Unreal/Nuke | Free / $20 mo Pro / $80 mo Cine | Pre-production planning, screen QC, simplicity; a 3D pipeline is assumed |
| Skyglass [4] | AI-generated or Unreal environments, background remover, AI relighting | ~$5 / $13 / $25 mo | Aimed at creators; not about keying quality or VFX handoff |
| CamTrackAR (FXhome) [5] | Records iPhone video plus camera track; chroma key; exports to HitFilm, Blender, AE, FBX | Free; $4.99 mo or $29.99 | Only the iPhone camera; no QC or planning |
| Green Screener [6] | Screen evenness, banding view, spill analyser, simple keyer | $9.99 | QC and a simple keyer only; no planner, recording or handoff |
| Cine Meter [7] | Waveform and false colour for checking screen evenness | Paid | Measures only; no keying |
| Artemis Pro / Cadrage [8] | Director's viewfinders that simulate a cinema camera and lens FOV | $29.99 / $19.99 | No keying |
| Accsoon SeeMo 4K [9] | HDMI → iPhone/iPad monitor adapter with its own app | Hardware | Monitoring only, no key |

**The gap:** no single tool takes a crew from *"will this screen and this distance work?"* through *"is it keying cleanly on the A-camera right now?"* to *"here's the take package for post"*, without first building a 3D scene. That is Screen Check's space.

**Timing helps.** iPhones are now credible A-cameras. *28 Years Later* was shot largely on iPhone 15 Pro Max rigs [10]. The iPhone 17 Pro adds ProRes RAW, Apple Log 2, genlock and external timecode, and Blackmagic's Camera ProDock adds HDMI out, BNC timecode and genlock [11]. CorridorKey has given the indie VFX community a free, high-quality keyer that needs an alpha hint and a good plate, which is exactly what an on-set tool can supply.

---

## 5. The core extension: key the real camera

### 5.1 Input options and what the platform actually allows

| Path | Hardware | Platform facts | Recommendation |
|---|---|---|---|
| **iPad + HDMI capture dongle** | Camera HDMI → UVC capture card (Elgato Cam Link class, or MS2130-class dongles from ~$20) → USB-C iPad | iPadOS 17+ supports external UVC cameras through `AVCaptureDevice.DeviceType.external` on USB-C iPads [12] | **Build this first.** Cheapest route to keying the real footage. Latency varies by card; one user report puts cheap dongles at 6–11 frames [13]. Test a few and publish a "known good" list. |
| **iPhone as the A-camera** | iPhone 15 Pro and later, optionally with Blackmagic ProDock | AVFoundation gives Apple Log, ProRes, 4K, manual control and the telephoto lens. The LiDAR depth camera is also available through AVFoundation (`builtInLiDARDepthCamera`, streaming depth up to 320×240) [14]. You lose ARKit's 6-DoF tracking. | **Build second.** Record a clean Log/ProRes plate for post while previewing the composite from a downscaled copy. |
| **iPhone + external camera over USB** | Accsoon SeeMo | iPhone does **not** support UVC; Apple DTS has confirmed it [15]. SeeMo uses MFi and its own app. Jetset integrates SeeMo, so a partnership is possible. | Contact Accsoon about an SDK. Don't build around generic dongles on iPhone. |
| **Network feed** | Wireless transmitter, laptop or another iPhone | The NDI SDK supports finding, sending and receiving on iOS [16]. SRT/RTMP is available through HaishinKit (BSD-3, ~3k stars) [17]. | Later. Useful for video village and for sending the composite out. |
| **iPhone on the cinema camera as a tracker** | iPhone rigged on the A-camera + SeeMo | Needs offset and lens calibration between the two cameras | Don't compete with Jetset Cine here. Export OpenTrackIO / USD so the two tools can work together. |

### 5.2 Architecture change: one keyer, many frame sources

Today the engine is built around `ARFrame`. Split it:

```
protocol FrameSource {
    // pixel buffer (YCbCr or BGRA), presentation time, optional:
    // intrinsics, camera pose, LiDAR depth + confidence, person mask, timecode
    var frames: AsyncStream<SourceFrame> { get }
    var capabilities: SourceCapabilities { get }   // tracking? depth? log? fps?
}

ARKitSource        // today's path: tracking, LiDAR, person segmentation
AVCaptureSource    // iPhone as A-camera: Log/ProRes, tele lens, LiDAR via AVFoundation
ExternalUVCSource  // iPad + HDMI capture card
NetworkSource      // NDI / SRT
FileSource         // recorded clips: demo footage, tests, review of takes
```

The fused `scComposite` kernel, the mask pipeline and the QC overlays then work on any source, and the capability-routing system you already have decides what each source can do. `FileSource` matters most for reliability: it lets you run real footage through the real engine in automated tests (section 9).

---

## 6. Roadmap

Effort estimates assume one experienced iOS developer. Value is judged from the indie VFX user's point of view.

### Now (0–6 weeks): make it trustworthy

| # | Item | Why | Effort |
|---|---|---|---|
| 1 | Commit the sources, `.gitignore`, CI | Section 3.1 | 1 day |
| 2 | Device test pass + diagnostics HUD (per-stage GPU ms, dropped frames, thermal state, latency) | Section 3.2, 3.5 | 1 week |
| 3 | Lock AE/AWB/AF after sampling; report evenness in stops | Section 3.4 | 2–3 days |
| 4 | One-tap **Learn Screen** (section 8.1) | Turns four separate actions into one | 1 week |
| 5 | Landscape rig layout + rig lock + hardware-button shortcuts | Section 8.2 | 1–2 weeks |
| 6 | Clean output on a real external display (not mirroring) | Section 8.4 | 3 days |
| 7 | Real demo clip, and CC0 HDRIs from Poly Haven [18] for the 360° mode | Section 3.7; 360° is useless without good panoramas | 2 days |
| 8 | Consolidate the docs | Section 3.3 | 1 day |

### Next (1–3 months): make it useful on a real shoot

| # | Item | Why | Effort |
|---|---|---|---|
| 9 | `FrameSource` refactor + **iPad UVC input** | Section 5; this is the original problem | 3–4 weeks |
| 10 | **Keyability QC overlays + Key Score + take report** (section 7.1) | The feature nobody else does well | 3 weeks |
| 11 | **Screen-reference keying** (IBK-style, section 7.2) | Big quality gain on unevenly lit indie screens; also gives physically correct edge colour | 2 weeks |
| 12 | **Take recording** (section 7.4) | Moves the app from a preview toy to a production tool; editors get temp comps on day one | 3–4 weeks |
| 13 | Tap-to-keep object with EdgeTAM (Apache-2.0, ~16 FPS on iPhone 15 Pro Max) [19] | Props in Smart mode on non-LiDAR phones, chosen by the user instead of guessed | 2–3 weeks |

### Later (3–9 months): differentiate

| # | Item | Notes |
|---|---|---|
| 14 | iPhone as A-camera (`AVCaptureSource`) | Record Apple Log / ProRes clean plate + key preview; add a `.cube` LUT for viewing Log; timecode from ProDock on iPhone 17 Pro |
| 15 | Wireless monitor | Composite to the director's iPad over the local network using VideoToolbox low-latency encoding; NDI out for video village / OBS / vMix |
| 16 | Video plates and Gaussian splats | Looping video plates for car interiors and windows (process-plate work). Splat scenes from Scaniverse/Polycam rendered with MetalSplatter (MIT; PLY/SPZ/.splat) [20] |
| 17 | Shadow catcher + light match | Use the LiDAR floor plane to transfer the performer's shadow to the plate; match colour with simple statistics transfer (see the licence notes in 7.5) |
| 18 | Mac companion ("Screen Check Desk") | Ingest takes, align to camera originals, run the user's own keyer (e.g. CorridorKey), export EXR + Nuke/Resolve/AE/Blender setups. Natural place for a paid tier |
| 19 | Interop exports | OpenTrackIO (SMPTE RIS-OSVP, v1 January 2025) [21], USD camera, `.chan`, FBX |
| 20 | Cinema-camera FOV simulation | Crop the iPhone image to match the planner's sensor and focal length, with frame lines. Combined with the live key, this becomes a keyed director's viewfinder for location scouts |

---

## 7. Feature details

### 7.1 Keyability QC: "will it key?"

The evenness measurement is a good start, but it only runs once, on a bare screen. Turn it into live overlays that run while the performer is in frame, plus a score:

| Check | How to compute it cheaply | Fix shown to the user |
|---|---|---|
| Screen evenness (false colour) | Deviation of screen pixels from the screen reference (7.2), in ⅓-stop bands | "Left third is 0.7 stop under: raise the left screen light or move it back" |
| Screen purity / saturation | Chroma distance of the screen from neutral, relative to its luminance | "Screen is desaturated: it is over-lit or the fabric is washed out" |
| Spill on talent | Despill amount inside the protected (person/depth) mask, as a heatmap | "Spill on shoulders: move 0.5 m forward". Use the planner's separation model for the distance |
| Noise in the screen | Local standard deviation of chroma over flat screen areas | "Screen is noisy: lower ISO or add light" |
| Edge softness | Width of the alpha transition band (motion blur / defocus) | "Fast motion at 1/48 will give soft edges: consider a 1/96 shutter" |
| Wardrobe / prop conflict | Share of foreground pixels within the key hue tolerance | "Green in the wardrobe" |
| Shadows on the screen | Luminance dips against the screen reference behind the subject | "Performer shadow on screen: separate or flag" |
| Coverage | Where alpha never reaches 0 at the frame edge | "Screen runs out at top right: garbage-matte or reframe" |

Each check feeds a green/amber/red Key Score with the reason in words, never colour alone. **Export Take Report** writes a PDF + JSON with the score, the overlay snapshots, the locked camera settings and the plan. A VFX artist would actually want to receive that.

### 7.2 Screen-reference keying (IBK-style)

Nuke's IBK keyer, invented by Paul Lambert, the VFX supervisor on *Dune*, keys each pixel against a per-pixel **screen colour plate** instead of one picked colour [22]. That plate is either a shot clean plate or one generated from the frame (`IBKColour`). This is the right model for indie screens, which are rarely lit evenly. It fits the existing engine well:

1. **Build the screen reference S(x)** at ~¼ working resolution. Take pixels that are confidently screen (strong key-channel dominance, outside the person/depth protection mask). Fill the holes behind the performer by **push-pull** interpolation: build a mip chain of weighted colour (premultiplied by confidence), then fill holes from coarser levels on the way up (Gortler et al., *The Lumigraph*, SIGGRAPH 1996). It is a few small GPU passes.
2. **Stabilise S.** With a locked-off camera, accumulate S over time. With ARKit plane lock (already built in 2.0), store S in the screen plane's UV space and reproject it every frame, so the reference survives camera moves.
3. **Key against S**, IBK-style:
   `α = 1 − clamp( D(C) / D(S) )`, with `D(x) = x_k − (w_a·x_a + w_b·x_b)`
   where *k* is the key channel and *a*, *b* the other two. This is IBK's weighted form; the max-based, exposure-normalised dominance the engine already uses works here too.
4. **Unmix instead of channel-limiting despill.** Since the camera pixel is C = αF + (1−α)S, the premultiplied foreground is **αF = C − (1−α)S**, and the composite over background B becomes:
   `out = C + (1 − α)(B − S)`
   This follows directly from the compositing equation that Smith & Blinn set out for screen matting (*Blue Screen Matting*, SIGGRAPH 1996). Hair and motion-blurred edges come out with the correct colour instead of a grey fringe, and the whole thing is one pass in `scComposite`.

The same S plate drives the evenness false colour in 7.1, and it goes into the take package as `screen_reference.exr` for post.

### 7.3 Learn Screen and camera locking

See section 8.1. The engine side: lock AE/AWB/AF through the configurable capture device (ARKit) or `AVCaptureDevice` (AVFoundation), then build S, fit the LiDAR plane and run the evenness measurement, all from the same 1–2 seconds of bare-screen frames.

### 7.4 Take recording package

```
S012_T03.screencheck/
├── plate.mov             clean source (HEVC 10-bit; Apple Log/ProRes in A-camera mode)
├── comp.mov              composite proxy for editorial (HEVC, optional burnt-in TC)
├── alpha.mov             alpha hint (HEVC with alpha [23] or 8-bit grey)
├── screen_reference.exr  the S plate from 7.2
├── camera.usda           ARKit camera path; also .chan / FBX / OpenTrackIO JSONL
├── depth/                LiDAR depth + confidence (float16, sensor resolution)
├── report.json / .pdf    Key Score, QC snapshots, locked camera settings, plan
└── manifest.json         timecode, fps, colour spaces, device, app version
```

Notes:

- Write with `AVAssetWriter` and set `movieFragmentInterval` so a crash or a dead battery doesn't lose the take.
- Tag colour spaces honestly. Export `Input`/`AlphaHint` for CorridorKey as 16-bit PNG or EXR rather than 8-bit PNG to avoid banding in screen gradients. CorridorKey accepts sRGB or linear input [2].
- Sync to camera originals: timecode where available (Tentacle Sync, which Jetset already uses [1]; ProDock on iPhone 17 Pro). Otherwise the Mac companion can align takes by matching image content between the proxy and the original.
- Keep raw LiDAR depth and confidence. Desktop methods like Prompt Depth Anything (CVPR 2025) use phone LiDAR as a prompt to produce accurate 4K metric depth [24], which is useful for relighting and depth-of-field in post.

### 7.5 On-device ML: what you can ship legally

The project's principle is "no third-party dependencies". Apple's built-in Vision models should stay the first choice. If you add models, the licence decides:

| Model | Use | Licence | Ship in a paid App Store app? |
|---|---|---|---|
| EdgeTAM [19] | Tap-to-keep object tracking | Apache-2.0 | Yes |
| MODNet [25] | Trimap-free portrait matting | Apache-2.0 | Yes |
| BackgroundMattingV2 [26] | Matting with a captured clean plate | MIT | Yes (built for desktop GPUs; needs a mobile port) |
| Depth Anything V2 **Small** [27] | Monocular depth on non-LiDAR phones; Apple ships a Core ML build at ~34 ms on iPhone 15 Pro Max [28] | Apache-2.0 | Yes |
| Depth Anything V2 Base/Large | — | CC-BY-NC-4.0 | **No** |
| Robust Video Matting [29] | Temporally stable human matting, Core ML builds available | **GPL-3.0** | Not in a closed-source app |
| Harmonizer [30] | Real-time colour harmonisation | **CC BY-NC-SA 4.0** | **No** (use a Reinhard-style statistics transfer instead) |
| MatAnyone [31] | Highest-quality video matting | Research | Desktop/Mac companion only; too heavy for live |
| CorridorKey [2] | Final neural key | Custom (no resale) | **No**. Keep the file-based bridge |

---

## 8. User experience

### 8.1 One action for the bare-screen moment

Today, Sample, Auto Key, Measure Screen Evenness and Calibrate Screen Depth are four separate controls. All four need the same thing: bare screen in frame, before the performer steps in. Make it one button, **Learn Screen**, with a 2-second progress ring, followed by a result card:

> **Screen: good** · 0.4 stop spread · green learned · LiDAR plane at 3.2 m, 12° tilt
> Exposure locked: 1/48 · ISO 200 · 5600 K
> *Bring in talent →*

Keep the individual tools under an Advanced disclosure for people who want them.

### 8.2 Built for a rig

- **Landscape-first layout** on iPhone and iPad, with controls along the short edges and the image as large as possible.
- **Rig lock:** ignore touches until a long-press, so a bumped phone on a cage doesn't change the key.
- **Hardware buttons:** volume buttons as record/still via `AVCaptureEventInteraction` (iOS 17.2+; test it with an active ARKit session). On iPhone 16/17, map the Camera Control to the view mode (Composite / Matte / Camera / QC) with the capture-controls API when running the AVFoundation source.
- **Stages instead of one control panel:** Setup → Rehearse → Shoot → Review, each showing only its own controls.

### 8.3 Faster judgement

- **Split wipe** between composite and original. Tap-and-hold to peek at the original.
- **Plain-language alerts with fixes** from the QC checks (7.1), using the planner's models for the numbers. Add an optional haptic tick when a check goes red.
- **Remember the last background.** Today a Photos background has to be re-picked after every relaunch, which is annoying on set. Store a downsampled copy in the app container; the user picked it, so it's a reasonable default with a one-line note.

### 8.4 Real clean output

iPhone 15 and later output video over USB-C to an HDMI or DisplayPort monitor. Instead of mirroring, give the external display its own scene (`UISceneSession.Role.windowExternalDisplayNonInteractive`). It then shows the full-screen composite while the phone keeps showing controls and QC. That is what a client monitor should do.

---

## 9. Reliability and testing

1. **Replay tests with real footage.** With `FileSource`, run recorded clips through the full engine in CI. Compare mattes against ground truth using the standard matting metrics (SAD, MSE, gradient and connectivity errors, as used on alphamatting.com) and fail the build on regressions. For ARKit-specific behaviour, record sessions with Reality Composer and replay them through Xcode's ARKit replay option [32].
2. **Build a footage corpus.** Ten short clips from your own shoots: uneven screen, hair, motion blur, green wardrobe, a reflective prop, a dark room, a screen that is too small. This corpus is worth more than any new feature.
3. **Soak tests.** 15, 30 and 60 minutes in each quality mode on a LiDAR and a non-LiDAR phone. Record battery %/hour, thermal state over time, memory high-water mark and sustained FPS. Publish the results in the docs.
4. **Field telemetry, opt-in.** MetricKit gives crash, hang, CPU/GPU and thermal diagnostics with no images collected, which fits the privacy principles.
5. **Watchdog.** If no frame has rendered for 500 ms, switch to camera passthrough with a banner. Never show a frozen frame that looks live.
6. **Per-stage budgets.** Add `os_signpost` intervals around each stage and check them in Instruments' Metal System Trace. Write the budget into the docs (for example, at 30 fps: capture → composite ≤ 12 ms GPU).

## 10. Speed

- **Default to the project frame rate** (24/25/30) and spend the headroom on latency and thermals. Offer 60 fps only on request. A 60 fps preview of a 24 fps shot misrepresents the motion blur the camera will actually record.
- **Check that YCbCr → RGB happens inside the fused kernel**, reading ARKit's planes through `CVMetalTextureCache` with no copies. If Core Image inserts colour-matching passes, set the context's working and output colour spaces explicitly.
- **Present with `CAMetalDisplayLink`** (iOS 17+) to get the target presentation time and cut latency. Set the ProMotion frame-rate range to the content rate.
- **Run neural models at a low rate and propagate between runs** using the ARKit pose or a cheap motion estimate. 2.0 already does this for AI object assist; extend it to EdgeTAM.
- **If Instruments shows Core Image overhead on the hot path,** move the fused kernel to a native Metal compute pipeline. The kernel code mostly carries over, and you get direct control over scheduling and memory.

## 11. Business viability

**Positioning:** "Plan the screen. Check the key on your real camera. Hand post everything it needs." No 3D scene required.

**Pricing (suggestion, to be tested):** indie filmmakers dislike subscriptions, and the comparable one-off apps sit at $10–30.

- **Free:** planner, live key on the built-in camera, Learn Screen, basic QC. This is how people find you.
- **Pro (one-time ~$29–49, or ~$4.99/month):** external input, take recording and packages, take reports, 360°/video/splat plates, external clean output.
- **Desk (Mac, later):** ingest, alignment, keyer bridge, EXR and script export. Studios and post houses pay here.
- **Education pricing** for film schools. A planner plus a live key is a strong teaching tool.

**Distribution:** filmmaking YouTube, r/Filmmakers, r/vfx, the CorridorKey community (a CorridorKey-ready take package is a natural hook), film schools, and gear partners. Accsoon already works with Jetset, and Elgato and other capture-card makers want iPad use cases.

**Validate before building the expensive parts.** Interview 10–15 indie filmmakers and VFX artists, then join three real shoots with the Now and Next items. Measure:

- minutes from "screen up" to "good key";
- problems caught on set that would otherwise have reached post;
- keying hours saved in post, compared with a similar earlier project.

If those numbers move, the product is viable. If they don't, the shoots will show why.

## 12. What not to build

- **Full virtual production with 3D scenes.** Jetset does it with a team and a head start. Export OpenTrackIO / USD and work alongside it instead.
- **AI-generated backgrounds.** That's Skyglass's space. VFX artists bring their own plates.
- **A bundled final keyer.** CorridorKey's licence forbids it, and the live key's job is to judge, not to finish.
- **More sliders.** Every new control should replace an old one or live in Advanced.
- **Non-commercial or GPL models** in the App Store build (see 7.5).

---

## References

Competitors and hardware

1. Lightcraft Jetset: pricing and tiers, [Camera Jabber](https://camerajabber.com/?p=1412075); feature update, [Broadcast Beat](https://www.broadcastbeat.com/lightcraft-jetset-expands-iphone-virtual-production-tool-with-a-dozen-new-features); SeeMo + Jetset Cine, [Newsshooter](https://www.newsshooter.com/2024/02/15/accsoon-seemo-lightcraft-jetset-cine-app-allow-you-to-use-your-iphone-15-pro-max-as-a-camera-tracker-real-time-previsualization-device/); overview, [ProVideo Coalition](https://www.provideocoalition.com/jetset/)
2. CorridorKey, [GitHub: nikopueringer/CorridorKey](https://github.com/nikopueringer/CorridorKey); [No Film School](https://nofilmschool.com/corridorkey-ai-chroma-keyer)
3. ARKit camera configuration: [Apple Developer Forums thread 766844](https://developer.apple.com/forums/thread/766844); [Unity ARKit camera docs](https://docs.unity3d.com/Packages/com.unity.xr.arkit@6.1/manual/arkit-camera.html)
4. Skyglass: [DIY Photography](https://www.diyphotography.net/?p=221017); [Streaming Media review](https://www.streamingmedia.com/Producer/Articles/Editorial/Featured-Articles/Review-Skyglass-Movie-Effects-Camera-App-159155.aspx) (pricing changes often; check before quoting)
5. CamTrackAR: [No Film School](https://nofilmschool.com/fxhomes-free-virtual-production-app-now-chroma-key-filters); [ProVideo Coalition](https://www.provideocoalition.com/camtrackar-v2-0-ios-virtual-production-app-gets-new-features/)
6. Green Screener: [App Store](https://apps.apple.com/app/id604935529)
7. Cine Meter: [adamwilt.com](https://www.adamwilt.com/cinemeter)
8. Artemis Pro: [App Store](https://apple.co/4bmYKQM); Cadrage: [App Store](https://apple.co/41kqnWe)
9. Accsoon SeeMo 4K: [Newsshooter](https://www.newsshooter.com/2024/03/25/accsoon-seemo-4k-hdmi-adapter-for-ios/)
10. *28 Years Later* on iPhone 15 Pro Max: [No Film School](https://nofilmschool.com/28-years-later-iphone)
11. Blackmagic Camera ProDock and iPhone 17 Pro: [No Film School](https://nofilmschool.com/blackmagic-camera-prodock)

Apple platform

12. UVC on USB-C iPad: [Switcher Studio support](https://support.switcherstudio.com/article/613-uvc-cameras); WWDC23 "Support external cameras in your iPadOS app"
13. Cheap HDMI dongle latency report: [Cinelerra-GG mailing list](https://lists.cinelerra-gg.org/pipermail/cin/2023-February/006297.html)
14. LiDAR depth through AVFoundation: [Capturing depth using the LiDAR camera](https://developer.apple.com/documentation/avfoundation/capturing-depth-using-the-lidar-camera); [WWDC22 session 110429](https://developer.apple.com/videos/play/wwdc2022/110429/)
15. No UVC on iPhone: [Apple Developer Forums thread 775883](https://developer.apple.com/forums/thread/775883)
16. NDI on iOS: [NDI SDK platform considerations](https://docs.ndi.video/all/developing-with-ndi/sdk/platform-considerations)
17. HaishinKit (RTMP/SRT): [GitHub](https://github.com/HaishinKit/HaishinKit.swift)
18. Poly Haven CC0 HDRIs: [polyhaven.com](https://polyhaven.com/hdris)
23. HEVC with alpha: [WWDC19 session 506](https://developer.apple.com/videos/play/wwdc2019/506/)
32. ARKit session record and replay: [Recording and replaying AR session data](https://developer.apple.com/documentation/arkit/recording-and-replaying-ar-session-data)

Research and open source

19. EdgeTAM (CVPR 2025): [GitHub: facebookresearch/EdgeTAM](https://github.com/facebookresearch/EdgeTAM)
20. MetalSplatter: [GitHub: scier/MetalSplatter](https://github.com/scier/MetalSplatter); Kerbl et al., *3D Gaussian Splatting for Real-Time Radiance Field Rendering*, SIGGRAPH 2023
21. OpenTrackIO: [SMPTE RIS-OSVP documentation](https://ris-pub.smpte.org/ris-osvp-metadata-camdkit/)
22. IBK: [Foundry IBKGizmo reference](https://learn.foundry.com/nuke/content/reference_guide/keyer_nodes/ibkgizmo.html); [befores & afters on Paul Lambert](https://beforesandafters.com/2022/03/25/how-dune-vfx-supervisor-paul-lambert-invented-nukes-ibk-keyer/). Also Smith & Blinn, *Blue Screen Matting*, SIGGRAPH 1996; Gortler et al., *The Lumigraph*, SIGGRAPH 1996 (push-pull)
24. Prompt Depth Anything (CVPR 2025): [arXiv 2412.14015](https://arxiv.org/abs/2412.14015)
25. MODNet: [GitHub: ZHKKKe/MODNet](https://github.com/ZHKKKe/MODNet)
26. BackgroundMattingV2 (CVPR 2021): [GitHub: PeterL1n/BackgroundMattingV2](https://github.com/PeterL1n/BackgroundMattingV2)
27. Depth Anything V2: [GitHub](https://github.com/DepthAnything/Depth-Anything-V2)
28. Apple Core ML Depth Anything V2 Small: [Hugging Face](https://huggingface.co/apple/coreml-depth-anything-v2-small)
29. Robust Video Matting (WACV 2022): [GitHub: PeterL1n/RobustVideoMatting](https://github.com/PeterL1n/RobustVideoMatting)
30. Harmonizer (ECCV 2022): [GitHub: ZHKKKe/Harmonizer](https://github.com/ZHKKKe/Harmonizer)
31. MatAnyone (CVPR 2025): [GitHub: pq-yang/MatAnyone](https://github.com/pq-yang/MatAnyone)
