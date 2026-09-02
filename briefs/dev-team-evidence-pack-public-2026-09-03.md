# Stayezy — Dev Team Evidence Pack (Shareable)

Date: 2026-09-03  
Audience: Stayezy engineering team  
Purpose: establish the current production state using evidence, then agree on what must be verified in source/configuration.

> This document separates **confirmed production observations** from **conditional security scenarios**. A conditional scenario is not being presented as an already-proven exploit; it explains the consequence if the missing server-side/configuration control is also absent.

---

# ===== START HERE — SHARE THIS SECTION WITH THE DEV TEAM =====

## 1. Can we first map ownership and the architecture end to end?

**Question**  
Can each engineer state what they currently own, then walk through Flutter/web -> backend services -> database/cache/storage -> Firebase/chat/notifications -> payments, including production vs staging?

**Why this is being asked**  
The production flows visibly use more than one API/service surface. The web host flow calls `devnode.stayezy.co/api/properties`; the complete authority boundary is not yet documented in this audit.

**Proof**  
- [`audit/module-01-host-listing-create-step1.md` lines 9-15](../audit/module-01-host-listing-create-step1.md#L9-L15)

**If this remains unclear**  
Incidents can bounce between teams, migrations can leave two sources of truth, and security fixes can be applied to one API surface while another remains exposed.

---

## 2. What is the Laravel -> Node migration state, and which system is authoritative for each domain?

**Question**  
Please show an endpoint/domain ownership matrix for auth, users, listings, search, bookings, payments, favourites, chat and admin functions. Identify anything currently implemented in both systems and the cutover plan.

**Evidence level**  
The audit confirms multiple active API surfaces, but does **not** independently prove which framework owns each one. This question requires repository/deployment evidence from the team.

**Consequence if ownership is ambiguous**  
Dual writes, inconsistent validation/auth rules, stale endpoints and data divergence become likely during migration.

---

## 3. What is the canonical property identity model, and how do we prevent accidental duplication?

**Question**  
What exactly is unique: physical property, listing, unit/room, clone, or deal? Which stable IDs and DB constraints prevent accidental duplicates? Please show the relevant schema/constraints.

**Confirmed evidence**  
Search/UI contains many near-identical listing names/images, which needs domain-model clarification. More importantly, Last Minute Deals visibly showed the same `Aerra Farmstay` details twice with conflicting price states.

**Proof**  
- [`audit/module-01-host-listing-flow.md` lines 43-50](../audit/module-01-host-listing-flow.md#L43-L50)
- [`audit/module-01-last-minute-deals-and-sticky-ui.md` lines 7-36](../audit/module-01-last-minute-deals-and-sticky-ui.md#L7-L36)

**If not fixed/defined**  
Possible outcomes include duplicate inventory, conflicting prices, double operational work, ambiguous booking ownership and customer disputes. If two rows represent legitimate units, the UI still needs to make that identity explicit.

---

## 4. What release-mode logging guardrails exist in Flutter and the backend?

**Question**  
Please show the logging wrapper/configuration used in release builds: `kReleaseMode`/logger levels, network interceptor redaction, PII/secret redaction and CI/static checks that prevent raw `print`/debug output from shipping.

**Confirmed evidence**  
The Play-installed production build has logged live application session/auth data, profile fields, FCM token, precise location, device/analytics identifiers, full API responses and user search keystrokes.

**Proof**  
- [`audit/module-02-authentication.md` lines 24-44](../audit/module-02-authentication.md#L24-L44)
- [`audit/module-01-authenticated-favourites-and-chat-risk.md` lines 7-31](../audit/module-01-authenticated-favourites-and-chat-risk.md#L7-L31)
- [`audit/module-01-production-baseline.md` lines 173-179](../audit/module-01-production-baseline.md#L173-L179)
- [`audit/module-01-search.md` lines 77-93](../audit/module-01-search.md#L77-L93)

**Security consequence / concrete scenario**  
If the printed Stayezy token is a reusable bearer credential, a person/process that obtains a diagnostic log, rooted-device log, ADB capture, support dump or other privileged log export may obtain the user's session credential. If that token is long-lived and not device-bound/revoked, the impact can become session impersonation/account takeover until expiry or revocation. Even where the token is not replayable, email/profile/location/search-intent leakage remains a privacy incident.

---

## 5. How exactly are Stayezy sessions authenticated, expired and revoked?

**Question**  
Please show token issuance, lifetime, refresh/rotation, logout revocation, server-side authorization middleware and whether application sessions are device-bound.

**Confirmed evidence**  
A live Stayezy application session/auth token is printed in the authenticated production log path.

**Proof**  
- [`audit/module-02-authentication.md` lines 24-44](../audit/module-02-authentication.md#L24-L44)

**Security consequence**  
The logging defect becomes materially worse if the credential is long-lived, bearer-only, accepted from another client or remains valid after logout.

---

## 6. Is Firebase App Check intentionally disabled, and where is it enforced?

**Question**  
Please show Firebase App Check configuration/enforcement for the production Android app and every Firebase product used by auth/chat/storage/functions.

**Confirmed evidence**  
Google auth repeatedly reported `No AppCheckProvider installed` and used a placeholder token.

**Proof**  
- [`audit/module-02-authentication.md` lines 46-69](../audit/module-02-authentication.md#L46-L69)

**Conditional security scenario**  
Missing App Check does not itself prove auth bypass. However, if Firebase resources rely on App Check as part of abuse protection and enforcement is absent/weak, scripted non-genuine clients can interact with those Firebase surfaces more easily, increasing scraping, automated abuse and quota/cost risk.

---

## 7. Why does one Google signup appear to traverse multiple Firebase/Stayezy identities?

**Question**  
Please trace `GoogleSignIn -> Firebase credential -> Stayezy login/register -> any custom Firebase token -> persisted local session`, including account-linking and duplicate-account prevention.

**Confirmed evidence**  
One signup sequence reported two different Firebase Auth identifiers about 1.4 seconds apart before the Stayezy backend profile was loaded.

**Proof**  
- [`audit/module-02-authentication.md` lines 71-97](../audit/module-02-authentication.md#L71-L97)

**If mapping is not deterministic**  
Potential consequences include duplicate accounts, wrong ownership mapping, broken logout/re-login identity and authorization decisions being made against the wrong principal.

---

## 8. How is Chat implemented and how is conversation membership authorized?

**Question**  
Please show whether Chat uses Firestore, Realtime Database, backend API or WebSocket; the room/document schema; membership model; and read/write authorization rules.

**Observed evidence**  
Property-detail logs repeatedly emitted a value shaped like `413_3663`, where `3663` was the authenticated user ID and `413` matched another listing/owner-side ID. This is a hypothesis about a participant/room identifier, not yet proof of a vulnerability.

**Proof**  
- [`audit/module-01-authenticated-favourites-and-chat-risk.md` lines 65-82](../audit/module-01-authenticated-favourites-and-chat-risk.md#L65-L82)

**Conditional security scenario**  
Predictable room IDs are harmless **if** every read/write validates membership. If authorization is only client-side or Firebase/server rules fail to verify membership, an unrelated authenticated account could potentially read another user's private messages, attachments or write into the conversation. This is the BOLA/IDOR boundary we intend to test only with controlled accounts.

---

## 9. Does the backend derive ownership from the authenticated principal or trust client-supplied IDs?

**Question**  
Please show the ownership check for create/update listing and chat-related operations. Specifically, are `user_id` / `user_firechat` ignored or validated against the authenticated principal?

**Confirmed evidence**  
The web create-property JSON contains client-supplied `user_id` and `user_firechat` fields.

**Proof**  
- [`audit/module-01-host-listing-create-step1.md` lines 9-36](../audit/module-01-host-listing-create-step1.md#L9-L36)
- [`audit/module-01-host-listing-create-step1.md` lines 65-66](../audit/module-01-host-listing-create-step1.md#L65-L66)

**Conditional security scenario**  
Their presence is not itself a vulnerability. If the backend trusts a mutable client ID instead of binding ownership to the authenticated principal, this can become a horizontal/vertical authorization flaw where a user creates or modifies resources under another account.

---

## 10. What idempotency and draft-resume controls protect the four-step listing flow?

**Question**  
Please show idempotency keys, uniqueness constraints, draft state machine, retry semantics and cleanup for abandoned/incomplete listings.

**Confirmed evidence**  
A Step-1 backend property could be created while the web UI remained on Step 1, and later host-flow failures did not necessarily roll back the listing.

**Proof**  
- [`audit/module-01-host-listing-step1-manual-bypass.md` lines 16-33](../audit/module-01-host-listing-step1-manual-bypass.md#L16-L33)
- [`audit/module-01-host-listing-flow.md` lines 31-36](../audit/module-01-host-listing-flow.md#L31-L36)

**If not controlled**  
Retries/timeouts/double taps can leave duplicate or partial properties, dependent-table drift, inconsistent approval state and support cleanup work.

---

## 11. Why can property creation continue after a dependent DB write fails?

**Question**  
Please show transaction boundaries around property creation, day rates, media, amenities and final activation. Is partial state intentional as a draft state machine, or accidental?

**Confirmed evidence**  
The mobile flow hit a day-rate DB error, but later requests successfully updated the same property and it remained as a `Pending` listing.

**Proof**  
- [`audit/module-01-host-listing-flow.md` lines 31-36](../audit/module-01-host-listing-flow.md#L31-L36)

**If unintentional**  
A user may see a listing that exists but lacks required dependent data; retries can duplicate rows; approvals/bookings can operate against inconsistent configuration.

---

## 12. What validates and normalizes money/day-rate input before SQL?

**Question**  
Please show the Flutter parsing, API DTO/schema and DB type for `property_day_rates.amount`, including tests for comma/decimal/locale input.

**Confirmed evidence**  
Mobile host creation returned `SQLSTATE[01000] ... Data truncated for column 'amount'` and the disclosed SQL attempted to insert an amount shaped like `5500,`. The failure reproduced.

**Proof**  
- [`audit/module-01-host-listing-flow.md` lines 17-29](../audit/module-01-host-listing-flow.md#L17-L29)

**If not fixed**  
Hosts can be blocked from listing creation, malformed price values can propagate, and pricing data may become inconsistent depending on DB coercion/strict-mode behavior.

---

## 13. Why are raw database/SQL details returned to the production client?

**Question**  
Please show global exception middleware and the policy mapping validation errors vs internal errors. Production clients should receive a stable error code/message, while SQL/stack details remain server-side.

**Confirmed evidence**  
The mobile UI displayed SQLSTATE, table name, column name, SQL insert structure and submitted values.

**Proof**  
- [`audit/module-01-host-listing-flow.md` lines 17-29](../audit/module-01-host-listing-flow.md#L17-L29)
- [`audit/module-01-host-listing-flow.md` lines 52-53](../audit/module-01-host-listing-flow.md#L52-L53)

**Security consequence**  
This does not directly grant database access, but it gives an attacker reconnaissance information about schema/table/column names, query structure and validation behavior, reducing the uncertainty required to probe other API weaknesses. It also exposes internal values to ordinary users.

---

## 14. Why is web host Step 1 sending the wrong `banner_image` type?

**Question**  
Please show the intended upload sequence and shared request schema. Is the image supposed to be uploaded first and converted to a URL/storage key before `/api/properties`?

**Confirmed evidence**  
Normal flow: image previews, request sends `banner_image: {}`, backend returns 422 `banner_image must be a string`. Multiple JPG/PNG files reproduced. A controlled resend changing only that field to a string returned HTTP 200 and completed Step 1.

**Proof**  
- [`audit/module-01-host-listing-create-step1.md` lines 9-50](../audit/module-01-host-listing-create-step1.md#L9-L50)
- [`audit/module-01-host-listing-step1-manual-bypass.md` lines 6-33](../audit/module-01-host-listing-step1-manual-bypass.md#L6-L33)

**If not fixed**  
Normal web host onboarding is blocked at the first step, directly affecting supply acquisition and forcing manual/support workarounds.

---

## 15. Why is Google Map constructed and ~300 properties fetched while Home is active?

**Question**  
Please show the bottom-navigation/widget lifecycle and the measurement that justified eager Map construction. What is the smallest A/B isolation test we can run with Map lazy-mounted only when selected?

**Confirmed evidence**  
With Home visible and even idle, the Google Map platform view is constructed, Map API loads ~300 properties and repeated BLAST buffer exhaustion continues. A later Map visit benefits from work already paid on Home. Correlation is confirmed; Map being the direct BLAST root cause is **not** yet proven.

**Proof**  
- [`audit/module-01-production-baseline.md` lines 83-112](../audit/module-01-production-baseline.md#L83-L112)
- [`audit/module-01-production-baseline.md` lines 114-132](../audit/module-01-production-baseline.md#L114-L132)

**If not addressed/measured**  
Home startup can pay unnecessary native-view/network/GPU/memory cost, increasing jank, battery/data use and risk of ANR/crash under lower-end-device pressure.

---

## 16. Why does Property -> Back reconstruct/refetch Home and Map data?

**Question**  
Please show navigation mode, provider/controller scope and what invokes `_initializeData()` when returning from details.

**Confirmed evidence**  
Two isolated reproductions show Property -> `NewDashboardScreen`, provider state logged null, Map GET reissued, Home API reissued and ~300 Map properties reloaded while the user sees skeleton/loading state.

**Proof**  
- [`audit/module-01-home-return-behavior.md` lines 12-37](../audit/module-01-home-return-behavior.md#L12-L37)
- [`audit/module-01-home-return-behavior.md` lines 39-49](../audit/module-01-home-return-behavior.md#L39-L49)

**If not fixed**  
A common browse/back flow repeatedly spends network/render work, loses state/scroll continuity and compounds image/map pressure, making the app feel unstable even when APIs are healthy.

---

## 17. What is the actual cache/CDN strategy, and is cache accounting correct?

**Question**  
Please show image/CDN cache headers, Flutter image cache strategy, API TTLs, eviction rules and why temporary/cache accounting appears duplicated.

**Observed evidence**  
Images can render roughly 2–2.5 seconds after card metadata in the captured baseline, while temporary/cache values appear nearly identical and may be double-counted. Root cause is not yet assigned.

**Proof**  
- [`audit/module-01-production-baseline.md` lines 39-56](../audit/module-01-production-baseline.md#L39-L56)
- [`audit/module-01-production-baseline.md` lines 163-171](../audit/module-01-production-baseline.md#L163-L171)

**If not understood**  
Users repeatedly pay image/network/decode cost; incorrect cache accounting can trigger premature cleanup or mask real growth. This is primarily performance/reliability, not a security claim.

---

## 18. Can you show the actual production quality/observability gates rather than describing them?

**Question**  
Please open CI/CD, Flutter/web/backend test suites, lint/static-analysis gates, Crashlytics/Sentry, Mixpanel and alerting. Which checks block a release?

**Evidence supporting the question**  
Production testing has captured startup frame skips/BLAST pressure, analytics lifecycle exceptions, an unhandled favourites-response parse error, an async widget-state exception, raw release debug output and DB/API errors reaching clients.

**Proof**  
- [`audit/module-01-production-baseline.md` lines 70-101](../audit/module-01-production-baseline.md#L70-L101)
- [`audit/module-01-production-baseline.md` lines 134-146](../audit/module-01-production-baseline.md#L134-L146)
- [`audit/module-02-authentication.md` lines 116-133](../audit/module-02-authentication.md#L116-L133)
- [`audit/module-01-authenticated-favourites-and-chat-risk.md` lines 33-63](../audit/module-01-authenticated-favourites-and-chat-risk.md#L33-L63)

**If gates/monitoring are weak**  
The organization discovers preventable regressions from customers instead of pre-merge/pre-release checks. Security-sensitive logging and contract/type errors can repeatedly re-enter production even after one-off fixes.

---

## 19. Can you show database backup/recovery evidence and 30/90-day uptime/incident records?

**Question**  
Please show, live if possible:

- DB provider and backup mechanism;
- backup frequency and retention;
- encryption/access controls;
- point-in-time recovery;
- defined RPO/RTO;
- **date and result of the last successful restore test**;
- backup-failure alert owner;
- uptime monitor and 30/90-day API/web uptime;
- p95/p99 latency if available;
- incident/outage history and escalation/on-call owner.

**Audit evidence level**  
No backup or uptime records are available in the client-side audit. This is intentionally an **evidence request**, not a claim that backups are missing.

**Consequence if the controls do not exist**  
A database backup that has never been restored is not demonstrated recovery. In a destructive migration, operator mistake, compromised credential, database corruption or provider incident, the business could lose bookings/users/listings beyond the acceptable RPO or remain down beyond the acceptable RTO. Without uptime/incident records, recurring outages and latency regressions cannot be measured or owned.

---

## Final request to the team

For any point above, a repository/config/dashboard demonstration is more useful than a verbal answer. Where our current conclusion is only a hypothesis, source/config evidence should either confirm it or close it.

# ===== END HERE — SHAREABLE DEV-TEAM SECTION ENDS =====
