# Module 01 — Last Minute Deals and Overlay UX

## Scope

Anonymous production-app review of Last Minute Deals and persistent floating controls on property details.

## Finding LMD-001 — Floating Chat control competes with page content and duplicate CTAs

### Observed

On property-details screens, the persistent floating `Chat` pill remains anchored near the lower-right edge while the user scrolls through long content sections.

Evidence screenshots show:

- the floating Chat pill visually sitting on top of page content rather than occupying a reserved layout region;
- a `Verified Host` card that already contains its own `Chat` CTA while the persistent floating Chat pill overlaps the same area, creating duplicate/competing actions;
- the floating control continuing to occupy the visual foreground immediately above the persistent booking footer.

### UX impact

The control is intended to help users, but the current layering can instead obscure content and create ambiguity about which Chat action is primary. It also increases visual competition with the persistent `Book Now` CTA, which is the principal conversion action on the property screen.

### Current classification

P2 UX/conversion candidate.

### Verify in code/design

- whether the floating Chat control is intentionally global on all property-detail sections;
- whether the host-card Chat CTA and floating Chat CTA invoke the same destination;
- safe-area and bottom-booking-bar offsets;
- whether the floating control should collapse, hide near equivalent CTAs, or be integrated into a structured sticky action region.

## Finding LMD-002 — Sticky section navigation can visually duplicate during scroll

### Observed

A property-details screenshot shows two visually similar section-navigation rows stacked at the top (`About Stay`, `Location`, `Food`, `Liquor`, `Amenities`, `Reviews`, etc.). This looks like the original tab row and a pinned/sticky copy being visible at the same time.

### Impact

The duplicated navigation consumes vertical space and makes the page hierarchy look broken during scroll.

### Current classification

P2 UI-state/layout candidate. Reproduce under controlled scroll and inspect the sticky-header implementation before selecting a fix.

## Finding LMD-003 — Last Minute Deals refresh state is visually misplaced

### Observed

The Last Minute Deals screen exposes a circular refresh action in the top-right header. When refresh is triggered, the existing listing cards remain visible while a large pink loading spinner appears over the property-card/image area. A platform tooltip labelled `Refresh` also appears near the header action.

The supplied screen recording shows the refresh spinner rendered directly over an existing listing image/card rather than as a cohesive list-level refresh state.

### UX impact

The user cannot immediately tell whether one card is loading, the entire list is refreshing, or the overlay is an error/temporary artifact. The spinner competes with property imagery and appears disconnected from the refresh control that initiated it.

### Current classification

P2 UX/loading-state candidate.

### Preferred direction to evaluate

Do not choose an implementation until code is inspected, but likely cleaner patterns include:

- pull-to-refresh with a standard top refresh indicator;
- a compact progress state attached to/near the header refresh action;
- preserving stale data during refresh without placing a list-level spinner over a specific property card;
- disabling/de-emphasizing the refresh action while a refresh request is in flight.

## Next runtime check

Capture an isolated Last Minute Deals refresh pass to determine:

- exact endpoint(s) called by refresh;
- duplicate/concurrent requests;
- response and processing latency;
- whether image/card data is discarded and reconstructed;
- frame jank during refresh and first scroll after refresh;
- whether unrelated Home/Map work is triggered in the background.
