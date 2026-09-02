# Stayezy — Dev Team Evidence Brief

Date: 2026-09-03

## Purpose

This is not a list of accusations and should not be used as one. It is a discussion brief that separates:

- **confirmed observations** from the production app/web flows;
- **working hypotheses** that require source/config access;
- **questions where the team should provide evidence** (architecture diagrams, logs, dashboards, DB schema, backup records, etc.).

Do **not** claim that the code was AI-generated or "AI slop" without commit/history evidence. What is currently supportable is narrower and stronger: production guardrails appear weak in several areas (release logging, validation, error sanitization, state isolation, API contract handling and failure recovery).

---

## Evidence-backed discussion points

### 1. Architecture walkthrough — evidence first

**Observed evidence**

The tested product visibly touches multiple systems/services:

- Flutter Android app (`com.cw.stayezy`)
- `admin.stayezy.co` APIs
- `devnode.stayezy.co` APIs on the web host flow
- Firebase Auth / FCM
- Mixpanel
- Google Maps
- Cloudflare R2-hosted property media

This is enough to justify an end-to-end architecture walkthrough, but **not** enough to infer the complete topology.

**Ask the team to show**

- request path from Flutter/web to backend(s);
- which services own auth, listings, bookings, chat, payments and search;
- which databases, queues/caches and storage systems are used;
- production/staging boundaries.

**Source:** `audit/module-01-production-baseline.md`, `audit/module-02-authentication.md`, `audit/module-01-host-listing-flow.md`.

---

### 2. Laravel vs Node ownership / migration state

**Observed evidence**

The web create-listing flow calls `devnode.stayezy.co`, while the mobile production app calls endpoints under `admin.stayezy.co`.

That confirms more than one backend/API surface is in active use, but it does **not** prove which codebase is authoritative for each domain.

**Ask the team to show**

- endpoint/domain ownership matrix;
- what is still Laravel;
- what has moved to Node;
- what exists in both systems;
- migration plan and cutover criteria.

---

### 3. Auth/session handling — confirmed sensitive release logging

**Confirmed evidence**

The production Android build logs the authenticated profile response, including:

- account identifiers;
- email/profile information;
- a live Stayezy application session/auth token.

Other captures also expose FCM tokens, precise location and detailed analytics/device identifiers.

This is a **confirmed release-logging security/privacy defect**, independent of whether the token is reusable outside the device.

**Ask the team to show**

- token lifetime;
- token rotation/revocation;
- logout behavior;
- server-side authorization middleware;
- how release builds disable/redact sensitive logs.

**Source:** `audit/module-02-authentication.md` — AUTH-001.

---

### 4. Release logging guardrails — strong evidence of missing/ineffective gating

**Confirmed evidence**

Production captures include large numbers of debug-style logs such as:

- raw API URLs and responses;
- tokens and identifiers;
- precise coordinates;
- property payloads;
- repeated implementation debug prints;
- verbose Mixpanel logs.

This supports the claim that release logging is not sufficiently gated/redacted.

**Ask the team to show**

- the logging wrapper used in Flutter;
- compile-time/environment guards (`kReleaseMode`, logger levels, interceptors, etc.);
- network-response redaction rules;
- policy for secrets/PII in logs.

**Source:** `audit/module-01-production-baseline.md` — H-007; `audit/module-02-authentication.md` — AUTH-001.

---

### 5. Chat implementation — not yet claimed, but evidence warrants inspection

**Observed evidence**

Authenticated/property flows expose `user_firechat`-style identifiers and values shaped like two-user combinations in debug output. Chat authorization has not yet been tested with controlled A/B/C accounts.

**Do not claim a chat vulnerability yet.**

**Ask the team to show**

- Firestore / Realtime DB / API / WebSocket implementation;
- conversation document/room schema;
- how host/guest membership is represented;
- where authorization is enforced.

**Source:** `audit/module-01-authenticated-favourites-and-chat-risk.md`.

---

### 6. Chat room authorization / BOLA boundary — pending controlled verification

**Current evidence level:** hypothesis / test pending.

Predictable or user-derived room IDs are not a vulnerability by themselves. The security issue exists only if an unrelated authenticated user can read/write a conversation they do not belong to.

**Ask the team to show**

- security rules/middleware for read/write;
- whether room membership is validated server-side;
- whether any client-supplied `user_id` is trusted for authorization.

**Planned safe test:** Account A (host) ↔ Account B (guest), then Account C attempts only the authorization boundary against the controlled conversation.

---

### 7. Property/listing identity model — visual duplication needs model clarification

**Confirmed evidence**

The product shows many visually near-identical listings (same/similar names and images, often differentiated only by a suffix/number). Last Minute Deals also showed the same `Aerra Farmstay` adjacent with the same location/bedrooms/guests/distance but different prices.

This is enough to ask for the domain model, but not enough to label every similar card an accidental DB duplicate.

**Ask the team to define**

- physical property vs listing vs room/unit;
- clone semantics;
- stable unique identity shown to users/admins;
- when two records with the same media/name are legitimate.

**Source:** `audit/module-01-host-listing-flow.md` — H-HOST-005; `audit/module-01-last-minute-deals-and-sticky-ui.md` — LMD-001.

---

### 8. Duplicate prevention / idempotency — evidence of duplicate-creation risk

**Observed evidence**

During QA, Step 1 could create a backend property record even though the web UI remained on Step 1 because the request was manually replayed. The same QA title was also exercised through separate web/mobile paths, producing multiple records during testing.

This does not prove normal users can always create duplicates, but it demonstrates that the create flow needs an idempotency/identity strategy.

**Ask the team to show**

- idempotency keys for multi-step listing creation;
- duplicate constraints;
- draft-resume logic;
- cleanup behavior for abandoned/incomplete property rows.

**Source:** `audit/module-01-host-listing-create-step1.md`, `audit/module-01-host-listing-step1-manual-bypass.md`, `audit/module-01-host-listing-flow.md`.

---

### 9. Multi-step property creation is not atomic — confirmed partial-state behavior

**Confirmed evidence**

The mobile host flow hit a DB error while inserting a day rate, but the property record remained and later updates/media succeeded. The listing ultimately existed in `Pending` state.

That means a failed sub-step can coexist with a partially created/updated listing.

**Risk**

- orphaned/incomplete dependent rows;
- UI state diverging from backend state;
- duplicate retries;
- hard-to-debug support cases.

**Ask the team to show**

- transaction boundaries;
- draft state machine;
- retry/idempotency behavior;
- compensating cleanup jobs.

**Source:** `audit/module-01-host-listing-flow.md` — H-HOST-003.

---

### 10. `property_day_rates.amount` validation/type handling — confirmed failure

**Confirmed evidence**

Mobile property creation returned a backend error:

`SQLSTATE[01000]: Warning: 1265 Data truncated for column 'amount' at row 1`

The SQL presented to the client attempted to insert an amount shaped like `5500,` into `property_day_rates.amount`.

This reproduced more than once.

**Ask the team to show**

- DB type for `amount`;
- request DTO/schema;
- numeric parsing/normalization in Flutter and backend;
- validation tests for commas/decimals/locale input.

**Source:** `audit/module-01-host-listing-flow.md` — H-HOST-002.

---

### 11. Raw DB/SQL exceptions reach the client — confirmed production error-sanitization problem

**Confirmed evidence**

The mobile UI displayed the SQLSTATE warning including:

- table name (`property_day_rates`);
- column name (`amount`);
- the SQL insert shape;
- concrete submitted values.

This is both poor UX and unnecessary internal implementation disclosure.

**Ask the team to show**

- global exception/error middleware;
- error mapping policy (`4xx` validation vs `5xx` internal errors);
- production stack/SQL sanitization.

**Source:** `audit/module-01-host-listing-flow.md` — H-HOST-002.

---

### 12. Web `banner_image` contract is broken — confirmed frontend/API mismatch

**Confirmed evidence**

Normal web flow:

- UI accepts and previews the selected image;
- `POST /api/properties` sends `banner_image: {}`;
- backend returns HTTP 422: `banner_image must be a string`.

Controlled replay:

- same request with `banner_image` changed to a string returned HTTP 200 and completed Step 1.

This isolates the failure to the frontend/upload/API contract rather than PNG/JPEG compatibility.

**Ask the team to show**

- intended upload sequence;
- file-to-URL/storage-key conversion;
- request schema/type definition shared between frontend/backend;
- tests for Step 1 listing creation.

**Source:** `audit/module-01-host-listing-flow.md` — H-HOST-001.

---

### 13. Map is constructed/fetched while Home is active — confirmed eager work

**Confirmed evidence**

On Home, before the user opens the Map tab:

- Google Map platform view is constructed;
- Map API is called;
- 300 properties are loaded;
- repeated BLAST buffer exhaustion appears during the same period.

A later Map visit feels faster because much of the work was already paid on Home.

**Do not claim the Map is definitively the root cause of BLAST errors yet.**

**Ask the team to show**

- bottom-nav widget tree;
- whether Map is mounted in an `IndexedStack`/kept alive;
- reason for preloading 300 properties;
- measurements that justify the preload trade-off.

**Source:** `audit/module-01-production-baseline.md` — H-003/H-005.

---

### 14. Returning Property → Home re-runs Home + Map initialization — confirmed

**Confirmed evidence**

Two isolated reproductions show:

- `Listingdetials -> NewDashboardScreen`;
- provider state logs as null;
- Map API is called again;
- `/api/home-data` is called again;
- 300 Map properties are reloaded;
- user sees Home skeleton/loading instead of immediate state restoration.

This is not just a repaint; network/state initialization actually restarts.

**Ask the team to show**

- route/navigation pattern (`push`, replacement, custom router);
- provider/controller scope;
- `_initializeData()` lifecycle trigger;
- intended state-restoration strategy.

**Source:** `audit/module-01-home-return-behavior.md` — HR-001.

---

### 15. Cache strategy needs review — evidence exists, root cause not fully assigned

**Observed evidence**

- property details can hit cache on repeat visits;
- listing images visibly arrive after metadata;
- app logs temporary/cache values that appear nearly identical and may be double-counted;
- cache grows materially during browsing.

This supports inspection of the caching strategy, not a blanket claim that caching is absent.

**Ask the team to show**

- image cache/CDN headers;
- property/API cache TTLs;
- eviction policy;
- cache-directory accounting;
- prefetch strategy.

**Source:** `audit/module-01-production-baseline.md` — H-001/H-006; property-detail audits.

---

### 16. Analytics instrumentation exists, but funnel reliability is unproven

**Observed evidence**

Mixpanel screen-view/exit events are active and verbose. Anonymous lifecycle errors were also observed (`null distinct_id`, plugin reply/lifecycle exceptions).

So the correct question is not "do we have analytics?" but "which events are trustworthy enough to make product decisions?"

**Ask the team to show**

- production Mixpanel project;
- event schema/taxonomy;
- identity merge rules anonymous → authenticated;
- funnel definitions for Search → Property → Chat → Booking → Payment;
- data-quality checks.

**Source:** `audit/module-01-production-baseline.md` — H-004; `audit/module-02-authentication.md`.

---

### 17. Crash/ANR monitoring must be demonstrated, not assumed

**Observed evidence**

The audit has already captured:

- unhandled Flutter exceptions;
- repeated frame-skip bursts;
- persistent BLAST buffer pressure;
- service-level parse exceptions;
- DB/API errors surfaced to clients.

This is enough to request actual crash/ANR telemetry.

**Ask the team to show, live if possible**

- Crashlytics/Sentry/etc. last 30 days;
- crash-free users/sessions;
- top crashes/ANRs;
- app version distribution;
- alerting/escalation path.

Do not accept "we have Crashlytics" without opening the dashboard.

**Source:** production baseline, auth and property-detail audits.

---

### 18. Known-production-issues question should be evidence-comparison, not accusation

Instead of asking "what is broken?", ask each developer:

> "What are the top three production issues you already know about, and what evidence/dashboard/log led you to each one?"

Then compare their answers against observed audit evidence:

- release token/PII logging;
- Home/Map eager work and buffer pressure;
- Home reinitialization after Back;
- web host Step 1 image contract blocker;
- mobile day-rate DB error and raw SQL disclosure;
- Last Minute duplicate/conflicting-price card;
- sticky/detail overlay defects;
- image-loading delays;
- auth/Mixpanel lifecycle errors.

This tests whether production problems are known, measured and owned.

---

### 19. Database backups + uptime/incident records — operational evidence request

**Current audit evidence:** none. This must be obtained from the team/infrastructure, so it should be framed as a request for records, not a claim.

**Ask the team to show**

#### Database resilience

- backup mechanism and provider;
- backup frequency;
- retention period;
- encryption/access controls;
- point-in-time recovery availability;
- documented RPO and RTO;
- date of the **last successful restore test** (not merely last backup);
- who receives backup-failure alerts.

#### Uptime / incidents

- uptime monitor used;
- 30/90-day API/web uptime;
- p95/p99 API latency if available;
- incident/outage history;
- alert thresholds;
- on-call/escalation owner;
- status-page or internal incident records.

A backup that has never been restored in a test is not yet demonstrated recovery capability.

---

## What the evidence supports about engineering guardrails

The current audit supports saying:

> "There are multiple signs that production engineering guardrails are inconsistent or missing: sensitive release logging, raw DB exceptions reaching clients, frontend/backend type mismatch, insufficient input normalization, partial-state multi-step writes, repeated unhandled exceptions and expensive lifecycle work. I want to understand the current development/review/testing controls before changing implementation."

The current audit does **not** support saying:

> "The entire codebase is AI-generated" or "the developers cannot code."

That requires source review, commit history, tests and PR review evidence. Keep the meeting technical and evidence-led.

---

## Suggested evidence to request during the meeting

Ask them to open, not merely describe:

1. architecture diagram or deployment topology;
2. repo list + active branches;
3. CI/CD pipeline and test gates;
4. production logging configuration;
5. auth middleware/session implementation;
6. DB schema for properties, property_day_rates, users and chat/conversations;
7. Firebase security/App Check config;
8. Crashlytics/Sentry dashboard;
9. Mixpanel production project/funnel;
10. DB backup dashboard + last restore-test evidence;
11. uptime/incident dashboard for last 30/90 days.

This turns the discussion from opinion into verifiable system state.
