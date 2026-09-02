# Module 01 — Host Listing Step 1 Manual Bypass

## Context
Production web host-listing flow on `https://stayezy.co/create-listing`.

## Confirmed defect
The UI accepts and previews a selected main image, but the Step 1 request serializes `banner_image` as an object (`{}`), while the API validates the field as a string.

Observed normal request:
- `POST https://devnode.stayezy.co/api/properties`
- Response: `422`
- Message: `banner_image must be a string`

Multiple JPG/PNG images reproduced the same failure.

## Controlled QA bypass
Using Firefox DevTools Edit and Resend on the same authenticated test account, only the `banner_image` field was changed from an object to a string URL. No user IDs or auth headers were changed.

The value was not produced by the application's normal media-upload path; it was a manually supplied URL-shaped string used only to test the API contract.

The API then returned `200` with:
- `status: true`
- `message: Step 1 completed`
- created property id: `1666`
- unique id: `Stay1666`
- name: `QA Test Property - Do Not Book`
- status: `Pending`
- active: `0`
- add_listing_status: `0`

This proves two separate problems:

1. **Frontend/API contract failure:** the normal production web client sends the wrong runtime type (`{}`) for a field the backend expects to be a string.
2. **Weak semantic validation at the Step-1 boundary:** changing the field to an arbitrary URL-shaped string was sufficient for the API request to pass validation and create the Step-1 property record. The request was not required, at this boundary, to prove that the value came from the application's own upload pipeline or represented a valid owned media asset.

The second point is stronger than a simple type mismatch: the observed validator appears able to reject the JSON type while not demonstrating that the string is a legitimate Stayezy media reference.

## Important limitation
The successful response returned:

- `banner_image: null`
- `banner_compressed: null`

Therefore the audit has **not** proven that the arbitrary string is persisted as the public banner, fetched by the server, or rendered to other users. The current evidence proves acceptance at the Step-1 validation/request boundary and creation of the listing record, while the actual media persistence path remains unclear.

## Required guardrails to verify in source/config
The team should show the intended invariant for `banner_image`. Depending on the design, the API should normally require one of the following rather than merely `string`:

- a server-generated storage object key tied to the authenticated user/listing;
- a URL restricted to an approved Stayezy/CDN/storage origin;
- an upload ID returned by a completed media-upload endpoint.

Also verify:

- asset ownership is checked server-side;
- MIME/content type and actual file signature are validated at upload;
- file size/dimensions are bounded;
- untrusted URL schemes/hosts are rejected if URLs are accepted;
- the backend does not blindly fetch arbitrary user-supplied URLs;
- the final listing cannot activate with a missing/invalid required banner;
- frontend/backend schemas are shared or contract-tested.

## Conditional security scenarios
These scenarios are **not yet confirmed vulnerabilities**; they become relevant depending on what source inspection shows:

- **SSRF risk:** if a backend image/compression worker later fetches arbitrary user-supplied URLs, missing host/network restrictions could allow requests toward internal/private services.
- **External-content/tracking risk:** if arbitrary external URLs are persisted and rendered, a listing could cause user devices to request attacker-controlled media endpoints, exposing request metadata/IP and creating content-integrity problems.
- **Cross-account media reference risk:** if storage object keys/URLs are accepted without verifying ownership, one account may be able to reference another account's uploaded asset.
- **Data-integrity risk:** if Step 1 can succeed with a semantically invalid banner and later steps do not enforce invariants, incomplete/invalid listings can accumulate or progress through the workflow.

Do not claim any of these as exploited until the corresponding persistence/fetch/authorization behavior is verified with source/config or controlled accounts.

## Additional observation
The successful response still returned `banner_image: null` / `banner_compressed: null`, so the supplied string was accepted for validation but was not persisted as the listing banner in this Step 1 response. This suggests the intended image-upload/persistence path is separate, incomplete, or not coupled correctly to the create-property contract and requires source/API inspection.

## Severity
- **P1 product blocker:** a new host cannot complete Step 1 through the normal web UI when selecting a main image.
- **P1/P2 engineering-guardrail finding:** the API accepted a manually supplied URL-shaped string at the validation boundary without demonstrating media provenance/ownership. Security severity depends on the downstream persistence/fetch behavior described above.

## Security note — ownership fields
The request also includes client-supplied `user_id` and `user_firechat`. This is not evidence of an authorization flaw by itself. Backend ownership must later be verified against the authenticated principal rather than trusting these fields.

## Raw evidence handling
Screenshots/network captures exposed authentication headers. Raw credentials/tokens must not be committed to the audit repository; only sanitized findings are recorded here.
