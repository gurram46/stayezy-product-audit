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

### Visual reproduction

On the Dreamland farmhouse details screen, tapping the pink `Videos` control opens a large video overlay on top of the property-details page.

The resulting video viewer shows the video, progress bar and mute/audio control, but no dedicated close (`X`) or back control inside the video overlay. The white back arrow visible near the top-left belongs to the underlying property-details screen and is visually behind/outside the video overlay rather than part of the video viewer.

The user was unable to exit the video through an obvious in-app control and ultimately used the Android system back/navigation control on the physical phone.

This is now a **confirmed UX/navigation defect**, not merely a hypothesis from logcat.

### Failure scenario

A user opens a property video, especially on gesture-navigation devices or in a mirrored/remote-control environment, and cannot discover how to dismiss the media viewer. The only reliable escape path observed was Android system navigation.

### Smallest safe correction

Provide an explicit close/back affordance inside the video viewer overlay itself, with appropriate safe-area placement and semantics/accessibility. System back should remain supported as a secondary path.

**Current severity:** P1/P2 UX candidate because it can effectively trap the user in a media state.

## Finding PD-008 — Apparent video playback lag requires physical-device confirmation

Video playback looked severely laggy while the phone was being viewed/controlled through Android Studio device mirroring. That environment can itself degrade or distort video playback/rendering, so this cannot yet be classified as a Stayezy playback-performance bug.

Required confirmation test:

1. play the same property video directly on the physical Motorola Edge 40 with Android Studio mirroring disconnected;
2. observe frame continuity, audio/video sync and responsiveness;
3. only if lag persists, capture a dedicated logcat/perf trace and inspect ExoPlayer/MediaCodec, network throughput, source bitrate/resolution and surface lifecycle.

**Status:** unconfirmed pending physical-device-only reproduction.

## Evidence

Committed visual evidence:

- `evidence/module-01-property-details/001-dreamland-property-details.jpg`
- `evidence/module-01-property-details/002-video-overlay-no-close-control.jpg`

Committed raw cold-run evidence (gzip-compressed, byte-preserving copies of the uploaded UTF-16 text files):

- `evidence/module-01-property-details/stayezy-property-details-cold-gfx.txt.gz`
- `evidence/module-01-property-details/stayezy-property-details-cold-log.txt.gz`

The raw cold log is unusually short and does not contain enough transition detail to attribute the residual lag. The `gfxinfo` snapshot is the stronger quantitative evidence for this run.
