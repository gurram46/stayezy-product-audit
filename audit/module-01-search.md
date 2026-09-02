# Module 01 — Search Flow

## Scope

Anonymous production-app search test on 2026-09-02 using Motorola Edge 40 / Android 15. Search path tested from Home through location entry, dates, guests, results, property opens, back-navigation, and result scrolling.

## Positive observations

- Search is functional and returns relevant-looking result cards with distance/time, price, capacity, images and property metadata.
- On entering `SearchNew` while anonymous, the app logs `Token/UserId missing - API not called`, which is cleaner than the anonymous Home path that still triggers an unauthorized-token request.
- No fatal crash or ANR was observed in this search session.
- The filtered log does not show a large `Choreographer: Skipped N frames` burst during the search flow itself.

## Finding S-001 — Main search UI does not expose property type even though backend results are category-aware

The visible search UI exposes only:

- location;
- dates;
- guest count.

There is no obvious property-type/category input despite the product prominently organizing inventory on Home into Farmhouses, Service Apartments, Villas, Camping & Adventure, etc.

The backend response is already category-aware: the captured response reports `count: 2` category groups and explicitly returns `category_id` / `category_name` fields, including a `Farmhouses` group with 51 results. This means category is part of the search/result data model even though the primary search form does not currently let users express that intent directly.

**Assessment:** product/UX opportunity, not a correctness bug. Whether to add property type should be decided with behavioral data (search conversion, filter usage, result clicks, booking conversion), not assumption alone.

**Current severity:** P2 product/UX candidate.

## Finding S-002 — Search form has avoidable keyboard/focus friction and weak hierarchy

Observed directly in the production UI:

- location entry requires extra focus/keyboard management before comfortably progressing through the rest of the form;
- the form uses large disconnected cards with substantial empty vertical space;
- after search, the top summary field compresses a long address to a truncated single-line value while date/guest context is pushed into a second line;
- the search-summary control reads visually more like a cramped text field than a polished, scannable search state.

The interaction works, but the path has unnecessary friction for a high-intent conversion surface.

**Current severity:** P2/P3 UX.

## Finding S-003 — Search result loading window is roughly 2.6–3.6 seconds in this capture

The filtered log shows:

- loading/progress state begins at `19:03:48.733`;
- raw search response is logged at `19:03:51.332` — about **2.60 s** later;
- parsed/model state (`Instance of 'Getfilter'`) appears at `19:03:52.339` — about **3.61 s** after the progress state begins.

The response reports:

- `current_page: 1`;
- `per_page: 20`;
- `total: 52`;
- `total_pages: 3`;
- `count: 2` category groups.

The filtered capture does not expose a clean request-start timestamp for the network call itself, so 2.60 s must be treated as a **user-visible loading-window estimate**, not a precise server latency measurement.

**Current severity:** P2 performance candidate because search is a conversion-critical path.

## Finding S-004 — Client pagination state appears inconsistent with the API contract

Immediately after the API reports `total_pages: 3`, the client repeatedly logs:

`Total properties: 21 | lastPage: 999 | hasMoreData: true`

The same line is emitted many times during build/scroll activity.

No clear page-2/page-3 search request was found in this captured run.

**Working hypothesis only:** the search results screen may be using a sentinel/hard-coded `lastPage: 999` or stale pagination state instead of consuming the server's real `total_pages`. This could produce unnecessary pagination checks, continued `hasMoreData=true`, or incorrect end-of-list behavior. Source inspection and a dedicated end-of-results pagination test are required before classifying it as a functional bug.

**Current severity:** P2 candidate.

## Finding S-005 — Search/results screens generate excessive production debug output during typing and rendering

The release build logs every typed search-state mutation:

- `m`
- `ma`
- `mad`
- `madh`
- `madhp`
- `madhpu`
- `madhpur`

It also emits large API response payloads, repeated pagination-state lines, image URLs, and per-frame/per-scroll offset logs.

This extends the already-recorded production logging concern from H-007: user search terms can reveal location intent or other private behavior, while per-frame debug output can also add avoidable work/noise in a performance-sensitive UI.

**Current severity:** P2 privacy/engineering-hardening.

## Finding S-006 — Session-level gfxinfo confirms jank, but this capture is not Search-only

Post-run `gfxinfo` after the reset reports:

- 1,373 frames rendered;
- 151 janky frames (**11.00%**);
- 88 missed-vsync events;
- 110 high-input-latency events;
- 88 slow-UI-thread events;
- 57 slow draw-command events;
- 151 frame-deadline misses;
- 95th percentile frame: 19 ms;
- 99th percentile frame: 53 ms.

However, this measurement includes more than the search interaction: after results loaded, the session also opened property details multiple times and performed extensive scrolling. Therefore the 11.00% figure **must not be attributed solely to Search**.

The earlier `gfxinfo ... reset` command printed a pre-reset snapshot of 1,937 frames / 384 janky frames (19.82%). That output belongs to accumulated activity before reset and is not part of the isolated search measurement.

**Assessment:** jank is real at session level, but a shorter Search-only profile is needed if Search-specific rendering cost becomes a priority.

## Behavioral analytics recommendation

Search should be instrumented as a conversion funnel rather than judged only by UI preference. At minimum, record stable analytics events for:

- search opened;
- location query started / suggestion selected;
- dates selected;
- guests selected;
- search submitted;
- results first rendered;
- zero-results state;
- result clicked;
- filter/category applied;
- property detail opened from search;
- returned to results;
- booking started;
- checkout/payment reached;
- booking completed.

Track latency fields such as search-submit → first-result-render and result-click → property-usable. This matches the product goal of understanding where users drop between Home and payment and making changes based on measured behavior rather than intuition.

## Code-level checks once repository access is available

1. Inspect SearchNew focus/keyboard handling and whether location selection should automatically dismiss the keyboard / move focus.
2. Trace the search request and measure network time separately from decoding/model conversion/widget render time.
3. Inspect why API `total_pages: 3` becomes client `lastPage: 999`.
4. Verify infinite-scroll termination and whether page 2/3 are fetched exactly once when required.
5. Remove release-mode logging of raw keystrokes, full search responses, image URLs, and per-frame scroll offsets.
6. Check whether category/property-type filtering already exists in API parameters but is simply not exposed by the primary UI.
7. Add funnel analytics with latency and conversion events before making substantial search-layout changes.

## Evidence

- production screenshot of Search form with location/date/guest criteria;
- production screenshot of result cards and compressed search-summary header;
- `stayezy-search-flow.txt` supplied during the audit;
- `stayezy-search-gfx.txt` supplied during the audit;
- key sanitized excerpts preserved separately under `evidence/module-01-search/`.
