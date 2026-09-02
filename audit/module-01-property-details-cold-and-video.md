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

This changes the earlier interpretation: the severe continuous lag seen under Android Studio mirroring is not confirmed as an app bug, but **startup/initial-playback jank is physically reproducible** and therefore belongs to the Stayezy performance investigation.

**Current severity:** P2 performance/UX investigation.

### Measurement note

A later `dumpsys gfxinfo com.cw.stayezy reset` command printed a pre-reset snapshot of 529 frames / 52 janky frames (9.83%), 30 missed vsyncs and 40 slow-UI-thread events. Because `reset` prints the accumulated counters *before* clearing them, those numbers must not be attributed specifically to the new physical-video run without a post-run `gfxinfo` capture.

## Finding PD-009 — Video loop/restart appears to recreate ExoPlayer instead of reusing the player

The dedicated physical-device capture gives a much stronger explanation for the visible video-start/restart hitch.

Across roughly 31 seconds of video activity, the log shows:

- **4 ExoPlayer initializations**;
- **3 ExoPlayer releases**;
- a repeating release → new init → first decoded frame sequence;
- **17** `PipelineWatcher ... pipelineFull: too many frames in pipeline (6)` events;
- **36** `MediaCodec discarded an unknown buffer` messages during codec teardown;
- **33** `GraphicsTracker: cannot deallocate due to being stopped` messages during player/surface cleanup.

Representative lifecycle timings:

- first player: Init `18:04:02.459` → first decoded output `18:04:03.606` ≈ **1.15 s**;
- second player: Init `18:04:11.526` → first output `18:04:12.462` ≈ **0.94 s**;
- third player: Init `18:04:20.349` → first output `18:04:21.412` ≈ **1.06 s**;
- fourth player: Init `18:04:29.302` → first output `18:04:30.179` ≈ **0.88 s**.

The preceding player reaches EOS and is released before the next player is initialized. This strongly suggests the short property video is being looped/restarted by tearing down and recreating the ExoPlayer/controller rather than keeping one player alive and seeking/repeating within the same instance.

That lifecycle is a strong candidate for the visible hitch the user reports when playback begins/restarts. It also explains why sustained playback feels substantially smoother after startup: once the decoder is established, the log shows output around **29–32 frames/s** for a 30-fps 720×1280 AVC stream.

There is no explicit ExoPlayer `STATE_BUFFERING`/buffering event in this capture, so the evidence does **not** currently support blaming the network/CDN as the primary cause. Network behavior should still be measured separately before ruling it out completely.

### Post-run `gfxinfo`

The isolated post-run snapshot reports:

- 44 UI frames rendered;
- 5 janky frames (**11.36%**);
- 3 missed-vsync events;
- 2 slow-UI-thread events;
- 5 frame-deadline misses;
- 95th percentile UI frame: 24 ms;
- 99th percentile UI frame: 73 ms;
- 10 active surfaces and 8 BufferQueues;
- ~272.6 MB estimated `GraphicBufferAllocator` allocation at capture time.

The 44-frame sample is small and Android `gfxinfo` measures the ViewRoot/UI path, not every frame rendered by the video surface. Therefore the percentage should be treated as evidence of **brief UI jank around the interaction**, not as the video playback frame rate.

### Code-level verification / smallest likely correction

When source access is available:

1. trace the Flutter video widget/controller lifecycle and confirm what causes a new ExoPlayer instance at each EOS;
2. if the clip is intended to loop, configure the existing player for repeat/seek-to-start rather than dispose/recreate;
3. keep controller/player ownership stable across widget rebuilds unless the media source actually changes;
4. ensure cleanup happens once when the viewer is dismissed, not at every loop boundary;
5. profile again after the lifecycle fix to verify the pipeline-full and teardown noise falls materially.

**Current severity:** P1/P2 performance candidate because it is repeatedly user-visible and produces avoidable codec/surface churn.

## Evidence

Committed visual evidence:

- `evidence/module-01-property-details/001-dreamland-property-details.jpg`
- `evidence/module-01-property-details/002-video-overlay-no-close-control.jpg`

Committed raw cold-run evidence (gzip-compressed, byte-preserving copies of the uploaded UTF-16 text files):

- `evidence/module-01-property-details/stayezy-property-details-cold-gfx.txt.gz`
- `evidence/module-01-property-details/stayezy-property-details-cold-log.txt.gz`

Committed filtered physical-video evidence:

- `evidence/module-01-property-details/video-physical-key-log-evidence.txt`

The original raw physical-video files were also supplied during the audit. The filtered evidence file preserves the key timestamps, counts and interpretation needed to reproduce the lifecycle finding without relying on a giant raw-log diff.
