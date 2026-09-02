# Stayezy — Dev Team Evidence Pack Addendum: Media Validation Guardrails

Date: 2026-09-03
Audience: Stayezy engineering team

# ===== START HERE — SHARE THIS SECTION WITH THE DEV TEAM =====

## Why did a manually supplied `banner_image` string pass Step 1?

**Question**

The normal web client selected and previewed an image but sent `banner_image: {}`, which the API rejected with HTTP 422 because it expected a string. During a controlled QA replay, only that field was changed to a manually supplied URL-shaped string that did **not** come from the normal media-upload path. The API then returned HTTP 200 / `Step 1 completed` and created the property record.

Please show:

- what `banner_image` is supposed to represent: owned storage key, approved CDN URL, upload ID, or arbitrary URL;
- where the server verifies that the media reference was produced by Stayezy's upload pipeline;
- where asset ownership is bound to the authenticated host/listing;
- allowed URL schemes/origins if URLs are accepted;
- MIME/content-signature, size and dimension validation at upload;
- whether any backend image/compression worker fetches a client-supplied URL;
- why the successful Step-1 response still returned `banner_image: null` / `banner_compressed: null`;
- what invariant prevents a listing from activating with missing/invalid required media;
- what contract/integration test prevents the web serializer and backend schema from drifting again.

**Confirmed evidence**

1. Normal production web request sent the wrong runtime type (`{}`) and was rejected.
2. Multiple JPG/PNG selections reproduced the same normal-flow failure.
3. A controlled replay changed only `banner_image` to a URL-shaped string; the endpoint accepted it and completed Step 1.
4. The created Step-1 response still reported `banner_image: null` and `banner_compressed: null`.

This is therefore more than a file-format issue. It demonstrates both a frontend/API contract failure and weakly demonstrated **semantic validation** at the Step-1 request boundary: the endpoint accepted the string without proving, at that boundary, that it represented a valid Stayezy-owned media asset.

**Proof**

- [`audit/module-01-host-listing-create-step1.md` lines 9-50](../audit/module-01-host-listing-create-step1.md#L9-L50)
- [`audit/module-01-host-listing-step1-manual-bypass.md` lines 30-95](../audit/module-01-host-listing-step1-manual-bypass.md#L30-L95)

**Consequence if the contract/guardrails remain weak**

At minimum, normal web host onboarding remains blocked and incomplete/invalid media state can be created. Depending on downstream implementation, additional risks can exist.

**Conditional security scenarios — verify in source before classifying as vulnerabilities**

- If backend media/compression code fetches arbitrary supplied URLs, lack of host/private-network restrictions can create an SSRF path.
- If arbitrary external URLs are persisted and rendered, listings can reference attacker-controlled content/tracking endpoints.
- If storage references are accepted without authenticated ownership checks, one account may be able to reference another account's uploaded media.
- If final activation does not re-check required-media invariants, partial/invalid listings can progress into operational flows.

The audit has **not** proven that the manually supplied URL is persisted, fetched or rendered. The HTTP 200 plus `banner_image: null` is exactly why the team should show the full media lifecycle instead of treating `string` type validation as sufficient.

# ===== END HERE — SHAREABLE DEV-TEAM SECTION ENDS =====
