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

- Vertical scrolling is smooth on Motorola Edge 40 / Android 15.
- Listing metadata/cards appear quickly.
- No crash, freeze, or obvious interaction failure observed during the initial home-page pass.

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

## Evidence to preserve

Baseline app-info/storage screenshots and home-page screenshots were supplied during this audit and should be retained under `evidence/module-01-*` when binary evidence is committed.
