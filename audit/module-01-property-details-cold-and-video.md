# Module 01C — Cold Property Details + Video Viewer

## Test state

- Device: Motorola Edge 40
- Android: 15
- App version: 1.1.0
- Account state: anonymous / no account created
- Flow: cold app start → Home → same property details → scroll → open property video
- User-observed result: cold property open was noticeably better than the prior warm Map→Property route, but visible transition/scroll lag remained.

## Finding PD-006 — Cold property-details run still has measurable jank

Android `gfxinfo` captured after this run reports:

- 571 total frames rendered
- 38 janky frames (6.65%) using the current metric
- 88 janky frames (15.41%) using the legacy metric
- 13 missed vsync events
- 71 high-input-latency events
- 22 slow UI-thread events
- 26 slow draw-command events
- 38 frame-deadline misses
- 95th percentile frame time: 13 ms
- 99th percentile frame time: 31 ms

The user independently reported visible residual lag. This confirms the cold flow is improved relative to the warm Map→Property path but is not fully smooth.

`gfxinfo` also shows multiple active rendering surfaces/buffers and an estimated ~268 MB allocated by `GraphicBufferAllocator` at capture time. Treat the aggregate allocator value as diagnostic context rather than proof of a leak; code-level lifecycle inspection is still required.

**Current severity:** P2 performance investigation.

## Finding PD-007 — Property video viewer has no dedicated visible close/back control

### Visual and physical-device reproduction

On the Dreamland farmhouse details screen, tapping the pink `Videos` control opens a large video overlay on top of the property-details page.

The resulting video viewer shows the video, progress bar and mute/audio control, but no dedicated close (`X`) or back control inside the video overlay. The white back arrow visible near the top-left belongs to the underlying property-details screen and is visually behind/outside the video overlay rather than part of the video viewer.

This was reproduced again while using the physical Motorola Edge 40 directly. The user still could not find an in-viewer back/close affordance and had to rely on Android system navigation to escape.

This is a **confirmed UX/navigation defect**.

### Failure scenario

A user opens a property video and cannot discover how to dismiss the media viewer. The only reliable escape path observed is Android system navigation. This is especially problematic for users unfamiliar with system gestures and for accessibility/remote-control scenarios.

### Smallest safe correction

Provide an explicit close/back affordance inside the video viewer overlay itself, with appropriate safe-area placement and semantics/accessibility. System back should remain supported as a secondary path.

**Current severity:** P1/P2 UX candidate because it can effectively trap the user in a media state.

## Finding PD-008 — Physical-device playback is mostly smooth after startup, but video startup/loading jank is reproducible

The same property video was tested directly on the physical Motorola Edge 40 rather than through Android Studio mirroring.

Observed directly on-device:

- the cold app/property path still showed noticeable loading/transition lag;
- opening/starting the video introduced a noticeable loading/startup delay and a small burst of lag as playback began;
- once playback had started and settled, the video itself was not continuously laggy;
- the missing in-viewer close/back control remained present on the physical device.

This changes the earlier interpretation: the severe continuous lag seen under Android Studio mirroring is not confirmed as an app bug, but **startup/buffering/initial-playback jank is physically reproducible** and therefore belongs to the Stayezy performance investigation.

Potential areas to verify later, without assuming a cause:

- time to first frame and buffering behavior;
- ExoPlayer/controller initialization lifecycle;
- whether video controllers/codecs are created too early or recreated unnecessarily;
- source bitrate/resolution versus device/network conditions;
- whether the already-observed Map/native-surface pressure remains active during the property/video flow;
- transition work on the Flutter main isolate.

**Current severity:** P2 performance/UX investigation.

### Measurement note

A later `dumpsys gfxinfo com.cw.stayezy reset` command printed a pre-reset snapshot of 529 frames / 52 janky frames (9.83%), 30 missed vsyncs and 40 slow-UI-thread events. Because `reset` prints the accumulated counters *before* clearing them, those numbers must not be attributed specifically to the new physical-video run without a post-run `gfxinfo` capture. A dedicated post-run capture is required for isolation.

## Evidence

Committed visual evidence:

- `evidence/module-01-property-details/001-dreamland-property-details.jpg`
- `evidence/module-01-property-details/002-video-overlay-no-close-control.jpg`

Committed raw cold-run evidence (gzip-compressed, byte-preserving copies of the uploaded UTF-16 text files):

- `evidence/module-01-property-details/stayezy-property-details-cold-gfx.txt.gz`
- `evidence/module-01-property-details/stayezy-property-details-cold-log.txt.gz`

The original raw cold log is unusually short and does not contain enough transition detail to attribute the residual lag. The original `gfxinfo` snapshot is the stronger quantitative evidence for that run.
