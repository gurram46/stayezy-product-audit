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

The API then returned `200` with:
- `status: true`
- `message: Step 1 completed`
- created property id: `1666`
- unique id: `Stay1666`
- name: `QA Test Property - Do Not Book`
- status: `Pending`
- active: `0`
- add_listing_status: `0`

This confirms the backend route accepts the string form and the normal production web request is violating the API contract.

## Additional observation
The successful response still returned `banner_image: null` / `banner_compressed: null`, so the supplied string was accepted for validation but was not persisted as the listing banner in this Step 1 response. This suggests the intended image-upload/persistence path is separate or incomplete and requires source/API inspection.

## Severity
P1 host-onboarding blocker: a new host cannot complete Step 1 through the normal web UI when selecting a main image.

## Security note
The request also includes client-supplied `user_id` and `user_firechat`. This is not evidence of an authorization flaw by itself. Backend ownership must later be verified against the authenticated principal rather than trusting these fields.

## Raw evidence handling
Screenshots/network captures exposed authentication headers. Raw credentials/tokens must not be committed to the audit repository; only sanitized findings are recorded here.
