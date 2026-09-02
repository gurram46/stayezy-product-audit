# Module 01 — Last Minute Deals and Sticky UI

## Scope

Anonymous production-app review of the Last Minute Deals list and property-detail scrolling behaviour on Stayezy Android 1.1.0.

## Finding LMD-001 — Duplicate property cards appear after refresh

### Observed

After using the Last Minute Deals refresh flow, the same property appears twice in the visible list:

- Property: `Aerra Farmstay`
- Location: `Shamirpet, Hyderabad`
- Same hero image
- Same 2 Bedrooms / 12 Guests metadata
- Same distance label: `34.47 KM away`

The two cards are adjacent but show different deal-price states:

- one card shows `8500` struck through and `7500` current price;
- the next card shows `9000` struck through and `8500` current price.

### Assessment

This is a real product/data-consistency defect candidate, not only a visual issue. The client list may be appending refreshed results without replacing/deduplicating existing items, or the backend may be returning multiple records for the same property/deal context. Source inspection/API payload comparison is required before assigning root cause.

### Severity

P1/P2 candidate. Duplicate inventory with conflicting prices can damage booking confidence and can result in the same property being represented with inconsistent commercial information.

### Smallest safe correction to verify later

- identify every deal/listing by stable property/deal ID rather than display fields;
- replace refresh state atomically instead of appending if refresh semantics are intended;
- deduplicate by stable ID before rendering as a defensive measure;
- verify whether different date/deal records are intentionally represented and, if so, make that distinction explicit in the UI rather than rendering visually identical cards.

## Finding UI-DETAIL-001 — Sticky section navigation overlaps page content and controls

### Observed

While scrolling a property detail page, the `About Stay / Location / Food / Liquor / Amenities / Reviews / Home Rules` navigation row becomes sticky/floating, but multiple screenshots show it visually colliding with underlying content and top controls.

Examples include:

- body text remaining visible behind/above the pinned navigation row;
- the pinned row appearing underneath/through the Android status area;
- back/share/favourite controls visually intersecting the sticky region during some scroll positions;
- in one earlier reproduction, both the original and sticky section rows were visible at once;
- section content can begin directly behind the sticky header, reducing readability and making the page feel unstable while scrolling.

### Assessment

Confirmed layout/scroll-state defect. The sticky header offset/stacking implementation is not consistently accounting for the status area, hero controls and transition point between the normal and pinned header.

### Severity

P2 UX defect. It does not block booking but is highly visible on a core property-detail flow and creates ambiguity about which content/control layer is active.

### Correction direction to verify in code

Use a single coordinated sliver/sticky-header implementation with explicit safe-area/top inset handling, deterministic z-order, and one source of truth for whether the section header is normal or pinned. Avoid rendering both normal and sticky copies simultaneously.

## Finding UI-DETAIL-002 — Floating Chat control competes with page content and other Chat actions

### Observed

A floating `Chat` pill remains anchored near the lower-right while the property page scrolls. It overlays body copy, the embedded location map and amenity/review content. The Verified Host section also contains its own Chat action, producing duplicate Chat CTAs in the same viewport in some states.

### Assessment

The floating control is meant to help, but its persistent overlay currently creates visual noise and can cover content. It also competes with the primary persistent booking CTA at the bottom.

### Severity

P2 UX/conversion-quality issue.

### Product/design direction

- keep one clear Chat affordance per viewport;
- reserve the persistent bottom area primarily for booking conversion;
- if Chat must remain floating, define collision-safe spacing against sticky headers, maps, cards and bottom booking controls;
- consider contextual Chat placement in the Verified Host/help section instead of a global overlay.

## Finding LMD-002 — Refresh feedback is visually detached from list state

### Observed

The Last Minute Deals screen uses a top-right refresh icon. When invoked, a loading spinner appears over the listing-card/image region while stale cards remain visible. The progress indicator therefore looks associated with a specific card rather than the whole list refresh.

### Assessment

Confirmed UI feedback issue. Whether refresh also causes unnecessary network/list reconstruction still requires a valid isolated log capture.

### Severity

P3/P2 UX issue, potentially higher if logs confirm duplicated/refetched state causing LMD-001.

## Recovered-capture note

A later attempt to recover the overwritten Last Minute Deals log from Android's ring buffer did not preserve the actual refresh/API interaction. The recovered file contains only a short lifecycle segment around `19:32:20–19:32:24`, so it cannot distinguish client-side append behaviour from a duplicate backend payload.

The corresponding `gfxinfo` snapshot still shows measurable session jank:

- 1,111 frames rendered
- 113 janky frames (**10.17%**)
- 101 high-input-latency events
- 60 slow UI-thread events
- 65 slow draw-command events
- 113 missed frame deadlines
- 95th percentile frame time: 18 ms
- 99th percentile frame time: 46 ms

These figures cover the broader active session and must not be attributed exclusively to the refresh action. They are consistent with the wider rendering/jank findings already observed across the app.

## Evidence note

The supplied screenshots visibly capture the duplicate `Aerra Farmstay` cards, conflicting prices, sticky-header/content collisions, floating Chat overlap and Last Minute Deals refresh presentation.

The first Last Minute Deals log capture was accidentally overwritten by starting the same PowerShell redirection command a second time. The later ring-buffer recovery did not retain the original refresh records, so source/API inspection will be required to assign root cause for LMD-001 unless the flow is reproduced again in isolation.