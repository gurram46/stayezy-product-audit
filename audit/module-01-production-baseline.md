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

**Evidence:** Android `Choreographer` reported two startup frame-skip bursts:

- `Skipped 44 frames! The application may be doing too much work on its main thread.`
- `Skipped 64 frames! The application may be doing too much work on its main thread.`

At 60 Hz these represent roughly 0.7 s and 1.1 s of missed frame budget respectively. The exact 20–30 second user-perceived lag cannot be attributed solely to these two entries, but the log confirms real main-thread/rendering pressure during startup.

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

### Finding H-004 — Anonymous analytics initialization throws a plugin exception

During an anonymous launch, Mixpanel attempts identification with no user ID and logs:

- `Can't identify with null distinct_id.`
- `java.lang.IllegalStateException: Reply already submitted`
- `Uncaught exception in binary message listener`

The app continues running, so this is not currently a fatal crash, but it is a real initialization error in the anonymous path.

**Likely correction to verify in code:** do not call user `identify` with a null ID; keep the anonymous/device identity until login, then alias/identify deliberately.

**Current severity:** P2.

### Finding H-005 — Expensive/unnecessary anonymous startup work needs review

Observed during the anonymous Home launch:

- Home API call begins with an empty token.
- Another request (`addPropertyGetData`) takes ~528 ms and returns `{status: false, message: Unauthorized Token}`.
- Home API timing is ~934 ms / ~952 ms total.
- Map API is called immediately and returns 300 properties even though the user is on Home.
- A duplicate Home API request is detected and skipped (`REQUEST ALREADY PENDING`).
- GC frees ~25 MB of allocation-space data including ~10 MB of large-object-space objects during startup.

None of these alone proves the root cause of the visible lag. Together they show that the anonymous dashboard startup is doing substantial network, map, analytics and allocation work that should be profiled and reduced.

**Current severity:** P2 performance/efficiency investigation.

### Finding H-006 — Cache accounting looks potentially duplicated

The app logs approximately:

- temporary/cache directory: ~19.94 MB
- cache: ~19.94 MB
- support: ~130.8 KB
- computed `Before`: ~40.01 MB
- cleanup threshold: 300 MB

On Android, temporary/cache paths can resolve to the same underlying cache directory. The near-exact duplication suggests the cleanup accounting may be counting the same cache twice. Verify the actual directory paths in code before treating this as confirmed.

**Potential effect if confirmed:** the app could believe it has reached the 300 MB cleanup threshold when actual unique cache usage is closer to half that value.

**Current severity:** P3/P2 candidate.

### Finding H-007 — Production build logs runtime identifiers and precise location

The Play-installed production build logs detailed analytics payloads and runtime data, including device identifiers, analytics identifiers/tokens, an FCM registration token, precise latitude/longitude, API URLs and large API responses.

**Assessment:** release logging should be minimized/redacted. Do not commit the raw clean log to a public repository because it contains device/session-specific values and precise location data.

**Current severity:** P2 security/privacy hardening.

## Immediate codebase checks once repository access is available

1. Inspect the bottom-navigation/dashboard widget tree and verify whether the Map tab/`GoogleMap` is constructed and rendering while Home is active.
2. Profile startup work on the Flutter main isolate and identify what is executed before/around the `Choreographer` skipped-frame bursts.
3. Trace anonymous analytics initialization and remove the null Mixpanel `identify` path.
4. Identify the anonymous request returning `Unauthorized Token` and avoid making it when unauthenticated if it is not required.
5. Inspect why 300 map properties are fetched during Home startup and whether the Map module can be lazy-initialized only when the user opens Map.
6. Inspect image URL dimensions, response sizes, HTTP cache headers and Flutter image-provider/prefetch strategy for H-001.
7. Verify cache-directory accounting to ensure temporary/cache directories are not double-counted.
8. Disable/redact verbose release logs containing FCM token, location, device IDs and full API responses.

## Evidence to preserve

- Baseline app-info/storage screenshots.
- Home-page screenshots showing delayed image placeholders/listing rails.
- `Screen_recording_20260902_160212.mp4`.
- First broad logcat capture.
- Clean Stayezy-process logcat capture (`stayezy-home-clean-log.txt`) retained outside the public repo unless redacted.
- Idle Home Stayezy-process logcat capture (`stayezy-home-idle-log.txt`) retained outside the public repo unless redacted.
