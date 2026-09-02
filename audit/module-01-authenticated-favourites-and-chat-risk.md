# Module 01 — Authenticated Favourites and Chat-Risk Follow-up

## Scope

Authenticated production-app review after creation of a test user account. This note records confirmed findings from the favourites/property-detail interaction and frames the next chat-security check without assigning an unverified root cause.

## Finding AUTH-LOG-002 — Authenticated session/profile data continues to be printed in release logs

### Confirmed

During the authenticated favourites/property-detail flow, release logcat repeatedly prints:

- the current user ID;
- the user's email/profile fields;
- the Stayezy auth/session token;
- the FCM push token;
- precise location coordinates;
- profile/API responses.

This confirms the earlier auth-log finding is persistent and not limited to initial signup.

### Severity

P1 security/privacy hardening issue. Severity may escalate if the printed Stayezy token is a long-lived bearer credential that can be replayed from another client. That replay property has not yet been tested and must not be assumed.

### Required remediation direction

- remove sensitive production logging entirely;
- never print auth tokens, FCM tokens, PII, precise coordinates, or full authenticated API bodies;
- route release logs through structured redaction;
- verify token expiry, revocation, rotation and device/session binding once backend access is available.

## Finding FAV-001 — Property-details async callback calls setState after state disposal

### Confirmed

After quickly opening and leaving a property-details screen during the favourites interaction, Flutter logs an unhandled exception from `Listingdetials.calculateDistance1`:

`Null check operator used on a null value`

The stack shows `State.setState` from an asynchronous continuation after navigation away from the screen.

### Assessment

This is consistent with an async lifecycle defect: a delayed distance/location callback appears to update widget state after the relevant state is no longer valid. Source inspection should verify `mounted` checks/cancellation and any forced null assertions.

### Severity

P2 reliability issue. It is a real unhandled exception even though the app remained usable in this reproduction.

## Finding PERF-FAV-001 — Authenticated favourites/property-details session remains janky

The isolated graphics capture recorded:

- 214 rendered frames;
- 33 janky frames (15.42%);
- 26 missed vsyncs;
- 27 slow UI-thread events;
- 33 frame-deadline misses;
- 95th percentile frame time 44 ms;
- 99th percentile frame time 150 ms.

This is session-level evidence and should not be attributed to one exact favourites operation without narrower instrumentation.

## Chat-security hypothesis to test next

Property-detail logs repeatedly print a value shaped like `413_3663` while the authenticated user ID is `3663` and the listing/deal data references another user/owner ID `413`.

This strongly suggests the app may derive some conversation/peer identifier from the two participant IDs, but the meaning is **not yet confirmed**. Predictable chat/conversation IDs are not themselves a vulnerability if server/Firebase authorization rules enforce membership on every read/write.

The next chat audit should therefore verify, using only owned test accounts and passive production observation:

1. what backend/service stores chat data;
2. what conversation/document identifier is used;
3. whether message payloads, peer IDs, tokens or PII are logged;
4. whether the client receives more conversations/messages than the current user should see;
5. whether authorization is enforced server-side/through Firebase rules rather than only by client-side filtering;
6. whether unread counters, attachments and message history obey the same authorization boundary.

Do not enumerate or attempt access to unrelated users' conversations. A two-owned-account test or source/rules inspection is the correct way to test horizontal authorization/BOLA/IDOR safely.

## Raw evidence handling

Do not commit the raw authenticated log files because they contain a live test-user token, email address, FCM token and precise location. Only sanitized findings belong in this repository.
