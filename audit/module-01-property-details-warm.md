# Module 01B — Property Details Warm-Session Baseline

## Test state

- Device: Motorola Edge 40
- Android: 15
- App version: 1.1.0
- Account state: anonymous / no account created
- Session state: warm session after Home + Map browsing
- Test flow: Map → Home → open one property → scroll property details
- User-observed behavior: noticeable transition lag when opening the property, visible image delay, and small-but-noticeable scroll jank.

## Evidence source

Raw process log: `stayezy-property-details-warm-log.txt`.

Do **not** commit the raw log publicly. It contains runtime/device analytics identifiers and remote asset URLs.

## Finding PD-001 — Property-details jank is strongly correlated with rendering-buffer saturation

At the exact transition into `Listingdetials`, Android reports:

- `PipelineWatcher: pipelineFull: too many frames in pipeline (6)`
- repeated `BLASTBufferQueue ... Can't acquire next buffer. Already acquired max frames 8 max:6 + 2`

The warm property-details capture contains **166 BLAST buffer-acquisition failures**. They occur in several distinct bursts:

- 25 errors from about 16:46:24.755–16:46:24.967
- 39 errors from about 16:46:40.252–16:46:40.564
- 51 errors from about 16:46:41.433–16:46:42.208
- 14 errors from about 16:47:09.135–16:47:09.237
- 37 errors from about 16:47:12.002–16:47:12.290

This means the rendering-surface problem found on Home/Map continues into property details and is active during the same period in which the user sees transition/scroll lag.

**Classification:** confirmed rendering-pressure symptom; root cause not yet proven.

**Severity:** P1/P2 candidate.

## Finding PD-002 — Media/video teardown happens immediately during the property-details transition

Immediately after the `Listingdetials` screen-view event, the process performs significant media/surface cleanup:

- ExoPlayer is released.
- AVC video and AAC audio `MediaCodec` instances are flushed/restarted/released.
- codec surfaces are disconnected/reconnected.
- multiple codec buffers are discarded as unknown.
- `GraphicsTracker` reports multiple failed deallocations while the codec/surface is being stopped.

This is temporally aligned with the first visible property-open lag and the first BLAST-buffer burst.

**Working hypothesis only:** a video/media preview from the previous listing/home screen may still own a surface/decoder and is being torn down while the property-details screen is being constructed. Combined with the already-known off-screen Google Maps platform view, this may create unnecessary surface/GPU contention during navigation.

**Code check required:** trace lifecycle/disposal of all video players/ExoPlayer-backed widgets and native platform views during `DisplayFeatureSubScreen → Listingdetials` navigation.

## Finding PD-003 — Production code performs extremely noisy scroll-time logging

The property-details capture contains approximately:

- **621** `Scroll offset: ... Remaining: ...` log lines
- **72** `imagesfullscr[crindex]...<same image URL>` log lines

All 72 image log entries reference the **same image URL**.

This strongly suggests that code on the property-details screen is executing/logging on scroll/rebuild paths at a very high frequency. The repeated image URL log appears while the screen is being scrolled and may indicate that the image/carousel subtree is rebuilding repeatedly.

This alone does **not** prove that the image is downloaded 72 times. It does prove that the production build is repeatedly executing the associated debug/logging path.

**Why it matters:** synchronous/very frequent `print`/logging on Flutter UI-scroll paths adds avoidable main-isolate/logcat overhead and makes profiling harder. If the same rebuild also recreates image/video widgets, the performance cost could be much larger than the logging itself.

**Severity:** P2 performance/code-quality issue.

**Code check required:** find the `Scroll offset:` and `imagesfullscr[crindex]` logging statements. Verify whether they are inside a scroll listener/build method and whether the surrounding widget subtree is being rebuilt unnecessarily.

## Finding PD-004 — No property-details backend request appears in this capture

The warm property-details log contains no clear property-details GET/POST request or response timing. The navigation event appears first, followed by rendering/media work.

**Interpretation:** for this warm-session path, the property-details screen is likely receiving enough listing data from already-loaded state/navigation arguments to construct the screen without waiting for a new details API call, or the network layer is not instrumented in this portion of the capture.

Therefore the observed property-open lag cannot currently be blamed on a slow backend call. The strongest evidence in this run points to local rendering/media/widget work.

**Important:** do not tell the team this proves the backend is innocent in general. It only means this particular warm property-details capture does not show a details API bottleneck.

## Finding PD-005 — Image-delay cause is not yet proven by logcat

The user visibly observed delayed property images. The log repeatedly prints a Cloudflare R2 asset URL, but logcat does not expose useful HTTP timing/cache-hit/decode timing for that image in this capture.

So the image UX issue is real from visual observation, but the cause still needs one of:

- network inspection / image request timing;
- response-size and dimensions check;
- HTTP cache-header inspection;
- Flutter image-cache instrumentation;
- DevTools/performance trace;
- code inspection of image provider/widget rebuild behavior.

Do not assume R2/CDN is slow without measuring it.

## Immediate codebase checks once access is available

1. Find the `Listingdetials` widget and inspect its first build/init path.
2. Locate every `Scroll offset:` print/log statement and remove/gate release logging after profiling.
3. Locate `imagesfullscr[crindex]` logging and determine why the same image path is evaluated/logged repeatedly during scroll.
4. Inspect whether the image carousel/hero is rebuilt on every parent scroll tick.
5. Inspect ExoPlayer/video-player lifecycle on the previous screen; confirm controllers and surfaces are disposed before/after navigation at the correct time.
6. Check whether the off-screen Map platform view remains mounted while property details are open.
7. Profile the transition with Flutter DevTools frame chart/raster thread once source/debug build access is available.
8. Measure property image payload dimensions/bytes and cache headers before selecting an image-loading fix.

## Current assessment

The warm property-details lag looks more like a **client-side rendering/surface/widget-lifecycle problem** than a demonstrated server-side latency problem in this specific run. The highest-value suspects are:

1. persistent/native surface contention already seen on Home/Map;
2. media-player/codec teardown during navigation;
3. excessively noisy scroll/rebuild paths;
4. image widget/cache behavior.

No fatal crash, ANR, OOM, or explicit Flutter exception was observed in this capture.
