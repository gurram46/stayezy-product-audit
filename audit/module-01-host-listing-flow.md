# Module 01 — Host Listing Flow

Date: 2026-09-03

## Scope
Controlled host-side listing creation using a QA property named `QA Test Property - Do Not Book`.

## Confirmed findings

### H-HOST-001 — Web Step 1 image contract failure
The web create-listing UI accepts and previews a selected image, but the request serializes `banner_image` as an object (`{}`) while the backend requires a string. The normal request returns HTTP 422 with `banner_image must be a string`.

A controlled manual resend with `banner_image` supplied as a string returned HTTP 200 and created Step 1 successfully, confirming a frontend/API contract mismatch rather than an invalid JPEG/PNG file.

Impact: P1 host-onboarding blocker on web Step 1.

### H-HOST-002 — Mobile day-rate submission causes SQL/database error
During mobile listing setup, submitting a Friday rate produced a backend 500 error:

`SQLSTATE[01000]: Warning: 1265 Data truncated for column 'amount' at row 1`

The server-side SQL shown in the response attempted to insert an amount shaped as `5500,` into `property_day_rates.amount` for property `1667`.

The same error reproduced more than once.

Questions for code review:
- What is the exact DB type for `property_day_rates.amount`?
- Why is the client/API passing a value with a trailing comma instead of a normalized numeric amount?
- Why is the raw SQL/database exception returned to the production client instead of a sanitized validation error?

### H-HOST-003 — Listing flow can proceed after day-rate failure
Despite the day-rate error, later requests updated property `1667` successfully and the property ultimately had a valid banner image URL and remained `Pending`.

This suggests the listing workflow is only partially transactional. A failed sub-step can coexist with a partially created/updated property record.

Risk: partially configured listings and state divergence between UI, property row, and dependent tables.

### H-HOST-004 — Media upload and rendering pressure
The mobile flow performs a long-running media upload with explicit progress reporting. During this and other host-listing screens, the app continued to emit repeated BLAST buffer exhaustion messages (`Already acquired max frames 8 max:6 + 2`).

This is consistent with the broader rendering/surface-pressure issue already observed elsewhere in Module 1.

### H-HOST-005 — Listing identity / apparent duplicates need product-model clarification
Search results can show many listings with nearly identical names/images, and the QA process itself created multiple property records while testing separate web and mobile flows. Before classifying all similar-looking rows as duplicates, the team needs to clarify whether the data model represents:
- one property per physical property,
- one listing per unit/room,
- cloned inventory,
- or accidental duplicate property records.

The UI should make unit identity explicit if multiple legitimate units share the same base property name and imagery.

## Security / operational note
Raw client/server errors expose internal SQL/table/column details. Production logging also remains overly verbose. Do not commit raw runtime logs containing authentication/session data.

## Status
Anonymous and authenticated guest-side baseline is substantially complete. Host listing flow has now been sampled on both web and mobile. Chat authorization testing is deferred until the QA listing is available/approved or the team provides a safe test path.
