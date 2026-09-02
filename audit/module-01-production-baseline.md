# Module 01 — Production Baseline

## Test environment

- Device: Motorola Edge 40
- Android: 15
- Network: ACT 5G Wi‑Fi
- Date: 2026-09-02
- Baseline start: 15:50 IST
- App version: 1.1.0
- Package: `com.cw.stayezy`
- Install state: Fresh install
- Account state: No account created
- Permission granted: Location only
- Notifications: Off

## Fresh-install storage baseline

Measured shortly after first launch:

- App size: 80.73 MB
- User data: 2.12 MB
- Cache: 7.70 MB
- Total storage: 90.54 MB
- Data + cache generated after install/first launch: ~9.82 MB

### Assessment

Observation only, not yet classified as a defect. The first-launch cache may be normal image/map/API caching. Re-measure after browsing and after a cold restart to determine whether cache growth is reasonable.

## Home page — normal-use baseline

### Positive observations

- Listing metadata/cards generally appear quickly.
- No fatal crash or ANR was observed during the anonymous Home-page pass.
- Normal vertical scrolling can be smooth after the page settles.

### Finding H-001 — Listing images load later than listing metadata

**Observed:** On the home page, card/listing structure and text become available before property images. When scrolling through horizontal/vertical listing rails, some cards show a grey image placeholder and the image appears later.

**Video evidence:** `Screen_recording_20260902_160212.mp4` supplied during the audit. Approximate examples from the recording show grey placeholders persisting for roughly 2–2.5 seconds before some images render.

**Current classification:** Performance/UX observation; not yet a confirmed implementation bug.

**Potential causes to verify later:**

- lazy loading only after a card enters the viewport, with little/no prefetching;
- large original image payloads rather than appropriately sized thumbnails;
- slow image origin/CDN response;
- missing or ineffective HTTP/device image caching;
- image decoding/rendering cost;
- repeated cache misses.

Do not select a fix until network timings, image payload sizes, cache headers, and implementation are inspected.

## Clean cold-start reproduction — 16:18 IST

A second Home-page test was run with screen recording/screen sharing disabled. The app process was force-stopped, the recent task was manually cleared, and the app was opened again for a clean cold-start observation.

**User-observed behaviour:**

- noticeable startup/early-session lag for roughly 20–30 seconds;
- small but visible vertical-scroll jank during the early Home session;
- horizontal card scrolling still showed the image-loading delay seen in H-001.

Note: Android may leave a task snapshot in Recents after `am force-stop`; that behaviour itself is not a Stayezy defect.

### Finding H-002 — Confirmed startup/main-thread jank

**Evidence:** Android `Choreographer` reported startup frame-skip bursts across repeated cold-start tests, including:

- `Skipped 44 frames! The application may be doing too much work on its main thread.`
- `Skipped 64 frames! The application may be doing too much work on its main thread.`
- `Skipped 72 frames! The application may be doing too much work on its main thread.`
- `Skipped 70 frames! The application may be doing too much work on its main thread.` during the later Map-baseline cold start.

At 60 Hz these bursts are substantial enough to produce visible jank. The exact 20–30 second user-perceived lag cannot be attributed solely to these entries, but the logs confirm real main-thread/rendering pressure during startup.

**Current severity:** P2 performance/stability investigation.

### Finding H-003 — Rendering surface repeatedly exhausts available buffers without screen recording

The clean log still repeatedly reports:

`BLASTBufferQueue ... Can't acquire next buffer. Already acquired max frames 8 max:6 + 2`

This occurs many times during the Home-page session and persists well after initial startup. Because the same condition reproduced with screen recording disabled, it is no longer reasonable to attribute the earlier buffer errors only to screen capture.

**Important correlation to inspect in code:**

- The app initializes Google Maps immediately after navigating to `NewDashboardScreen`.
- A `GoogleMapController`/platform view is created while the user is on the Home tab.
- The app simultaneously calls the map endpoint and loads **300 properties**.
- Buffer-queue errors begin and continue around this same period.

**Working hypothesis only:** the Map tab may be mounted/kept alive while the Home tab is visible (for example through an `IndexedStack` or equivalent), leaving the Google Maps platform view rendering off-screen and competing for buffers/GPU/main-thread work. This must be verified from the Flutter navigation/widget tree before calling it the root cause.

**Current severity:** P1/P2 candidate because it is directly correlated with user-visible jank and repeats continuously.

#### Idle Home confirmation — 16:27 IST

A third run was captured with the user leaving the Home screen untouched after launch. This substantially strengthens H-003:

- startup reports `Skipped 72 frames! The application may be doing too much work on its main thread.`;
- `GoogleMapController.onCreate` and Flutter `PlatformViewsController` creation occur immediately after navigation to `NewDashboardScreen`, confirming that a Google Maps platform view is actually constructed even though the visible tab is Home;
- the Map API is called immediately and later reports `Map API properties loaded: 300`;
- the first `BLASTBufferQueue ... Can't acquire next buffer` burst begins around `16:27:12.439`;
- the same error continues in repeated bursts while the screen is idle and is still present around `16:28:43.562` — at least ~91 seconds after the first observed buffer exhaustion in this run.

**Interpretation:** this is no longer an interaction-only or screen-recording-only symptom. The rendering surface continues hitting buffer exhaustion while the user is not scrolling or touching the screen. Off-screen Map initialization is now confirmed; whether that platform view is the direct cause of the BLAST failures still requires code-level verification or an A/B build where the Map view is lazily mounted.

#### Map interaction baseline — 16:36 IST

The anonymous Map flow was tested after another clean restart. The user reported that the first Map visit had a noticeable loading period, zoom/pan was generally smooth once loaded, and a second visit felt faster with only a short loading period around the property-card/bottom-sheet content.

The captured log explains an important part of this behaviour:

- `NewDashboardScreen` becomes active at `16:36:31.979`.
- Google Maps SDK initialization is already occurring at about `16:36:32.010`.
- `GoogleMapController` is installed at `16:36:32.065`.
- The Map property API request starts at `16:36:32.162` — before the later screen-navigation event associated with the user's Map interaction.
- The Map API returns HTTP 200 at `16:36:33.342`, roughly **1.18 s** after the request.
- `Map API properties loaded: 300` appears at `16:36:34.367`, roughly **2.21 s** after the request began.
- The next instrumented screen transition occurs at `16:36:42.420`, more than eight seconds after the 300 Map properties had already been loaded.
- There is only **one** Map API GET in this capture. The faster second visit therefore appears to be reusing already-loaded state/data rather than performing a second full Map-property fetch.
- The same BLAST buffer-exhaustion condition is extremely noisy in this capture: **328** `Can't acquire next buffer` entries were counted between approximately `16:36:32.365` and `16:36:43.548`, spanning the background/preload period and the later visible interaction.

**Interpretation:** the Map itself can feel reasonably fast because a significant part of its work is being paid for while the user is still on Home. That is a valid UX trade-off only if intentional and measured. Given the confirmed Home startup jank and buffer pressure, the current preload strategy may be optimizing Map-open latency at the expense of the initial Home experience.

**Do not call this a Map performance bug yet.** The Map interaction itself was mostly smooth after loading. The architectural issue to verify is whether pre-constructing the Google Maps platform view and eagerly fetching 300 properties on Home is justified versus lazily mounting/fetching when the Map tab is first selected, or prefetching only lightweight data without mounting the native platform view.

### Finding H-004 — Anonymous analytics initialization throws plugin exceptions

During anonymous launches, Mixpanel attempts identification with no user ID and logs:

- `Can't identify with null distinct_id.`
- `java.lang.IllegalStateException: Reply already submitted`
- `Uncaught exception in binary message listener`

The later cold restart also shows Mixpanel `track()`/`flush()` calls hitting a null native `MixpanelAPI` instance around activity/engine teardown/recreation. The app continues running, so these are not currently fatal crashes, but they are real analytics lifecycle/anonymous-path errors.

**Likely correction to verify in code:** do not call user `identify` with a null ID; keep the anonymous/device identity until login, then alias/identify deliberately. Also verify that analytics calls cannot race plugin initialization/disposal during Flutter engine/activity lifecycle changes.

**Current severity:** P2.

### Finding H-005 — Expensive/unnecessary anonymous startup work needs review

Observed during the anonymous Home launch:

- Home API call begins with an empty token.
- Another request (`addPropertyGetData`) returns `{status: false, message: Unauthorized Token}` despite no account being created.
- Home API timing is roughly 0.9 s in repeated captures.
- Map API is called immediately and returns 300 properties even though the user is on Home.
- A duplicate Home API request is detected and skipped (`REQUEST ALREADY PENDING`).
- Earlier captures show substantial allocation/GC activity during startup.

None of these alone proves the root cause of the visible lag. Together they show that the anonymous dashboard startup is doing substantial network, map, analytics and allocation work that should be profiled and reduced.

**Current severity:** P2 performance/efficiency investigation.

### Finding H-006 — Cache accounting looks potentially duplicated

The app logs nearly identical values for its temporary/cache and cache directories. In the latest Map capture, both are reported as about **22.39 MB**, while the computed `Before` total is about **44.90 MB** plus a small support directory.

On Android, temporary/cache paths can resolve to the same underlying cache directory. The near-exact duplication suggests the cleanup accounting may be counting the same cache twice. Verify the actual directory paths in code before treating this as confirmed.

**Potential effect if confirmed:** the app could believe it has reached the 300 MB cleanup threshold when actual unique cache usage is closer to half that value.

**Current severity:** P3/P2 candidate.

### Finding H-007 — Production build logs runtime identifiers and precise location

The Play-installed production build logs detailed analytics payloads and runtime data, including device identifiers, analytics identifiers/tokens, an FCM registration token, precise latitude/longitude, API URLs and large API responses.

**Assessment:** release logging should be minimized/redacted. Do not commit raw logcat captures to a public repository because they contain device/session-specific values and precise location data.

**Current severity:** P2 security/privacy hardening.

## Immediate codebase checks once repository access is available

1. Inspect the bottom-navigation/dashboard widget tree and verify whether the Map tab/`GoogleMap` is constructed and rendering while Home is active.
2. Profile startup work on the Flutter main isolate and identify what is executed before/around the `Choreographer` skipped-frame bursts.
3. Trace anonymous analytics initialization and remove the null Mixpanel `identify` path; inspect plugin lifecycle around activity/engine recreation.
4. Identify the anonymous request returning `Unauthorized Token` and avoid making it when unauthenticated if it is not required.
5. Inspect why 300 map properties are fetched during Home startup. Compare current eager preload against lazy Map initialization or lightweight data-only prefetching.
6. Inspect image URL dimensions, response sizes, HTTP cache headers and Flutter image-provider/prefetch strategy for H-001.
7. Verify cache-directory accounting to ensure temporary/cache directories are not double-counted.
8. Disable/redact verbose release logs containing FCM token, location, device IDs and full API responses.

## Evidence to preserve

- Baseline app-info/storage screenshots.
- Home-page screenshots showing delayed image placeholders/listing rails.
- Map screenshot showing the anonymous Map result/property bottom sheet.
- `Screen_recording_20260902_160212.mp4`.
- First broad logcat capture.
- Clean Stayezy-process logcat capture (`stayezy-home-clean-log.txt`) retained outside the public repo unless redacted.
- Idle Home Stayezy-process logcat capture (`stayezy-home-idle-log.txt`) retained outside the public repo unless redacted.
- Map Stayezy-process logcat capture (`stayezy-map-log.txt`) retained outside the public repo unless redacted; it contains device/session/location identifiers and should not be committed raw.
